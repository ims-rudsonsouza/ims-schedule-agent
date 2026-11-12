# Proposta: agente de agendamento via WhatsApp para fisioterapia respiratória infantil

## Why

Hoje o agendamento das sessões domiciliares de fisioterapia respiratória infantil depende de uma secretária humana conciliando, por mensagem, a disponibilidade de 10 fisioterapeutas com a necessidade (muitas vezes urgente) dos pais. Um agente conversacional no WhatsApp, com tom humanizado, pode fazer esse papel 24h, reduzir o tempo de resposta às famílias, distribuir os atendimentos de forma mais justa entre as fisios e diminuir o tempo de deslocamento entre visitas.

Este documento substitui, para o MVP, partes do design anterior (`knowledge/whatsapp_scheduling_agent_design`): a agenda passa a ser a conta pessoal de cada fisio, Outlook ou Google Agenda, e o mesmo número de WhatsApp atende pais **e** fisioterapeutas.

## What Changes

- Novo agente conversacional em português (pt-BR) que atende pais/responsáveis pelo WhatsApp como uma secretária: coleta dados da criança e do endereço, entende a janela de horário desejada, oferece opções e confirma o agendamento.
- Novo motor de alocação que escolhe a fisioterapeuta considerando, nesta ordem: preferência explícita da família, raio de 10 km, menor tempo de deslocamento e equilíbrio de carga entre as fisios.
- Novo fluxo de coordenação com as fisioterapeutas pelo **mesmo número** de WhatsApp: oferta do atendimento, aceite/recusa, avisos de agendamento, cancelamento e remarcação.
- Regra de "oferta aberta": quando nenhuma fisio estiver a até 10 km, o horário é oferecido a **todas** as fisios com agenda livre naquele horário; a primeira que aceitar fica com o atendimento.
- Integração com as agendas pessoais das fisios, Outlook (Microsoft Graph) ou Google Agenda (Google Calendar API), como fonte da verdade: leitura da disponibilidade de cada fisio, gravação dos eventos confirmados e sincronização de mudanças feitas manualmente.
- Controles de privacidade para dados de saúde de crianças (LGPD arts. 11 e 14): consentimento do responsável, minimização do que é enviado às fisios e ao calendário, retenção e exclusão.
- Encaminhamento para atendimento humano quando o agente não resolver ou a família pedir.

## Capabilities

### New Capabilities

- `atendimento-responsaveis`: conversa com pais/responsáveis no WhatsApp — coleta de dados, oferta de horários, confirmação, cancelamento, remarcação e passagem para humano.
- `coordenacao-fisioterapeutas`: conversa com as fisioterapeutas pelo mesmo número — ofertas de atendimento, aceite/recusa, prazos de resposta e notificações.
- `alocacao-fisioterapeuta`: regras de elegibilidade e ranking (preferência, raio de 10 km, deslocamento, equilíbrio de carga) e a regra de oferta aberta.
- `agenda-fisioterapeutas`: conexão da conta pessoal de cada fisio (Outlook ou Google), leitura de disponibilidade e gravação/sincronização de eventos.
- `canal-whatsapp`: número único, identificação do papel (responsável x fisio) pelo telefone, janela de 24h e templates aprovados.
- `privacidade-lgpd`: consentimento, minimização de dados, controle de acesso, retenção e direitos do titular.

### Modified Capabilities

Nenhuma (o repositório ainda não tem specs).

## Impact

- **Sistemas externos novos**: WhatsApp Business Platform (via BSP, ex.: Twilio, ou Meta Cloud API direta), Microsoft Graph (contas Microsoft pessoais), Google Calendar API, Google Maps (geocodificação e tempo de rota), API da Anthropic (Claude) para a conversa.
- **Dados**: cadastro de responsáveis, crianças, fisioterapeutas (com endereço de residência), agendamentos e ofertas; todos sensíveis sob a LGPD.
- **Operação**: as fisios passam a receber ofertas e confirmar pelo WhatsApp; a agenda pessoal delas (Outlook ou Google) passa a ser lida e escrita pelo sistema.
- **Código**: projeto ainda sem código; esta mudança define a arquitetura e a especificação do MVP.
