# Spec Delta

## Purpose

Proteger dados pessoais e de saúde de crianças, responsáveis e fisioterapeutas conforme a LGPD, em especial os arts. 11 (dados sensíveis) e 14 (crianças e adolescentes).

## ADDED Requirements

### Requirement: Consentimento do responsável
Antes de coletar qualquer dado da criança, o sistema SHALL obter consentimento específico e destacado de um dos pais ou responsável legal, com link para o aviso de privacidade, e SHALL registrar data, hora, versão do aviso e o número que consentiu.

#### Scenario: Responsável recusa
- **WHEN** o responsável não aceita o aviso de privacidade
- **THEN** o agente não coleta dados da criança e oferece contato com a equipe humana por telefone

### Requirement: Minimização de dados
O sistema SHALL coletar somente os dados necessários para agendar e realizar a visita, e MUST NOT pedir diagnóstico ou detalhes clínicos pelo WhatsApp além do motivo geral do atendimento.

#### Scenario: Responsável envia exame pelo chat
- **WHEN** o responsável envia foto de exame ou laudo
- **THEN** o agente não armazena o conteúdo no histórico de longo prazo e orienta entregá-lo diretamente à fisio na visita

### Requirement: Acesso restrito por fisioterapeuta
Cada fisioterapeuta SHALL ter acesso apenas aos dados dos atendimentos que aceitou; ofertas não aceitas MUST conter somente dados não identificáveis da criança.

#### Scenario: Fisio que recusou a oferta
- **WHEN** uma fisio recusou uma oferta
- **THEN** ela nunca recebe nome nem endereço completo daquela criança

### Requirement: Retenção e exclusão
O sistema SHALL apagar o histórico de conversa após prazo definido (padrão 90 dias) e SHALL atender pedidos do responsável de acesso, correção e exclusão dos dados, respeitando obrigações legais de guarda.

#### Scenario: Pedido de exclusão
- **WHEN** o responsável escreve "quero que apaguem meus dados"
- **THEN** o pedido é registrado, a equipe é avisada e o responsável recebe confirmação com prazo de atendimento

### Requirement: Segurança e registro de acesso
Os dados SHALL ser criptografados em trânsito e em repouso, armazenados no Brasil, e todo acesso humano a dados de pacientes SHALL ser registrado.

#### Scenario: Consulta da equipe
- **WHEN** alguém da equipe consulta o cadastro de uma criança
- **THEN** o acesso fica registrado com usuário, data e motivo
