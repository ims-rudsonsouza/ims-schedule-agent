# Spec Delta

## Purpose

Operar um único número de WhatsApp Business para famílias e fisioterapeutas, identificando quem fala e respeitando as regras da plataforma.

## ADDED Requirements

### Requirement: Identificação do papel pelo telefone
O sistema SHALL identificar cada remetente pelo número de telefone: números cadastrados como fisioterapeuta seguem o fluxo de coordenação; os demais seguem o fluxo de responsáveis.

#### Scenario: Mensagem de uma fisio
- **WHEN** uma mensagem chega de um número cadastrado como fisioterapeuta
- **THEN** ela é tratada no fluxo de fisioterapeutas e nunca recebe o fluxo de agendamento de famílias

#### Scenario: Fisio que também é mãe de paciente
- **WHEN** uma fisio cadastrada pede para agendar para o próprio filho
- **THEN** o agente pergunta em qual papel ela está falando e segue o fluxo escolhido

### Requirement: Mensagens fora da janela de 24 horas
Mensagens iniciadas pelo sistema fora da janela de 24 horas da última mensagem do contato (ofertas, lembretes, avisos) MUST usar templates aprovados pela Meta.

#### Scenario: Oferta para fisio que não falou hoje
- **WHEN** o sistema precisa ofertar um atendimento a uma fisio cuja última mensagem foi há dois dias
- **THEN** a oferta é enviada como template aprovado com botões de resposta rápida

### Requirement: Processamento confiável de mensagens
O sistema SHALL processar cada mensagem recebida exatamente uma vez, mesmo com reenvio do webhook, e SHALL responder em até 10 segundos na maioria dos casos (indicando "digitando" quando possível).

#### Scenario: Webhook duplicado
- **WHEN** o provedor entrega a mesma mensagem duas vezes
- **THEN** o agente responde uma única vez

### Requirement: Lembretes à família
O sistema SHALL enviar lembrete ao responsável na véspera do atendimento, com opção de confirmar ou remarcar.

#### Scenario: Lembrete de véspera
- **WHEN** falta um dia para o atendimento
- **THEN** o responsável recebe o lembrete com botões "Confirmo" e "Preciso remarcar"
