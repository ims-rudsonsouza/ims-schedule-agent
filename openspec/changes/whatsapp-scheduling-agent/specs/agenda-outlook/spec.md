# Spec Delta

## Purpose

Usar as agendas Outlook das contas pessoais das fisioterapeutas como fonte da verdade de disponibilidade e de compromissos, lendo e gravando via Microsoft Graph.

## ADDED Requirements

### Requirement: Uma agenda por fisioterapeuta
Cada fisioterapeuta SHALL conectar a sua própria conta Outlook (e-mail pessoal) autorizando o acesso do sistema, que SHALL criar nela uma agenda dedicada aos atendimentos. O sistema SHALL gravar somente nessa agenda dedicada.

#### Scenario: Conexão da conta
- **WHEN** uma fisio cadastrada abre o link de conexão recebido pelo WhatsApp e autoriza o acesso
- **THEN** o sistema cria a agenda "Atendimentos IMS" na conta dela e passa a considerar a disponibilidade e os eventos dessa conta

#### Scenario: Fisio ainda não conectada
- **WHEN** uma fisio cadastrada ainda não conectou a conta
- **THEN** ela não recebe ofertas e a equipe vê o status "não conectada"

### Requirement: Perda de conexão com a agenda
Quando o acesso à agenda de uma fisio deixar de funcionar (permissão revogada, senha alterada, token expirado), o sistema SHALL parar de oferecer atendimentos a ela e SHALL enviar a ela e à equipe um aviso com link para reconectar.

#### Scenario: Fisio revoga o acesso
- **WHEN** a fisio remove a permissão do app na conta Microsoft dela
- **THEN** na próxima sincronização ela sai das ofertas e recebe no WhatsApp um link para reconectar

### Requirement: Privacidade dos compromissos pessoais
O sistema SHALL ler dos eventos pessoais da fisio apenas início, fim e status de ocupado, e MUST NOT ler, armazenar ou exibir título, local, descrição ou participantes desses eventos.

#### Scenario: Compromisso pessoal na agenda principal
- **WHEN** a fisio tem "Consulta médica" das 15h às 16h na agenda principal
- **THEN** o sistema registra apenas que ela está ocupada das 15h às 16h

### Requirement: Leitura de disponibilidade
O sistema SHALL tratar como disponíveis os blocos marcados como disponibilidade na agenda da fisio e como ocupado qualquer outro evento, inclusive os criados manualmente fora do sistema.

#### Scenario: Evento pessoal criado no celular
- **WHEN** a fisio cria pelo Outlook do celular um evento pessoal às 15h dentro de um bloco disponível
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
