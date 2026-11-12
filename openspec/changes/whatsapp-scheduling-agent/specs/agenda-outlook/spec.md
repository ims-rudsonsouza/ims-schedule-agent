# Spec Delta

## Purpose

Usar as agendas Outlook (Microsoft 365) das fisioterapeutas como fonte da verdade de disponibilidade e de compromissos, lendo e gravando via Microsoft Graph.

## ADDED Requirements

### Requirement: Uma agenda por fisioterapeuta
Cada fisioterapeuta SHALL ter uma agenda Outlook associada no cadastro, e o sistema SHALL ler e gravar somente nas agendas cadastradas.

#### Scenario: Cadastro de fisio
- **WHEN** uma fisio é cadastrada com sua agenda Outlook
- **THEN** o sistema passa a considerar a disponibilidade e os eventos dessa agenda

### Requirement: Leitura de disponibilidade
O sistema SHALL tratar como disponíveis os blocos marcados como disponibilidade na agenda da fisio e como ocupado qualquer outro evento, inclusive os criados manualmente fora do sistema.

#### Scenario: Evento pessoal criado no celular
- **WHEN** a fisio cria pelo Outlook do celular um evento "dentista" às 15h dentro de um bloco disponível
- **THEN** 15h deixa de ser oferecido para ela

### Requirement: Gravação do agendamento confirmado
Ao confirmar um atendimento, o sistema SHALL criar o evento na agenda da fisio contendo apenas: identificação curta do paciente (primeiro nome ou iniciais), endereço, telefone do responsável e código do agendamento, sem diagnóstico ou dados clínicos.

#### Scenario: Confirmação
- **WHEN** um atendimento é confirmado
- **THEN** o evento aparece na agenda Outlook da fisio com o horário reservado e os dados mínimos

### Requirement: Sincronização de mudanças externas
O sistema SHALL detectar em até 5 minutos alterações e exclusões feitas diretamente no Outlook em eventos criados por ele e SHALL refletir essas mudanças no agendamento, avisando a equipe quando a mudança afetar uma família.

#### Scenario: Fisio apaga o evento no Outlook
- **WHEN** a fisio exclui no Outlook um evento de atendimento
- **THEN** o sistema marca o agendamento como "a verificar", pergunta à fisio pelo WhatsApp se foi intencional e não avisa a família até a confirmação

### Requirement: Proteção contra conflito
O sistema MUST verificar novamente a agenda imediatamente antes de confirmar e MUST NOT confirmar um horário que tenha ficado ocupado desde a oferta.

#### Scenario: Horário ocupado durante a oferta
- **WHEN** a fisio aceita mas, nesse intervalo, um evento manual ocupou o horário
- **THEN** o aceite é recusado com explicação e a alocação segue para a próxima opção
