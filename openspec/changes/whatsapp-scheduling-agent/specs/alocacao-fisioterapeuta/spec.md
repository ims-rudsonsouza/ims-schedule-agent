# Spec Delta

## Purpose

Decidir quais fisioterapeutas podem atender um pedido e em que ordem ofertar, equilibrando deslocamento, carga de trabalho e preferência da família.

## ADDED Requirements

### Requirement: Elegibilidade por agenda
Uma fisioterapeuta SHALL ser elegível para um horário somente se o intervalo completo (sessão + tempo de deslocamento a partir do compromisso anterior + tempo de deslocamento até o compromisso seguinte + margem de relatório) couber na sua disponibilidade e não conflitar com nenhum evento da agenda.

#### Scenario: Deslocamento não cabe
- **WHEN** a fisio tem disponibilidade às 14h mas termina outro atendimento às 13h50 a 25 minutos de distância
- **THEN** ela não é elegível para 14h

### Requirement: Antecedência mínima e atendimento todos os dias
O sistema SHALL oferecer horários em qualquer dia da semana, inclusive fins de semana e feriados, sempre que houver bloco `Disponível`, e MUST NOT oferecer horário que comece em menos de 1 hora (antecedência mínima configurável). Para o mesmo dia, o horário SHALL deixar tempo para a fisio responder à oferta e se deslocar.

#### Scenario: Pedido para daqui a 40 minutos
- **WHEN** às 10h a família pede atendimento para 10h40
- **THEN** o agente explica que precisa de pelo menos 1 hora e oferece os primeiros horários a partir das 11h

#### Scenario: Feriado
- **WHEN** a família pede atendimento num feriado e uma fisio tem bloco `Disponível` nesse dia
- **THEN** o horário é oferecido normalmente

#### Scenario: Deslocamento maior que a folga
- **WHEN** às 10h a única fisio livre às 11h05 está a 50 minutos do endereço
- **THEN** 11h05 não é oferecido com ela, porque não sobra tempo para a resposta e o deslocamento

### Requirement: Preferência da família
Quando a família indicar uma fisioterapeuta preferida, o sistema SHALL ofertar primeiro os horários dessa fisio, independentemente de distância e carga. Se ela não tiver horário na janela pedida, o sistema SHALL perguntar se a família prefere outro horário com ela ou outra fisio. Outra fisio só SHALL ser considerada com a concordância da família.

#### Scenario: Fisio preferida disponível
- **WHEN** a família pede "quero com a Ana" e a Ana tem horário na janela
- **THEN** as opções oferecidas são da Ana, mesmo que haja fisio mais próxima

#### Scenario: Fisio preferida sem horário
- **WHEN** a Ana não tem horário na janela pedida
- **THEN** o agente oferece os próximos horários da Ana e, como alternativa, horários com outra fisio

### Requirement: Continuidade do tratamento
Para criança em tratamento, a fisioterapeuta que já a atende SHALL ser ofertada primeiro, salvo se a família pedir outra.

#### Scenario: Sessão de continuidade
- **WHEN** uma criança atendida pela Bia nas últimas sessões pede nova sessão sem citar fisio
- **THEN** a Bia é ofertada primeiro, se tiver horário

### Requirement: Raio de 10 km
Sem preferência, o sistema SHALL considerar primeiro as fisios elegíveis cuja distância de rota entre o ponto de partida no horário (residência ou atendimento anterior no dia) e o endereço da criança seja de até 10 km.

#### Scenario: Fisio vindo de atendimento próximo
- **WHEN** a fisio mora a 18 km mas terá atendimento anterior a 3 km do novo endereço
- **THEN** ela é considerada dentro do raio de 10 km para esse horário

### Requirement: Ranking por deslocamento e equilíbrio de carga
Entre as fisios dentro do raio, o sistema SHALL ordenar por uma pontuação que combina o tempo de deslocamento adicional e a taxa de ocupação da fisio (horas agendadas sobre horas disponíveis no período), com pesos configuráveis. A ordem resultante SHALL definir a sequência das ofertas.

#### Scenario: Empate de distância
- **WHEN** duas fisios estão a 15 minutos do endereço e uma tem 80% da agenda ocupada e a outra 20%
- **THEN** a fisio com 20% de ocupação é ofertada primeiro

#### Scenario: Diferença grande de distância
- **WHEN** uma fisio está a 5 minutos com 60% de ocupação e outra a 35 minutos com 40%
- **THEN** com os pesos padrão a fisio a 5 minutos é ofertada primeiro

### Requirement: Oferta aberta fora do raio
Quando nenhuma fisio elegível estiver a até 10 km, o sistema SHALL abrir a oferta para todas as fisios com agenda disponível no horário de interesse, conforme a spec de coordenação.

#### Scenario: Endereço distante de todas
- **WHEN** nenhuma fisio elegível está a até 10 km do endereço
- **THEN** todas as fisios livres naquele horário recebem a oferta ao mesmo tempo

### Requirement: Registro da decisão
O sistema SHALL registrar, para cada alocação, as candidatas avaliadas, suas pontuações e o motivo da escolha, para auditoria e ajuste dos pesos.

#### Scenario: Auditoria de uma alocação
- **WHEN** a equipe consulta um agendamento
- **THEN** é possível ver por que aquela fisio foi escolhida e quem mais foi considerada
