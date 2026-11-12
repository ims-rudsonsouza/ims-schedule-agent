# Design: agente de agendamento via WhatsApp

## Context

Motivação e escopo estão em `proposal.md`; o comportamento esperado está nas specs desta mudança. Este documento descreve **como** construir.

Ponto de partida: o design anterior (`knowledge/whatsapp_scheduling_agent_design`, jul/2026) propôs Python/FastAPI, Claude com tool-use, Postgres/PostGIS, Redis, Twilio, Google Maps e Google Calendar na AWS sa-east-1. Esta proposta **mantém** a base e **muda** três pontos a pedido do Rudson:

1. Agenda em **Outlook / Microsoft 365** (Microsoft Graph) em vez de Google Calendar.
2. O **mesmo número** de WhatsApp conversa com famílias e com fisioterapeutas.
3. Regras explícitas de alocação: preferência, raio de 10 km, menor deslocamento, equilíbrio de carga e oferta aberta para todas quando ninguém está no raio.

Restrições: um desenvolvedor, ~2 meses até o piloto, 10 fisios hoje, dados de saúde de crianças (LGPD arts. 11 e 14), fuso America/Sao_Paulo.

## Goals / Non-Goals

**Goals:**
- O motor de alocação é **determinístico e testável**; o LLM conversa e chama ferramentas, mas não escolhe fisio nem inventa horário.
- Separação rígida entre os dois papéis no mesmo número (família x fisio): prompts, ferramentas e dados diferentes.
- Outlook continua sendo a agenda que as fisios usam no dia a dia; o sistema se adapta a edições manuais.

**Non-Goals (MVP):**
- Prontuário, evolução clínica, faturamento, convênios e pagamentos.
- Otimização de rotas completa (VRP/OR-Tools); usamos heurística gulosa por inserção.
- Painel administrativo completo (apenas telas mínimas de cadastro e auditoria, ou planilha/Retool).
- Pacotes ou séries de sessões reservados pelo agente. Pelo WhatsApp chega o primeiro contato, uma sessão por vez. As seguintes são combinadas pessoalmente entre família e fisio, e a fisio as registra pela conversa ou direto na agenda (D3).

## Arquitetura

```
 Famílias ─┐                                         ┌─> Microsoft Graph (Outlook)
           ├─> WhatsApp (1 número) ─> Webhook API ───┤
 Fisios  ──┘    via BSP/Meta          (FastAPI)      ├─> Google Maps (geocode + rotas)
                                          │          └─> Claude API (conversa)
                                          v
                         Roteador de papel (telefone -> família | fisio | equipe)
                          │                  │
              Orquestrador família   Orquestrador fisio      Fila/agendador de tarefas
              (LLM + ferramentas)    (botões + LLM leve)     (prazos de oferta, lembretes,
                          │                  │                renovação de webhooks)
                          └──────┬───────────┘
                                 v
                    Motor de agendamento (determinístico)
                    - elegibilidade, ranking, ofertas, máquina de estados
                                 v
                 Postgres + PostGIS (fonte de estado)  ·  Redis (sessões, cache, locks)
```

| Componente | Responsabilidade |
| --- | --- |
| Webhook API | Valida assinatura do provedor, deduplica por `message_id`, grava a mensagem e responde 200 rápido; o processamento segue em fila. |
| Roteador de papel | Procura o telefone no cadastro de fisios e da equipe; caso contrário, trata como família. |
| Orquestrador família | Claude com tool-use e prompt de "secretária"; ferramentas: `consultar_cadastro`, `registrar_consentimento`, `geocodificar_endereco`, `buscar_opcoes`, `solicitar_agendamento`, `cancelar`, `remarcar`, `transferir_humano`. |
| Orquestrador fisio | Primeiro trata botões (Aceitar/Recusar) de forma determinística; texto livre vai a um LLM com ferramentas restritas (`detalhes_da_oferta`, `propor_alternativa`, `minha_agenda`, `registrar_sessao`, `informar_indisponibilidade`, `repassar`). |
| Motor de agendamento | Calcula janelas livres, deslocamento, ranking, cria e encerra ofertas, confirma agendamentos. Sem LLM. |
| Integração Outlook | Leitura (`calendarView`), escrita de eventos, assinaturas de notificação e `delta` como rede de segurança. |
| Agendador de tarefas | Expiração de ofertas, próxima candidata, lembretes de véspera, resumo diário para fisios, renovação das assinaturas do Graph. |

## Decisions

### D1. Fluxo de conversa com a família

```
1. Saudação + apresentação como assistente virtual
2. Consentimento LGPD (botões "Aceito" / "Ver aviso" / "Falar com a equipe")
3. Dados: responsável -> criança (nome, idade) -> endereço (texto ou localização do WhatsApp)
   -> geocodificação e confirmação do endereço ("Rua X, 123 - Bairro Y, certo?")
4. Necessidade: quando? (hoje / amanhã / dias e turnos) e preferência de fisio
   - checagem determinística de sinais de gravidade em toda mensagem -> orientação de emergência + humano
5. buscar_opcoes -> o agente apresenta 2-3 opções reais ("1) Ter 14h com Ana  2) Ter 16h com Bia ...")
6. Família escolhe -> solicitar_agendamento -> "Vou confirmar com a Ana e já te aviso" -> oferta à fisio (D3)
   (sempre com aceite da fisio; não há confirmação automática)
7. Confirmado -> resumo + .ics + lembrete de véspera agendado
   Sem aceite -> desculpas + novas opções ou humano
```

Quando a família não tem preferência, as opções podem vir de fisios diferentes; a opção 1 é sempre a mais bem ranqueada. Se a família escolher uma opção, a oferta vai à fisio daquela opção. Se ela tocar em **Recusar** (ou deixar o prazo expirar), o agente volta à família, informa que a fisio não tem disponibilidade e pergunta se pode verificar com outra fisio; só com o "sim" o motor procura outra fisio no mesmo horário. Se ela responder por texto com outro dia/horário, abre-se uma negociação (D3).

Alternativa considerada: perguntar à fisio antes de mostrar opções à família. Descartada porque deixa a família esperando sem saber o que escolher; mostrar opções reais e confirmar depois é o que uma secretária faz.

### D2. Algoritmo de alocação

**Entradas:** endereço geocodificado, janela desejada, preferência (opcional), histórico da criança.

**Passo 1 — Janelas candidatas por fisio.** Para cada fisio, ler disponibilidade e eventos (D4) e gerar inícios possíveis a cada 30 min dentro da janela. Para cada início, encontrar o compromisso anterior (P) e o seguinte (S) no dia; se não houver, usar a residência da fisio. O bloco precisa caber:

```
fim(P) + desloc(P -> novo) <= início
início + duração_sessão + margem_relatório + desloc(novo -> S) <= início(S)
```

Padrões: sessão 60 min, margem 10 min (configuráveis por tipo de serviço).

**Antecedência e dias:** todos os dias da semana, inclusive fins de semana e feriados, valem desde que haja bloco `Disponível`; não há calendário de feriados no motor. Para cada candidato, o início precisa respeitar:

```
início >= agora + 1 h                                         (antecedência mínima, configurável)
início >= agora + prazo_da_oferta + desloc(origem_atual -> novo) (tempo para responder e chegar)
```

A `origem_atual` é o atendimento em andamento ou anterior no dia, ou a residência. Pedidos com início em menos de 3 h são marcados `urgente`. Para eles, se não houver ninguém no raio, a oferta aberta sai na hora, e a família recebe o aviso "estou confirmando, te respondo em até X minutos". Na negociação de um pedido urgente, o prazo total é limitado pela mesma conta.

**Passo 2 — Preferência.** Se há fisio preferida (escolhida pela família ou "fisio de continuidade" das últimas sessões), retornar só as janelas dela. Sem janela na faixa pedida: retornar as próximas janelas dela + até 2 alternativas de outras fisios, marcadas como tal.

**Passo 3 — Raio de 10 km.** `dist_origem` = distância de rota entre a origem efetiva (P, ou residência se não houver P) e o endereço. Fisios com `dist_origem <= 10 km` formam o grupo **no raio**.

**Passo 4 — Pontuação (menor é melhor) para o grupo no raio:**

```
custo_desloc = desloc(P->novo) + desloc(novo->S) - desloc(P->S)     (minutos de deslocamento adicionados)
ocupacao     = horas_ocupadas_dentro_dos_blocos_Disponível / horas_disponíveis   (período: semana atual + próxima)
               (inclui sessões combinadas pessoalmente e eventos que a fisio criou direto na agenda)
score        = w_d * (custo_desloc / 60) + w_c * ocupacao
padrões: w_d = 1,0 ; w_c = 1,0  ->  10 pontos percentuais de ocupação "valem" ~6 min de deslocamento
```

- Usar **inserção** (custo adicional) e não apenas a distância da residência premia encaixar visitas vizinhas no mesmo roteiro.
- Ocupação relativa à disponibilidade atende "parecido conforme a disponibilidade": uma fisio com 10 h livres e 5 atendimentos está tão ocupada quanto uma com 4 h e 2.
- Trava de justiça opcional: se a ocupação de uma fisio ultrapassar a média do grupo em mais de 25 pp, ela só é ofertada se não houver outra no raio.

**Passo 5 — Fora do raio (oferta aberta).** Se o grupo no raio estiver vazio para o horário escolhido, criar uma **oferta aberta** para todas as fisios com o horário livre (Passo 1), qualquer distância. Primeira a aceitar fica com o atendimento.

**Passo 6 — Saída.** Para a família: até 3 opções (horário + fisio), deduplicadas por horário. Para o motor: a lista ordenada de candidatas por horário, usada nas ofertas sequenciais.

**Cálculo de deslocamento:** pré-filtro por distância em linha reta (PostGIS) para descartar quem está a mais de ~25 km quando há gente no raio; depois Google Routes (matriz de rotas) com horário de saída para tempo e distância reais. Cache por par (geohash de ~150 m, faixa de horário) por 7 dias para conter custo.

Alternativa considerada: OR-Tools/VRP. Excessivo para 10 fisios e volume inicial; reavaliar com dados do piloto (o log de decisões alimenta essa análise).

### D3. Ofertas às fisios e concorrência

Máquinas de estado:

```
Pedido:  COLETANDO -> OPCOES_ENVIADAS -> AGUARDANDO_FISIO -> CONFIRMADO
                                              |-> NEGOCIANDO (com 1 fisio) -> CONFIRMADO | SEM_ACORDO
                                              |-> AGUARDANDO_FAMILIA (fisio recusou / expirou) -> nova alocação | horários da mesma fisio
         SEM_ACORDO -> AGUARDANDO_FAMILIA
                                              |-> SEM_ACEITE -> (novas opções | humano)
         CONFIRMADO -> CANCELADO | REMARCADO | A_VERIFICAR (edição externa no Outlook)
Oferta:  ENVIADA -> ACEITA | RECUSADA | CONTRAPROPOSTA | EXPIRADA | PREENCHIDA (outra aceitou antes)
Negociação: ABERTA -> ACORDO | SEM_ACORDO (desistência, 3 rodadas ou 2 h)
Oferta aberta em suspensão: ENVIADA -> SUSPENSA (outra fisio negociando) -> ENVIADA | PREENCHIDA
```

- **Sequencial** (grupo no raio): uma oferta por vez, prazo padrão 20 min (8 min para pedidos do mesmo dia, 5 min para início em menos de 2 h). Recusa pelo botão ou expiração **não** dispara a próxima automaticamente: o pedido vai para `AGUARDANDO_FAMILIA` e a próxima candidata só recebe a oferta depois do "sim" da família.
- **Aberta** (fora do raio): todas ao mesmo tempo, prazo padrão 30 min. Recusas individuais não voltam à família; ela só é consultada se todas recusarem ou o prazo acabar sem aceite.
- **Concorrência:** o aceite executa uma transação que (1) trava a linha do pedido (`SELECT ... FOR UPDATE`), (2) verifica que o pedido ainda está `AGUARDANDO_FISIO`, (3) reconsulta o Outlook daquele intervalo, (4) cria o evento, (5) marca `CONFIRMADO` e as demais ofertas `PREENCHIDA`. Restrição de exclusão no Postgres (`EXCLUDE USING gist` em fisio + intervalo) impede dois agendamentos sobrepostos para a mesma fisio mesmo com bug de aplicação.
- Enquanto aguarda aceite, o horário fica **pré-reservado** no banco (não no Outlook), para não oferecer o mesmo horário a outra família.
- Conteúdo da oferta: data, horário, nome e idade da criança, endereço completo, distância/tempo e, se houver, observação logística ("prédio sem elevador"). O telefone do responsável só vai à fisio confirmada.
- **Negociação (conversa com a fisio):** a fisio pode responder por texto em vez de tocar nos botões. O LLM do papel fisio usa `propor_alternativa(oferta_id, inicio)`; o motor valida o horário na agenda dela com as mesmas regras de D2 (deslocamento, margem, conflitos) e, se não couber, o agente explica e pede outra opção. Uma proposta válida abre uma **negociação exclusiva**: o pedido fica travado para aquela fisio (nenhuma oferta sequencial nova; numa oferta aberta, as ofertas das demais ficam `SUSPENSA`) e o agente leva e traz as propostas entre família e fisio. Cada proposta validada é pré-reservada no banco enquanto a outra parte responde. Acordo = as duas partes disseram sim ao mesmo horário → reverificação da agenda e confirmação. Sem acordo (desistência, 3 rodadas ou 2 h, configuráveis): o pedido vai para `AGUARDANDO_FAMILIA` e o agente pergunta se pode verificar com outra fisio, tenha a família pedido aquela fisio ou não; com o "sim", a alocação é retomada sem ela (numa oferta aberta, as ofertas suspensas voltam a valer, se o horário ainda estiver livre).
- **Botão Recusar x resposta por texto:** o botão é a recusa explícita; o orquestrador da fisio trata o toque de forma determinística e o pedido vai para `AGUARDANDO_FAMILIA`, com a mensagem "A Bia não tem disponibilidade nesse horário. Posso verificar com outra fisioterapeuta, ou prefere outro horário com a Bia?". Isso vale para qualquer fisio, pedida pela família ou escolhida pelo ranking. Texto livre nunca é tratado como recusa: se indicar outro dia/horário, abre negociação; se for só uma negativa ("não posso"), o agente pergunta se ela quer recusar ou sugerir outro horário. O estouro do prazo é tratado como recusa.
- **Sessões seguintes e repasses (como funciona hoje):** depois da primeira visita, família e fisio combinam as próximas pessoalmente, e havendo divergência a própria fisio passa o atendimento a uma colega. O sistema não tenta substituir isso. A fisio pode (a) dizer ao agente "marquei a próxima da Maria quinta 10h", e então o sistema valida a agenda, cria o atendimento vinculado à criança e manda a confirmação e o lembrete à família; ou (b) só criar o evento na agenda dedicada, e então ele conta como ocupado e entra na ocupação, sem mensagens à família. Repasse: `repassar(agendamento_id, colega?)`. Com colega indicada, a oferta vai só a ela; sem colega, roda a alocação sem a fisio original. O evento só muda de agenda quando a nova fisio aceita, e a família é avisada da troca.
- Perguntas da fisio sobre a oferta ("tem elevador?", "a criança tem quantos meses?") são respondidas com `detalhes_da_oferta`; o que não estiver no cadastro pode ser perguntado à família pelo agente, sem encerrar a oferta. O prazo da oferta é estendido em 10 min enquanto houver conversa ativa.

### D4. Integração com as agendas das fisios (Outlook e Google Agenda)

**Modelo (definido pelo Rudson):** cada fisio tem **sua própria conta, com e-mail pessoal**, que pode ser Outlook.com/Hotmail (conta Microsoft) **ou Gmail** (Google Agenda). O sistema acessa a agenda de cada uma em nome dela, por um adaptador com duas implementações. As regras abaixo valem para as duas; os detalhes de cada provedor vêm no fim desta seção.

**Outlook (Microsoft Graph):**

- **Autenticação:** app registrado no Entra ID aceitando **contas Microsoft pessoais** (audiência `consumers` ou `common`), com **permissões delegadas** `Calendars.ReadWrite` e `offline_access`. Permissão de aplicativo (app-only) não funciona para contas pessoais. Cada fisio conecta a conta uma vez por um link enviado pelo WhatsApp (fluxo OAuth com código de autorização + PKCE) e consente. O refresh token fica criptografado (KMS) e é renovado a cada uso.
- **Desconexão:** se o token for revogado, a senha mudar ou a conta ficar muito tempo sem uso, a renovação falha. A fisio deixa de receber ofertas, e ela e a equipe recebem um aviso com novo link de conexão.
- **Agenda dedicada:** na conexão, o sistema cria na conta da fisio uma agenda chamada "Atendimentos IMS". Os blocos `Disponível` e os atendimentos ficam nela, e a fisio vê tudo no Outlook do celular junto com os compromissos pessoais.
- **Disponibilidade:** blocos com a categoria `Disponível` (podem ser recorrentes) na agenda dedicada. Ocupado = qualquer evento na agenda dedicada ou na agenda principal da fisio que não esteja marcado como livre. Leitura via `GET /me/calendars/{id}/calendarView?startDateTime=…&endDateTime=…&$select=start,end,showAs,categories,isCancelled` (expande recorrências). Dos eventos pessoais o sistema lê **só início, fim e status**, nunca título, local ou participantes.
- **Escrita:** `POST /me/calendars/{dedicada}/events` com `transactionId` (idempotência) e uma *single-value extended property* com o id do agendamento. O sistema só escreve na agenda dedicada.
- **Sincronização:** assinatura de notificações de alteração por fisio (agenda dedicada e principal), renovada antes de expirar (≈3 dias), e `delta` de `calendarView` a cada 5 min como rede de segurança. Edição/exclusão manual de um evento do sistema → pedido `A_VERIFICAR` + pergunta à fisio pelo WhatsApp.
- **Adaptador de agenda:** a integração fica atrás de uma interface (`conectar`, `ler_ocupacao`, `criar_evento`, `atualizar_evento`, `cancelar_evento`, `assinar_mudancas`, `sincronizar`). O motor de agendamento só conhece a interface, e o provedor de cada fisio fica no cadastro (`provedor_agenda`).

**Google Agenda (Google Calendar API):**
- **Autenticação:** projeto no Google Cloud com tela de consentimento OAuth externa e permissões delegadas com escopos mínimos: `calendar.app.created` (criar e gerenciar só as agendas criadas pelo app) e `calendar.freebusy` (ler apenas os horários ocupados das outras agendas). Mesmo fluxo de conexão por link no WhatsApp (OAuth + PKCE, `access_type=offline`), com refresh token criptografado.
- **Verificação do app:** com a tela de consentimento em modo "Teste", o Google expira os refresh tokens em 7 dias, o que derrubaria a conexão das fisios toda semana. Antes do piloto, o app precisa ser publicado e passar pela verificação do Google para escopos sensíveis (política de privacidade publicada, domínio verificado e vídeo do fluxo). Contar algumas semanas para isso no cronograma.
- **Agenda dedicada:** `calendars.insert` cria "Atendimentos IMS". Blocos `Disponível` e atendimentos ficam nela, e a fisio os vê no app Google Agenda.
- **Disponibilidade:** eventos da agenda dedicada via `events.list` (`singleEvents=true` para expandir recorrências). A ocupação das agendas pessoais vem de `freeBusy.query`, que já devolve **só intervalos ocupados**, sem título nem local, o que atende o requisito de privacidade por construção.
- **Escrita:** `events.insert` na agenda dedicada com `extendedProperties.private` contendo o id do agendamento, e um id de evento gerado pelo sistema para garantir idempotência.
- **Sincronização:** `events.watch` (notificações push por webhook) na agenda dedicada, com o canal renovado antes de expirar, mais `syncToken` incremental a cada 5 min como rede de segurança. As agendas pessoais não emitem notificações com esses escopos, então a ocupação é consultada por `freeBusy` na hora de montar opções e de novo antes de confirmar.

**Diferenças que o motor precisa saber:** no Outlook, uma mudança na agenda principal chega por notificação. No Google, o motor só descobre na próxima consulta de `freeBusy`. A reverificação antes de confirmar (spec `agenda-fisioterapeutas`) cobre os dois casos.
- **Cache:** janela de agenda em Redis por fisio (TTL curto, invalidado pelas notificações); sempre reconsultar antes de confirmar.

### D5. WhatsApp e número único

- **Provedor:** manter um adaptador de canal com duas implementações possíveis — Twilio (já decidido no design anterior; sandbox imediato) ou Meta Cloud API direta (mais barata por mensagem). Recomendação: começar no Twilio e migrar se o custo pesar.
- **Roteamento:** papel decidido pelo telefone (cadastro de fisios e equipe). Fisio que fala como mãe de paciente escolhe o papel explicitamente; a sessão guarda o papel ativo por 2 h.
- **Janela de 24 h:** ofertas, lembretes e avisos iniciados pelo sistema usam **templates de utilidade** aprovados com botões de resposta rápida: `oferta_atendimento`, `oferta_preenchida`, `oferta_expirada`, `confirmacao_agendamento`, `lembrete_vespera`, `aviso_cancelamento`, `resumo_agenda_dia`.
- **Política da Meta para IA:** a Meta restringiu chatbots de IA de propósito geral na API do WhatsApp Business; um assistente restrito ao agendamento da clínica é uso de negócio permitido, mas o prompt deve recusar conversas fora do escopo. Confirmar a política vigente ao registrar o número.
- **Humanização:** respostas curtas, indicador de "digitando", leve atraso proporcional ao tamanho do texto, emojis com moderação, nome da clínica, nunca fingir ser humano.

### D6. LLM e salvaguardas

- Claude com tool-use; histórico curto em Redis (últimas ~20 mensagens + resumo) e fatos estruturados no Postgres.
- Toda informação factual (horários, nomes, endereços) vem de ferramentas; o prompt proíbe inventar horários, e o servidor valida que qualquer opção enviada à família existe no resultado de `buscar_opcoes`.
- Detecção de gravidade é **determinística** (lista de termos + classificador simples) e roda antes do LLM.
- Toolsets separados por papel; o orquestrador da fisio não tem ferramentas que leiam dados de outras famílias.
- Avaliação: conjunto de ~50 conversas roteirizadas (feliz, ambígua, cancelamento, gravidade, fora de escopo) rodando em CI contra o prompt.

### D7. Dados (resumo)

| Entidade | Campos principais |
| --- | --- |
| Responsavel | id, telefone, nome, consentimento_em, versao_aviso |
| Crianca | id, responsavel_id, nome, data_nascimento, fisio_continuidade_id |
| Endereco | id, responsavel_id, texto, complemento, geom (PostGIS) |
| Fisioterapeuta | id, nome, telefone, residencia_geom, provedor_agenda, agenda_dedicada_id, token_ref (segredo criptografado), conexao_status, ativa |
| Pedido | id, crianca_id, endereco_id, janela, preferencia_fisio_id, estado, urgente |
| Oferta | id, pedido_id, fisio_id, inicio, fim, modo (sequencial/aberta), estado, expira_em |
| Agendamento | id, pedido_id, fisio_id, intervalo (tstzrange), outlook_event_id, estado |
| DecisaoAlocacao | pedido_id, candidatas (json com scores), escolhida, motivo |
| Mensagem | id_provedor (único), telefone, direcao, papel, texto, criada_em |

### D8. LGPD e dados de crianças

- **Base legal:** dados de saúde (art. 11) e de crianças (art. 14) → consentimento específico e destacado de pai/mãe ou responsável legal, além da tutela da saúde quando aplicável; registrar prova do consentimento.
- **Minimização:** sem diagnóstico no WhatsApp nem no Outlook; evento com primeiro nome/iniciais, endereço, telefone do responsável e código. Ofertas levam nome, idade e endereço da criança (necessários para a fisio decidir), mas não telefone do responsável nem dados clínicos. Como a oferta aberta envia esses dados a várias fisios, o aviso de privacidade deve dizer isso e as fisios devem assinar termo de confidencialidade; mensagens já entregues no celular delas não podem ser apagadas pelo sistema.
- **Operadores e transferência internacional (art. 33):** Twilio/Meta, Microsoft, Google e Anthropic processam dados fora do Brasil em parte; listar todos no aviso de privacidade e firmar DPAs. Banco e backups em sa-east-1.
- **Retenção:** histórico de conversa 90 dias; agendamentos pelo prazo que a clínica definir (ver perguntas); exclusão sob pedido.
- **Segurança:** TLS, criptografia em repouso (RDS/KMS), segredos no Secrets Manager, log de acesso humano, princípio do menor privilégio no Graph (D4): o sistema lê só horários das agendas pessoais e escreve só na agenda dedicada, no Outlook e no Google; tokens das fisios criptografados e revogáveis. O Google passa a ser operador (Google Calendar API) e deve constar no aviso de privacidade.
- Revisão por profissional de LGPD antes do piloto com dados reais.

### D9. Infraestrutura

Mantida a do design anterior: AWS sa-east-1, FastAPI em App Runner/Fargate, RDS Postgres + PostGIS, ElastiCache Redis, SQS (ou fila em Postgres) para tarefas com atraso, CloudWatch para métricas e alertas (ofertas expiradas, assinaturas do Graph prestes a vencer, erros do provedor de WhatsApp).

## Risks / Trade-offs

- [Fisios não respondem rápido às ofertas] → prazos curtos, lembrete de oferta pendente na metade do prazo, ondas paralelas configuráveis e alerta para a equipe humana (não há aceite automático, por decisão do Rudson).
- [Verificação do app no Google atrasa o piloto ou tokens expiram em 7 dias no modo Teste] → iniciar a verificação na fase de fundação; no piloto interno, usar contas Outlook ou aceitar a reconexão semanal.
- [Fisio revoga o acesso ou o token expira] → status de conexão monitorado, fisio sai das ofertas até reconectar, aviso com link para ela e para a equipe.
- [Agenda no Outlook fica desatualizada ou é usada de forma inconsistente] → convenção simples (categoria `Disponível`), resumo diário pelo WhatsApp para a fisio conferir, sincronização por notificação + delta.
- [Família espera demais pela confirmação] → mensagem honesta de "estou confirmando", prazo máximo por pedido (ex.: 60 min) antes de oferecer alternativas ou humano.
- [Dados da criança em celulares de fisios que não ficaram com o atendimento] → termo de confidencialidade, oferta sem telefone e sem dados clínicos, aviso de privacidade explícito.
- [Negociação exclusiva com uma fisio atrasa o agendamento] → prazo máximo de negociação (2 h / 3 rodadas), avisos às duas partes antes de expirar e consulta à família sobre outra fisio ao final.
- [Família consultada a cada recusa demora mais para ter um agendamento] → botões "Pode ser outra" / "Prefiro esperar" na pergunta, para resposta em um toque.
- [Suspender a oferta aberta pode irritar fisios que queriam o horário] → mensagem clara de "em negociação" e reenvio da oferta se a negociação falhar.
- [Mistura de papéis no mesmo número] → roteamento por telefone, toolsets separados, testes específicos de vazamento de dados entre papéis.
- [LLM inventa horário ou promete o que não pode] → validação servidor-side das opções, temperatura baixa, testes de regressão de conversa.
- [Custo de APIs de rota] → pré-filtro geográfico e cache por geohash.
- [Mudança de política da Meta sobre bots de IA] → escopo restrito ao agendamento e adaptador de canal que permite trocar de provedor.
- [Viés do equilíbrio de carga contra quem tem pouca disponibilidade] → ocupação relativa à disponibilidade e pesos ajustáveis com base no log de decisões.

## Migration Plan

Projeto novo, sem migração de dados. Implantação em etapas: (1) sandbox do WhatsApp com a equipe interna, (2) piloto com 2–3 fisios e famílias convidadas, (3) todas as 10 fisios. Rollback: desligar o webhook e voltar ao atendimento humano no mesmo número (o provedor permite redirecionar para a caixa de atendimento).

## Open Questions

As respostas abaixo ajustam parâmetros e detalhes; onde não houver resposta, vale a suposição indicada.

1. ~~**Agenda:** uma conta da clínica ou uma por fisio?~~ **Respondido:** cada fisio tem conta própria, com e-mail pessoal. Também pode haver fisio com Gmail, então o sistema suporta Outlook e Google Agenda (D4).
2. ~~**Como as fisios informam disponibilidade?**~~ **Respondido:** a fisio cria blocos `Disponível` na agenda dedicada (Outlook ou Google), como descrito em D4.
3. ~~**Pacotes e recorrência**~~ **Respondido:** pelo WhatsApp chega o primeiro contato, uma sessão por vez. As seguintes são combinadas pessoalmente entre família e fisio e registradas por ela (D3); em caso de divergência, a própria fisio repassa o atendimento a uma colega.
4. ~~**Urgência**~~ **Respondido:** há atendimento no mesmo dia, em fins de semana e feriados; antecedência mínima de 1 hora (D2).
5. ~~**Aceite da fisio**~~ **Respondido:** a fisio sempre precisa aceitar; não há aceite automático.
6. **Número de WhatsApp:** número novo ou o atual da clínica? A empresa já tem Meta Business verificada?
7. **Raio de 10 km:** contar a partir da residência e também do atendimento anterior do dia (suposição) ou só da residência? Distância de rota (suposição) ou em linha reta?
8. **Equilíbrio:** medir por horas agendadas sobre disponíveis em 2 semanas (suposição) ou por número de pacientes ativos?
9. **Oferta aberta e negociação:** suposição de que, se uma fisio propõe outro horário numa oferta aberta, as ofertas às demais ficam suspensas até o fim da negociação. Confirmar.
10. **Prazo da negociação:** 2 horas ou 3 rodadas sem acordo é razoável?
11. **Prazo de resposta das fisios:** 20 min é aceitável? E de madrugada, ofertas ficam para a manhã?
12. **Duração:** a sessão é sempre de 60 min + 10 min de relatório?
13. **Nome da fisio nas opções:** mostrar o nome da fisio nas opções à família (suposição) ou só após o aceite?
14. **Equipe humana:** quem recebe as transferências e em que horário? Existe um número atual da secretária que será migrado?
15. **Área atendida:** quais cidades/bairros? Fora dela o agente recusa ou passa para humano?
16. **Valores, convênio e pagamento** ficam fora do agente no MVP (suposição)?
17. **Retenção de agendamentos:** por quanto tempo a clínica precisa guardar o histórico de atendimentos?
