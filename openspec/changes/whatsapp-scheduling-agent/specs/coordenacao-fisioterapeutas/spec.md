# Spec Delta

## Purpose

Coordenar com as fisioterapeutas, pelo mesmo número de WhatsApp usado com as famílias, a oferta, o aceite e as mudanças de atendimentos.

## ADDED Requirements

### Requirement: Oferta de atendimento para a fisioterapeuta
O sistema SHALL enviar à fisioterapeuta uma oferta com data, horário, bairro, distância/tempo estimado de deslocamento e idade da criança, com opções "Aceitar" e "Recusar". O endereço completo e o nome da criança MUST ser enviados somente após o aceite.

#### Scenario: Oferta enviada
- **WHEN** a alocação escolhe uma fisio para um pedido
- **THEN** ela recebe a oferta com botões de resposta rápida e sem o endereço completo

#### Scenario: Aceite
- **WHEN** a fisio toca em "Aceitar" dentro do prazo e o horário ainda está livre
- **THEN** o agendamento é confirmado, o evento é criado na agenda dela e ela recebe o endereço completo e o nome da criança

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
