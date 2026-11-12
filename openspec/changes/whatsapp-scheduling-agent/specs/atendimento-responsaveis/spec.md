# Spec Delta

## Purpose

Conduzir, pelo WhatsApp e em tom humanizado, a conversa com pais e responsáveis para agendar, remarcar e cancelar sessões domiciliares de fisioterapia respiratória infantil. O foco é o primeiro contato, uma sessão por vez; as sessões seguintes são combinadas pessoalmente com a fisio.

## ADDED Requirements

### Requirement: Conversa humanizada em português
O agente SHALL conversar em português do Brasil, com tom cordial e acolhedor de secretária, mensagens curtas e uma pergunta por vez. O agente SHALL se apresentar como assistente virtual da clínica na primeira interação e nunca afirmar ser uma pessoa.

#### Scenario: Primeira mensagem de um responsável
- **WHEN** um número desconhecido envia "Oi, preciso marcar fisio pro meu filho"
- **THEN** o agente cumprimenta, se apresenta como assistente virtual da clínica e pede o consentimento de privacidade antes de qualquer pergunta sobre a criança

#### Scenario: Pergunta direta se é um robô
- **WHEN** o responsável pergunta "você é uma pessoa?"
- **THEN** o agente responde que é um assistente virtual e oferece falar com a equipe humana

### Requirement: Coleta dos dados mínimos para agendar
Antes de buscar horários, o agente SHALL obter: nome do responsável, nome e idade da criança, endereço completo do atendimento, janela de data/horário desejada e se há preferência por alguma fisioterapeuta. Para responsáveis já cadastrados, o agente SHALL reutilizar os dados e apenas confirmá-los.

#### Scenario: Responsável recorrente
- **WHEN** um responsável já cadastrado pede nova sessão
- **THEN** o agente confirma "mesmo endereço (Rua X, 123)?" em vez de pedir tudo de novo e pergunta se quer manter a mesma fisio

#### Scenario: Endereço ambíguo
- **WHEN** o endereço informado não pode ser localizado com precisão
- **THEN** o agente pede complemento (CEP, número ou ponto de referência) ou a localização do WhatsApp antes de seguir

### Requirement: Oferta de opções concretas
O agente SHALL oferecer de duas a três opções de horário reais, já verificadas na agenda, com dia, horário e nome da fisioterapeuta. O agente MUST NOT oferecer horários que não tenham sido retornados pela alocação.

#### Scenario: Horários disponíveis
- **WHEN** existem fisios elegíveis na janela pedida
- **THEN** o agente apresenta até três opções numeradas e pede que o responsável escolha uma

#### Scenario: Nenhum horário na janela pedida
- **WHEN** não há disponibilidade na janela pedida
- **THEN** o agente informa isso e sugere os horários mais próximos fora da janela ou pergunta por outra janela

### Requirement: Confirmação pendente com a fisioterapeuta
Como todo atendimento exige aceite da fisioterapeuta, o agente SHALL informar ao responsável que está confirmando com a fisio e SHALL avisá-lo assim que houver aceite, recusa, estouro de prazo ou proposta de outro horário pela fisio.

#### Scenario: Aceite recebido
- **WHEN** a fisio aceita a oferta
- **THEN** o responsável recebe o resumo (fisio, data, horário, endereço, duração) e um convite .ics

#### Scenario: Nenhuma fisio aceitou no prazo
- **WHEN** o prazo de resposta das fisios acaba sem aceite
- **THEN** o agente avisa o responsável, pede desculpas e oferece outros horários ou atendimento humano

#### Scenario: Fisio propõe outro horário
- **WHEN** a fisio responde com outro dia ou horário
- **THEN** o agente apresenta a proposta ao responsável, leva a resposta de volta à fisio e repete até haver acordo ou desistência

### Requirement: Cancelamento e remarcação pela conversa
O responsável SHALL poder cancelar ou remarcar um agendamento futuro pelo mesmo chat; o agente SHALL confirmar qual agendamento será alterado antes de executar e SHALL avisar a fisioterapeuta envolvida.

#### Scenario: Cancelamento
- **WHEN** o responsável escreve "preciso cancelar a de amanhã"
- **THEN** o agente identifica o agendamento, pede confirmação, cancela, libera o horário na agenda e notifica a fisio

### Requirement: Passagem para atendimento humano
O agente SHALL encaminhar a conversa para uma pessoa da equipe quando o responsável pedir, quando não conseguir entender após duas tentativas, ou quando houver relato de sinais de gravidade, e SHALL parar de responder automaticamente naquela conversa até a equipe devolvê-la.

#### Scenario: Sinal de gravidade
- **WHEN** o responsável relata, por exemplo, "ele está roxinho e com muita falta de ar"
- **THEN** o agente orienta procurar atendimento de emergência imediatamente (SAMU 192 ou pronto-socorro), não tenta agendar e alerta a equipe humana
