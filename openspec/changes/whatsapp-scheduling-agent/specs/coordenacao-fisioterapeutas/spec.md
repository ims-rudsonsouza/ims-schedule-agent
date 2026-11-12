# Spec Delta

## Purpose

Coordenar com as fisioterapeutas, pelo mesmo número de WhatsApp usado com as famílias, a oferta, o aceite e as mudanças de atendimentos.

## ADDED Requirements

### Requirement: Oferta de atendimento para a fisioterapeuta
O sistema SHALL enviar à fisioterapeuta uma oferta com data, horário, nome e idade da criança, endereço completo e distância/tempo estimado de deslocamento, com as opções "Aceitar" e "Recusar" e a possibilidade de responder por texto. O telefone do responsável SHALL ser enviado somente após a confirmação.

#### Scenario: Oferta enviada
- **WHEN** a alocação escolhe uma fisio para um pedido
- **THEN** ela recebe a oferta com nome da criança, endereço completo e botões de resposta rápida

#### Scenario: Aceite
- **WHEN** a fisio toca em "Aceitar" dentro do prazo e o horário ainda está livre
- **THEN** o agendamento é confirmado, o evento é criado na agenda dela e ela recebe o telefone do responsável

### Requirement: Prazo de resposta
Cada oferta individual SHALL ter prazo de resposta configurável (padrão 20 minutos). Para atendimentos que começam em poucas horas, o prazo SHALL ser reduzido para que, somado ao deslocamento, a resposta chegue antes da hora de sair (padrão 8 minutos no mesmo dia e 5 minutos para início em menos de 2 horas). O estouro do prazo SHALL ser tratado como recusa pelo botão.

#### Scenario: Oferta para daqui a 1h30
- **WHEN** às 10h uma oferta é enviada para um atendimento às 11h30
- **THEN** a fisio tem 5 minutos para responder

#### Scenario: Prazo esgotado
- **WHEN** o prazo termina sem resposta da fisio
- **THEN** a oferta expira, a fisio é avisada de que expirou e a família é consultada como em uma recusa

### Requirement: Botão Recusar volta à família
Quando a fisio tocar em "Recusar" na oferta de um horário escolhido pela família, o sistema MUST NOT ofertar o pedido a outra fisio automaticamente, seja a fisio pedida pela família ou não. O agente SHALL informar à família que a fisio não tem disponibilidade e perguntar se pode verificar com outra fisio; só com o "sim" da família o sistema SHALL buscar outra.

#### Scenario: Fisio toca em Recusar
- **WHEN** a família escolheu "terça 14h com a Bia" e a Bia toca em "Recusar"
- **THEN** a família recebe "A Bia não tem disponibilidade nesse horário. Posso verificar com outra fisioterapeuta?" e nenhuma outra fisio é acionada antes da resposta

#### Scenario: Família autoriza outra fisio
- **WHEN** a família responde que pode ser outra fisio
- **THEN** o sistema roda a alocação (raio, deslocamento, carga) sem a fisio que recusou e oferta o horário à próxima candidata

#### Scenario: Família prefere a mesma fisio
- **WHEN** a família responde que quer continuar com a Bia
- **THEN** o agente oferece os próximos horários livres da Bia

#### Scenario: Fisio responde por texto em vez do botão
- **WHEN** a fisio não toca em nenhum botão e escreve "esse dia não, mas quinta de manhã eu consigo"
- **THEN** o agente trata a mensagem como proposta de outro dia/horário e inicia uma negociação, sem consultar a família sobre outra fisio

### Requirement: Oferta aberta com primeira resposta vencedora
Na oferta aberta (sem fisio a até 10 km), o sistema SHALL enviar a oferta simultaneamente a todas as fisios elegíveis, e SHALL atribuir o atendimento à primeira que aceitar enquanto nenhuma rodada de propostas estiver com a família. As demais SHALL ser avisadas de que o horário foi preenchido. A família SHALL ser consultada somente quando todas recusarem ou o prazo acabar sem aceite.

#### Scenario: Duas fisios aceitam quase ao mesmo tempo
- **WHEN** duas fisios aceitam a mesma oferta aberta com segundos de diferença
- **THEN** apenas o primeiro aceite registrado é confirmado e a segunda recebe "esse horário já foi preenchido, obrigado!"

### Requirement: Fila de respostas na oferta aberta
Na oferta aberta, quando uma fisio propuser um ou mais horários, o sistema SHALL enviar essas opções à família como uma rodada e suspender as demais ofertas. Respostas que chegarem durante a suspensão SHALL entrar numa fila, sem serem descartadas. Se a família recusar a rodada, o sistema SHALL apresentar a próxima resposta da fila. Se a família recusar e sugerir um horário, uma resposta da fila compatível com essa sugestão SHALL ter prioridade sobre a ordem da fila.

#### Scenario: Fisio propõe dois horários
- **WHEN** numa oferta aberta para terça 14h a Carla responde "posso terça 16h ou quarta 9h"
- **THEN** a família recebe as duas opções da Carla e as ofertas às outras fisios ficam suspensas

#### Scenario: Família recusa a rodada
- **WHEN** a família recusa as opções da Carla e na fila há uma proposta da Dani para quarta 15h
- **THEN** a família recebe a opção da Dani e a Carla é avisada de que a família não aceitou

#### Scenario: Família sugere horário que está na fila
- **WHEN** a família recusa as opções da Carla e diz "quinta de manhã seria melhor", e na fila há a Dani com quarta 15h e depois a Edu com quinta 9h
- **THEN** a opção da Edu (quinta 9h) é apresentada primeiro, por ser compatível com o pedido da família

#### Scenario: Aceite do horário original durante a suspensão
- **WHEN** enquanto a família avalia as opções da Carla a Dani toca em "Aceitar" o horário original
- **THEN** o aceite da Dani entra no topo da fila e é apresentado à família logo depois da rodada atual, se a família não aceitar as opções da Carla

#### Scenario: Fila vazia
- **WHEN** a família recusa a rodada e não há mais respostas na fila
- **THEN** as ofertas suspensas voltam a valer até o prazo, e a família é consultada sobre outros horários se ninguém mais responder

### Requirement: Contraproposta da fisioterapeuta
Além dos botões, a fisioterapeuta SHALL poder responder à oferta por texto livre. Quando o texto indicar a intenção de outro dia ou horário, o sistema SHALL tratá-lo como proposta (e não como recusa), interpretar a sugestão, verificar se cabe na agenda dela (com deslocamento) e, se couber, abrir uma negociação com a família; se não couber, SHALL explicar o conflito e pedir outra sugestão.

#### Scenario: Fisio sugere outro horário
- **WHEN** a fisio responde à oferta de terça 14h com "terça não dá, consigo quarta às 10h"
- **THEN** o sistema confirma com ela "quarta 10h, certo? Vou consultar a família", verifica a agenda e abre a negociação

#### Scenario: Sugestão conflita com a agenda dela
- **WHEN** a fisio sugere quarta 10h mas já tem atendimento às 9h30 do outro lado da cidade
- **THEN** o agente explica que o horário não cabe com o deslocamento e pergunta se ela tem outra opção

#### Scenario: Pergunta antes de decidir
- **WHEN** a fisio pergunta "a criança mora em casa ou apartamento?" antes de responder
- **THEN** o agente responde com a informação do cadastro, ou diz que não tem e oferece perguntar à família, sem encerrar a oferta

### Requirement: Negociação exclusiva de dia e horário
Uma contraproposta SHALL abrir uma negociação entre a família e aquela fisio, com o agente intermediando as mensagens, seja a fisio pedida pela família ou não. Durante a negociação, o pedido MUST NOT ser ofertado a nenhuma outra fisio. Um horário só é confirmado quando família e fisio concordam com ele e a agenda é reverificada.

#### Scenario: Negociação até um acordo
- **WHEN** a Ana sugere quarta 10h, a família responde "quarta só depois das 15h" e a Ana aceita quarta 16h
- **THEN** o agente confere a agenda dela, confirma quarta 16h e avisa as duas partes

#### Scenario: Nenhuma oferta paralela durante a negociação
- **WHEN** a Bia, que não foi pedida pela família, está negociando o horário
- **THEN** nenhuma outra fisio recebe oferta daquele pedido enquanto a negociação estiver aberta

#### Scenario: Negociação durante oferta aberta
- **WHEN** numa oferta aberta a Carla sugere outro horário antes de qualquer aceite
- **THEN** as ofertas abertas às demais fisios são suspensas e as opções da Carla seguem para a família, conforme o requisito de fila da oferta aberta

### Requirement: Fim da negociação sem acordo
A negociação SHALL terminar sem acordo quando a família ou a fisio desistir, ou quando passar o prazo máximo (padrão 2 horas, ou 3 rodadas sem acordo). Ao terminar sem acordo, o sistema SHALL avisar a fisio e perguntar à família se pode verificar com outra fisio, inclusive quando a família tiver pedido aquela fisio; só com o "sim" da família o sistema SHALL buscar outra.

#### Scenario: Negociação falha com a fisio pedida
- **WHEN** a família pediu a Ana e não chegaram a um horário comum
- **THEN** o agente diz à família que não foi possível com a Ana e pergunta se ela aceita ser atendida por outra fisio

#### Scenario: Família aceita outra fisio após a negociação
- **WHEN** a família responde que pode ser outra fisio
- **THEN** o sistema roda a alocação sem a fisio da negociação, começando pelo horário escolhido pela família, se ainda estiver livre

#### Scenario: Prazo da negociação esgotado
- **WHEN** a negociação fica 2 horas sem acordo
- **THEN** ela é encerrada sem acordo, as duas partes são avisadas e a família é consultada sobre outra fisio

### Requirement: Aceite obrigatório da fisioterapeuta
Todo atendimento oferecido pelo sistema MUST ser aceito pela fisioterapeuta antes da confirmação à família, seja pelo botão "Aceitar" ou concordando com um horário na negociação. O sistema MUST NOT confirmar um atendimento só porque o horário está livre na agenda. Sessões que a própria fisio registra pela conversa contam como aceitas por ela.

#### Scenario: Horário livre sem resposta da fisio
- **WHEN** a fisio tem o horário livre na agenda mas ainda não respondeu à oferta
- **THEN** a família continua com o status "confirmando com a fisio" e nada é gravado como confirmado

#### Scenario: Sessão registrada pela própria fisio
- **WHEN** a fisio escreve "marquei a próxima da Maria quinta 10h"
- **THEN** o atendimento é confirmado sem nova oferta a ela

### Requirement: Notificações de mudanças
O sistema SHALL notificar a fisioterapeuta por WhatsApp sobre cancelamentos e remarcações dos seus atendimentos, e SHALL enviar um resumo da agenda do dia seguinte em horário configurável.

#### Scenario: Família cancela
- **WHEN** um responsável cancela um atendimento
- **THEN** a fisio recebe aviso com data e horário cancelados

### Requirement: Comandos da fisioterapeuta
A fisioterapeuta SHALL poder, pela conversa, consultar sua agenda e avisar indisponibilidade pontual.

#### Scenario: Consulta da agenda
- **WHEN** a fisio escreve "o que eu tenho amanhã?"
- **THEN** o agente lista os atendimentos do dia seguinte com horário, nome da criança e bairro

### Requirement: Sessões seguintes combinadas pessoalmente
As sessões seguintes de um tratamento são combinadas pessoalmente entre a família e a fisio. O sistema SHALL permitir que a fisio as registre pela conversa ("marquei a próxima da Maria quinta 10h") e SHALL tratar como ocupado qualquer evento que ela crie direto na agenda dedicada. Sessões registradas pela conversa SHALL receber evento na agenda e lembrete de véspera para a família.

#### Scenario: Fisio registra a próxima sessão pelo WhatsApp
- **WHEN** a fisio escreve "marquei com a Maria quinta às 10h"
- **THEN** o agente confirma a criança e o horário, verifica a agenda, cria o atendimento e a família recebe a confirmação e, depois, o lembrete de véspera

#### Scenario: Fisio cria o evento direto na agenda
- **WHEN** a fisio cria na agenda dedicada um evento que o sistema não criou
- **THEN** o horário passa a contar como ocupado e entra no cálculo de ocupação dela, sem enviar mensagens à família

### Requirement: Repasse entre fisioterapeutas
A fisioterapeuta SHALL poder repassar um atendimento já marcado: para uma colega que ela indicar, ou pedindo ao sistema que procure outra. O atendimento SHALL só sair da agenda da fisio original quando outra aceitar, e a família SHALL ser avisada de quem vai atender.

#### Scenario: Repasse para colega indicada
- **WHEN** a fisio escreve "a Bia vai atender a Maria na quinta 10h no meu lugar"
- **THEN** a Bia recebe a oferta e, se aceitar, o evento passa para a agenda dela e a família é avisada da troca

#### Scenario: Repasse sem colega indicada
- **WHEN** a fisio escreve "não vou conseguir ir na de quinta 14h, pode ver outra?"
- **THEN** o sistema confirma qual atendimento, roda a alocação sem ela e avisa a equipe humana se ninguém aceitar

#### Scenario: Colega indicada recusa
- **WHEN** a Bia recusa o repasse
- **THEN** a fisio original é avisada e o agente pergunta se ela quer indicar outra colega ou deixar o sistema procurar
