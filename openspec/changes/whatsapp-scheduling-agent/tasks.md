# Tasks

> Antes de iniciar o grupo 4, revisar as respostas do Rudson às perguntas em aberto de `design.md` e ajustar parâmetros.

## 1. Fundação

- [ ] 1.1 Criar projeto Python/FastAPI com estrutura `app/` (canal, orquestradores, motor, integrações) e verificar que `pytest` roda e `GET /health` retorna 200
- [ ] 1.2 Provisionar Postgres + PostGIS e Redis locais (docker compose) e verificar conexão por teste de integração
- [ ] 1.3 Criar migrações das entidades de D7, incluindo a restrição de exclusão por fisio + intervalo, e verificar por teste que dois agendamentos sobrepostos da mesma fisio são rejeitados
- [ ] 1.4 Configurar CI (lint, tipos, testes) e verificar pipeline verde no PR

## 2. Canal WhatsApp

- [ ] 2.1 Implementar adaptador de canal (interface + implementação Twilio) com validação de assinatura e verificar por teste que webhook com assinatura inválida é rejeitado
- [ ] 2.2 Implementar deduplicação por id da mensagem e fila de processamento; verificar por teste que webhook duplicado gera uma única resposta
- [ ] 2.3 Implementar roteador de papel por telefone (família, fisio, equipe, papel ambíguo) e verificar pelos cenários da spec `canal-whatsapp`
- [ ] 2.4 Redigir e submeter os templates de utilidade de D5; verificar aprovação no painel do provedor
- [ ] 2.5 Documentar em `docs/whatsapp.md` como configurar número, sandbox e templates

## 3. Integração Outlook

- [ ] 3.1 Registrar app no Entra ID com `Calendars.ReadWrite` restrito à caixa da agenda e verificar que o app não lê outra caixa do tenant
- [ ] 3.2 Implementar leitura de disponibilidade (blocos `Disponível` menos eventos ocupados) e verificar com testes usando respostas gravadas do Graph
- [ ] 3.3 Implementar criação/atualização/cancelamento de eventos com `transactionId` e propriedade estendida; verificar idempotência por teste
- [ ] 3.4 Implementar assinaturas de notificação, renovação automática e delta de segurança; verificar que exclusão manual marca o agendamento como `A_VERIFICAR`
- [ ] 3.5 Documentar em `docs/outlook.md` a convenção de disponibilidade para as fisios

## 4. Motor de alocação

- [ ] 4.1 Implementar geocodificação e matriz de rotas com cache por geohash; verificar por teste que pares repetidos não chamam a API
- [ ] 4.2 Implementar janelas candidatas com deslocamento e margem (D2 passo 1); verificar o cenário "deslocamento não cabe"
- [ ] 4.3 Implementar preferência e continuidade (passo 2); verificar os cenários da spec `alocacao-fisioterapeuta`
- [ ] 4.4 Implementar raio de 10 km com origem efetiva, pontuação por inserção + ocupação e trava de justiça (passos 3–4); verificar os cenários de empate e diferença de distância
- [ ] 4.5 Implementar decisão de oferta aberta (passo 5) e o registro `DecisaoAlocacao`; verificar por teste
- [ ] 4.6 Criar simulador com as 10 fisios e 200 pedidos sintéticos que reporte distribuição de ocupação e km por fisio; verificar que nenhuma fisio fica >25 pp acima da média com os pesos padrão

## 5. Coordenação com fisioterapeutas

- [ ] 5.1 Implementar máquinas de estado de Pedido e Oferta e verificar transições inválidas rejeitadas por teste
- [ ] 5.2 Implementar ofertas sequenciais com prazo, expiração e próxima candidata; verificar com relógio simulado
- [ ] 5.3 Implementar oferta aberta com primeiro aceite vencedor e aviso de "preenchido"; verificar com teste de aceites concorrentes
- [ ] 5.4 Implementar aceite automático, notificações de mudança e resumo diário; verificar pelos cenários da spec `coordenacao-fisioterapeutas`
- [ ] 5.5 Implementar negociação exclusiva (texto livre → `propor_alternativa`, validação na agenda, intermediação família↔fisio, trava do pedido, suspensão de ofertas abertas, prazo e fallback) e perguntas sobre a oferta; verificar pelos cenários de negociação da spec `coordenacao-fisioterapeutas`
- [ ] 5.6 Implementar a regra da fisio pedida (recusa/expiração volta à família, alocação sem ela só com o "sim" da família) e a diferença para a fisio de continuidade; verificar pelos cenários correspondentes das specs `coordenacao-fisioterapeutas` e `alocacao-fisioterapeuta`
- [ ] 5.7 Implementar comandos da fisio (agenda, indisponibilidade, repasse) com toolset restrito; verificar por teste que o toolset não acessa dados de outras famílias

## 6. Conversa com responsáveis

- [ ] 6.1 Escrever prompt de sistema da "secretária" e ferramentas da família (D6); verificar que o servidor bloqueia opções não retornadas por `buscar_opcoes`
- [ ] 6.2 Implementar detecção determinística de gravidade antes do LLM; verificar com lista de frases de teste
- [ ] 6.3 Implementar fluxo completo: consentimento, coleta, opções, solicitação, confirmação com .ics, cancelamento, remarcação e lembrete de véspera; verificar pelos cenários da spec `atendimento-responsaveis`
- [ ] 6.4 Implementar transferência para humano e pausa do bot; verificar que o agente não responde enquanto a conversa está com a equipe
- [ ] 6.5 Criar suíte de ~50 conversas roteirizadas e rodá-la no CI; verificar taxa de sucesso mínima definida

## 7. Privacidade (LGPD)

- [ ] 7.1 Implementar registro de consentimento com versão do aviso e verificar que sem consentimento nenhum dado da criança é gravado
- [ ] 7.2 Implementar minimização nas ofertas e no evento do Outlook; verificar por teste que a oferta contém nome, idade e endereço mas não telefone do responsável nem dados clínicos
- [ ] 7.3 Implementar retenção (expurgo de conversas em 90 dias) e fluxo de pedido de exclusão/acesso; verificar por teste do job de expurgo
- [ ] 7.4 Implementar log de acesso humano a dados de pacientes e verificar registro por teste
- [ ] 7.5 Redigir aviso de privacidade e lista de operadores/transferências internacionais em `docs/privacidade.md` para revisão jurídica

## 8. Piloto

- [ ] 8.1 Implantar em AWS sa-east-1 com alertas (ofertas expiradas, falhas do provedor, assinaturas do Graph a vencer) e verificar alerta de teste disparado
- [ ] 8.2 Rodar teste interno ponta a ponta com 2–3 fisios reais e verificar um agendamento completo família → fisio → Outlook → lembrete
- [ ] 8.3 Revisar após 2 semanas de piloto o log de decisões (ocupação e km por fisio) e ajustar pesos; registrar ajustes em `design.md`
