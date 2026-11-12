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
Cada oferta individual SHALL ter prazo de resposta configurável (padrão 20 minutos; menor para pedidos no mesmo dia). Em caso de recusa ou estouro de prazo, o sistema SHALL ofertar à próxima candidata do ranking, exceto quando a fisio for a pedida pela família (ver requisito de recusa da fisio pedida).

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
Além de aceitar ou recusar, a fisioterapeuta SHALL poder responder à oferta por texto livre sugerindo outro dia ou horário. O sistema SHALL interpretar a sugestão, verificar se cabe na agenda dela (com deslocamento) e, se couber, abrir uma negociação com a família; se não couber, SHALL explicar o conflito e pedir outra sugestão.

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
- **THEN** as ofertas abertas às demais fisios são suspensas e elas são avisadas de que o horário está em negociação

### Requirement: Fim da negociação sem acordo
A negociação SHALL terminar sem acordo quando a família ou a fisio desistir, ou quando passar o prazo máximo (padrão 2 horas, ou 3 rodadas sem acordo). Ao terminar sem acordo, o sistema SHALL avisar a fisio e seguir a regra de fallback: com fisio pedida, perguntar à família se outra fisio pode atender; sem fisio pedida, retomar a alocação com as próximas candidatas.

#### Scenario: Família pediu a fisio e a negociação falha
- **WHEN** a família pediu a Ana e não chegaram a um horário comum
- **THEN** o agente diz à família que não foi possível com a Ana e pergunta se ela aceita ser atendida por outra fisio

#### Scenario: Sem fisio pedida e a negociação falha
- **WHEN** a família não pediu fisio e a negociação com a Bia termina sem acordo
- **THEN** o sistema retoma as ofertas para as próximas candidatas, começando pelo horário escolhido pela família, se ainda estiver livre

#### Scenario: Prazo da negociação esgotado
- **WHEN** a negociação fica 2 horas sem acordo
- **THEN** ela é encerrada sem acordo, as duas partes são avisadas e a regra de fallback é aplicada

### Requirement: Recusa da fisio pedida pela família
Quando a fisio pedida explicitamente pela família recusar a oferta ou deixar o prazo expirar, o sistema MUST NOT ofertar o pedido a outra fisio automaticamente. O agente SHALL avisar a família de que essa fisio não está disponível e perguntar se outra fisio pode atender; só com o "sim" da família o sistema SHALL seguir para a alocação normal sem a fisio pedida.

#### Scenario: Ana recusa
- **WHEN** a família pediu a Ana e a Ana recusa a oferta
- **THEN** a família recebe "A Ana não está disponível nesse horário. Posso ver com outra fisioterapeuta?" e nenhuma outra fisio é acionada antes da resposta

#### Scenario: Família aceita outra fisio
- **WHEN** a família responde que pode ser outra fisio
- **THEN** o sistema roda a alocação normal (raio, deslocamento, carga) sem a Ana e oferta o horário

#### Scenario: Família prefere esperar a fisio pedida
- **WHEN** a família responde que só quer a Ana
- **THEN** o agente oferece os próximos horários livres da Ana

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
