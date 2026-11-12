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

### Requirement: Prazo de resposta e próxima candidata
Cada oferta individual SHALL ter prazo de resposta configurável (padrão 20 minutos; menor para pedidos no mesmo dia). Em caso de recusa ou estouro de prazo, o sistema SHALL ofertar à próxima candidata do ranking.

#### Scenario: Recusa
- **WHEN** a fisio recusa a oferta
- **THEN** a oferta é encerrada e a próxima fisio do ranking recebe a oferta

#### Scenario: Prazo esgotado
- **WHEN** o prazo termina sem resposta
- **THEN** a oferta expira, a fisio é avisada de que expirou e a próxima candidata recebe a oferta

### Requirement: Oferta aberta com primeira resposta vencedora
Na oferta aberta (sem fisio a até 10 km), o sistema SHALL enviar a oferta simultaneamente a todas as fisios elegíveis, e SHALL atribuir o atendimento à primeira que aceitar. As demais SHALL ser avisadas de que o horário foi preenchido.

#### Scenario: Duas fisios aceitam quase ao mesmo tempo
- **WHEN** duas fisios aceitam a mesma oferta aberta com segundos de diferença
- **THEN** apenas o primeiro aceite registrado é confirmado e a segunda recebe "esse horário já foi preenchido, obrigado!"

### Requirement: Contraproposta da fisioterapeuta
Além de aceitar ou recusar, a fisioterapeuta SHALL poder responder à oferta por texto livre sugerindo outro dia ou horário. O sistema SHALL interpretar a sugestão, verificar se cabe na agenda dela (com deslocamento) e, se couber, registrá-la como contraproposta para a família; se não couber, SHALL explicar o conflito e pedir outra sugestão.

#### Scenario: Fisio sugere outro horário
- **WHEN** a fisio responde à oferta de terça 14h com "terça não dá, consigo quarta às 10h"
- **THEN** o sistema confirma com ela "quarta 10h, certo? Vou consultar a família", verifica a agenda e registra a contraproposta

#### Scenario: Sugestão conflita com a agenda dela
- **WHEN** a fisio sugere quarta 10h mas já tem atendimento às 9h30 do outro lado da cidade
- **THEN** o agente explica que o horário não cabe com o deslocamento e pergunta se ela tem outra opção

#### Scenario: Pergunta antes de decidir
- **WHEN** a fisio pergunta "a criança mora em casa ou apartamento?" antes de responder
- **THEN** o agente responde com a informação do cadastro, ou diz que não tem e oferece perguntar à família, sem encerrar a oferta

### Requirement: Encaminhamento da contraproposta à família
Uma contraproposta SHALL encerrar a oferta original daquela fisio sem bloquear as demais candidatas para o horário original. O sistema SHALL apresentar a contraproposta à família quando nenhuma outra fisio aceitar o horário original, ou imediatamente quando a família tiver pedido aquela fisio especificamente. Se a família aceitar, o atendimento SHALL ser confirmado sem nova pergunta à fisio, após reverificar a agenda.

#### Scenario: Família com fisio preferida recebe a contraproposta
- **WHEN** a família pediu a Ana e a Ana sugere quarta 10h no lugar de terça 14h
- **THEN** a família recebe "A Ana não pode terça 14h, mas pode quarta às 10h. Fica bom?" e, se aceitar, o atendimento é confirmado e a Ana é avisada

#### Scenario: Outra fisio aceita o horário original
- **WHEN** a Bia sugeriu outro horário, mas a Carla aceita o horário original em seguida
- **THEN** o atendimento fica com a Carla e a Bia é avisada de que a sugestão não foi necessária

#### Scenario: Família recusa a contraproposta
- **WHEN** a família não aceita o horário sugerido pela fisio
- **THEN** a fisio é avisada e o agente oferece à família outros horários disponíveis

### Requirement: Aceite automático opcional
Uma fisioterapeuta SHALL poder ser configurada com aceite automático; nesse caso o sistema MUST confirmar sem esperar resposta e apenas notificá-la do novo agendamento.

#### Scenario: Fisio com aceite automático
- **WHEN** a alocação escolhe uma fisio com aceite automático
- **THEN** o agendamento é confirmado imediatamente e ela recebe a notificação com os dados completos

### Requirement: Notificações de mudanças
O sistema SHALL notificar a fisioterapeuta por WhatsApp sobre cancelamentos e remarcações dos seus atendimentos, e SHALL enviar um resumo da agenda do dia seguinte em horário configurável.

#### Scenario: Família cancela
- **WHEN** um responsável cancela um atendimento
- **THEN** a fisio recebe aviso com data e horário cancelados

### Requirement: Comandos da fisioterapeuta
A fisioterapeuta SHALL poder, pela conversa, consultar sua agenda, avisar indisponibilidade pontual e pedir para repassar um atendimento já aceito; o repasse SHALL disparar nova alocação e só liberar a fisio original quando outra aceitar ou a família for avisada.

#### Scenario: Fisio pede repasse
- **WHEN** a fisio escreve "não vou conseguir ir na de quinta 14h"
- **THEN** o sistema confirma qual atendimento, inicia nova alocação e avisa a equipe humana se ninguém aceitar
