# Template de System Design: Aula 04 - Microsserviços & Padrão Saga

> **Contexto do Cenário: O Pedido Pago Sem Produto na Black Friday**
>
> Sua equipe está arquitetando a transação distribuída de compra na Black Friday para o marketplace:
>
> 1. **O Incidente:** Um cliente comprou uma Smart TV de R$ 3.000 via Pix. O gateway de pagamentos debitou o valor imediatamente e confirmou a transação. No milissegundo seguinte, o serviço de Estoque esgotou as unidades e rejeitou a reserva. O dinheiro sumiu da conta do cliente, o produto não existe para entrega e o pedido travou em um estado inconsistente.
> 2. **O Desafio Distribuído:** Não existe transação única (`@Transactional`) entre bancos de dados autônomos de Checkout, Pagamento e Estoque.
> 3. **A Carga:** 5.000 TPS de checkout simultâneo e milhões de clientes consultando o status da compra em "Meus Pedidos".
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e Pragmatismo: 2PC vs Saga

- **Análise Crítica:** Em quais cenários arquiteturais uma transação ACID local ou Two-Phase Commit (2PC) resolveria o problema com menor custo operacional? Por que, para o cenário de 5.000 TPS na Black Friday com serviços e bancos autônomos, o 2PC é inviável e a consistência eventual se torna necessária?
  > Uma transação ACID local resolveria o problema com menor custo operacional quando todas as operações de negócio estivessem no mesmo banco de dados ou dentro de um monólito. Nesse cenário, o pagamento, a reserva de estoque e a criação do pedido poderiam ser executados dentro de uma única transação @Transactional, garantindo commit ou rollback atômico.

O Two-Phase Commit (2PC) também poderia resolver o problema em ambientes distribuídos menores, nos quais os serviços compartilham infraestrutura compatível com XA Transactions e possuem requisitos rigorosos de consistência forte. Nesse modelo, um coordenador central solicita que todos os participantes preparem a transação e somente efetiva o commit após a confirmação de todos.

Entretanto, para o cenário da Black Friday com 5.000 TPS, serviços independentes e bancos autônomos, o 2PC torna-se impraticável pelos seguintes motivos:

Introduz forte acoplamento entre os serviços participantes.
Mantém recursos bloqueados durante a coordenação da transação.
Aumenta significativamente a latência de cada checkout.
Reduz a disponibilidade do sistema quando algum participante está lento ou indisponível.
Cria um ponto central de coordenação que pode se tornar gargalo em eventos de pico.
Nem todos os bancos e tecnologias utilizadas pelos microsserviços oferecem suporte adequado a transações distribuídas XA.

Diante dessas restrições, a arquitetura precisa priorizar escalabilidade, disponibilidade e autonomia dos serviços, aceitando períodos curtos de inconsistência temporária. Por isso, a abordagem adequada é a Consistência Eventual, implementada através do padrão Saga, onde cada serviço executa sua própria transação local e, em caso de falha, são executadas ações compensatórias para restaurar a consistência do negócio.

### 1.2 O Modelo de Coordenação da Saga

- **A Escolha:** ( ) Coreografia (Orientada a Eventos)  ( ) Orquestração (Process Manager / Máquina de Estados)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Coreografia:* Como seu design resolve a falta de visibilidade centralizada (saber em tempo real onde o pedido está) e previne dependências cíclicas à medida que novos serviços entrarem no fluxo de compra?
  - *Se você escolheu Orquestração:* Qual componente atua como Orquestrador, como ele se comunica com os participantes (síncrono ou assíncrono) e como você evita que ele se transforme em um "serviço Deus", tirando a inteligência e a autonomia dos outros microsserviços?
  > Para o fluxo de compra da Black Friday, a Saga por Orquestração é a opção mais adequada porque fornece controle explícito do estado do pedido e visibilidade centralizada da transação distribuída.

O fluxo seria semelhante a:

Checkout cria o pedido com status PENDING.
Orquestrador envia comando para Pagamento.
Pagamento confirma cobrança e publica resposta.
Orquestrador envia comando para Estoque.
Estoque reserva item ou rejeita reserva.
Caso a reserva falhe, o Orquestrador dispara compensação para Pagamento (estorno Pix).
Pedido finaliza como CANCELLED ou CONFIRMED.
Componente Orquestrador

O componente responsável seria um serviço dedicado chamado, por exemplo:

Order Saga Orchestrator ou Checkout Workflow Service.

Sua responsabilidade é exclusivamente coordenar o fluxo da compra, mantendo uma máquina de estados como:
A comunicação entre o Orquestrador e os participantes deve ser predominantemente assíncrona, utilizando um broker como Kafka ou RabbitMQ.
A comunicação assíncrona reduz acoplamento, aumenta throughput e evita bloqueio de recursos durante picos de tráfego.
O Orquestrador não deve conter regras de negócio específicas de Pagamento ou Estoque.

Ele deve conhecer apenas:

Estados da Saga.
Próximos passos do fluxo.
Eventos de sucesso.
Eventos de falha.
Ações compensatórias.

As regras permanecem nos microsserviços especializados:

Pagamento

Autorizar Pix.
Realizar estorno.
Validar antifraude.

Estoque

Reservar produtos.
Controlar quantidade disponível.
Liberar reserva.

Checkout

Criar pedido.
Consultar status.

Dessa forma, o Orquestrador atua apenas como um coordenador de workflow, preservando a autonomia dos serviços e evitando concentração excessiva de responsabilidades.

Decisão Arquitetural Final

Padrão escolhido: Saga por Orquestração

Motivos principais:

Melhor rastreabilidade da compra.
Visibilidade centralizada do estado do pedido.
Facilidade de auditoria e observabilidade.
Fluxo de compensação explícito.
Menor complexidade operacional para equipes grandes.
Melhor adequação ao cenário de marketplace com alto volume (5.000 TPS) e múltiplos participantes.
Permite consistência eventual sem os gargalos e bloqueios do 2PC.
Forneça seus comentários sobre o BizChat

---

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de Estados e Fluxos Ponta a Ponta

- **Estados do Pedido:** Defina a máquina de estados explícita do ciclo de vida da transação no seu modelo (incluindo os estados iniciais, intermediários, de compensação e finais).

  > Como foi escolhida uma Saga por Orquestração, o pedido deve possuir uma máquina de estados explícita para permitir rastreabilidade, auditoria e recuperação em caso de falhas.
  CREATED
    ↓
PAYMENT_PROCESSING
    ↓
PAYMENT_CONFIRMED
    ↓
STOCK_RESERVING
    ↓
STOCK_RESERVED
    ↓
COMPLETED

Fluxo de compensação:

CREATED
    ↓
PAYMENT_PROCESSING
    ↓
PAYMENT_CONFIRMED
    ↓
STOCK_RESERVING
    ↓
STOCK_REJECTED
    ↓
REFUND_REQUESTED
    ↓
REFUND_COMPLETED
    ↓
CANCELLED

Fluxos de erro:

PAYMENT_PROCESSING
    ↓
PAYMENT_FAILED
    ↓
CANCELLED

ou

REFUND_REQUESTED
    ↓
REFUND_FAILED

Neste último caso, o pedido permanece em um estado que exige nova tentativa automática ou intervenção operacional.

EstadoDescriçãoCREATEDPedido criado pelo Checkout e Saga iniciada.PAYMENT_PROCESSINGSolicitação de cobrança enviada ao serviço de Pagamentos.PAYMENT_CONFIRMEDPagamento aprovado pelo gateway.PAYMENT_FAILEDCobrança rejeitada ou não processada.STOCK_RESERVSolicitação de reserva enviada ao Estoque.STOCK_RESERVEDEstoque reservado com sucesso.STOCKProduto indisponível ou reserva falhou.REFUND_REQUESTProcesso de estorno iniciado após falha na reserva.REFUND_COMPLETEDValor devolvido ao cliente.REFUND_FAILEDEstorno não concluído e necessita retentativa.COMPLETEDCompra concluída com sucesso.CANCELLEDSaga encerrada de forma compensada.

Fluxo de Sucesso (Happy Path)
Cliente confirma a compra.
Pedido entra em CREATED.
Orquestrador envia comando de pagamento.
Pedido muda para PAYMENT_PROCESSING.
Pagamento é aprovado.
Pedido muda para PAYMENT_CONFIRMED.
Orquestrador solicita reserva de estoque.
Pedido muda para STOCK_RESERVING.
Estoque reserva a unidade.
Pedido muda para STOCK_RESERVED.
Saga é finalizada.
Pedido muda para COMPLETED.

Fluxo de Compensação (Problema da Black Friday)
Cliente compra a TV.
Pedido entra em CREATED.
Pagamento é processado e aprovado.
Pedido muda para PAYMENT_CONFIRMED.
Orquestrador solicita reserva ao Estoque.
Estoque detecta que o último item acabou.
Pedido muda para STOCK_REJECTED.
Orquestrador dispara comando de estorno.
Pedido muda para REFUND_REQUESTED.
Pagamento devolve o valor ao cliente.
Pedido muda para REFUND_COMPLETED.
Saga é encerrada em CANCELLED.

Estados Finais da Saga

Existem apenas dois estados finais válidos para o negócio:

✅ COMPLETED

Pagamento confirmado.
Estoque reservado.
Pedido pronto para faturamento e entrega.

✅ CANCELLED

Alguma etapa falhou.
Todas as compensações necessárias foram executadas.
Cliente não ficou sem produto nem sem dinheiro.

Essa modelagem garante que, mesmo com 5.000 TPS e falhas distribuídas entre Pagamento e Estoque, o sistema mantenha rastreabilidade completa do pedido e alcance consistência eventual por meio das ações compensatórias da Saga.

- **Detalhamento dos Fluxos:**
  - **Caminho Feliz:** Descreva o fluxo de ponta a ponta da transação concluída com sucesso entre Checkout, Pagamento e Estoque.
  - **Caminho de Falha & Compensação:** Descreva o passo a passo a partir da falha na reserva de estoque após o pagamento aprovado. Como a ação compensatória (estorno) é disparada e executada sem intervenção humana?
  > Neste cenário, todas as etapas da Saga são executadas com sucesso e nenhuma compensação é necessária.

O cliente finaliza a compra da Smart TV no Checkout.
O serviço de Checkout cria o pedido com status CREATED e publica o evento OrderCreated.
O Orquestrador recebe o evento e altera o estado para PAYMENT_PROCESSING.
O Orquestrador envia o comando ProcessPayment para o serviço de Pagamentos.
O serviço de Pagamentos realiza a cobrança via Pix e registra sua transação local.
Após a confirmação do gateway, o serviço publica o evento PaymentApproved.
O Orquestrador recebe o evento e atualiza o pedido para PAYMENT_CONFIRMED.
Em seguida, envia o comando ReserveStock para o serviço de Estoque.
O serviço de Estoque verifica a disponibilidade e reserva a unidade da TV em sua própria transação local.
O serviço publica o evento StockReserved.
O Orquestrador recebe a confirmação e atualiza o pedido para STOCK_RESERVED.
A Saga é concluída e o pedido passa para COMPLETED.
O pedido fica disponível para faturamento, separação logística e entrega.

Cliente
   ↓
Checkout
   ↓ OrderCreated
Orquestrador
   ↓ ProcessPayment
Pagamento
   ↓ PaymentApproved
Orquestrador
   ↓ ReserveStock
Estoque
   ↓ StockReserved
Orquestrador
   ↓
COMPLETED

Este é exatamente o incidente descrito no cenário da Black Friday.

O cliente realiza a compra.
O Checkout cria o pedido em CREATED.
O Orquestrador solicita o processamento do pagamento.
O serviço de Pagamentos cobra os R$ 3.000 via Pix.
O gateway confirma a transação.
O serviço publica o evento PaymentApproved.
O pedido passa para PAYMENT_CONFIRMED.
O Orquestrador envia o comando ReserveStock.
O serviço de Estoque tenta reservar a última unidade disponível.
Durante a operação, identifica que outro pedido consumiu o estoque primeiro.
A reserva falha.
O serviço publica o evento StockRejected.

Disparo Automático da Compensação
O Orquestrador recebe o evento StockRejected.
A máquina de estados muda para REFUND_REQUESTED.
O Orquestrador envia automaticamente o comando RefundPayment.
O serviço de Pagamentos executa a operação de estorno.
O gateway confirma a devolução do valor ao cliente.
O serviço publica o evento PaymentRefunded.
O Orquestrador recebe a confirmação.
O estado muda para REFUND_COMPLETED.
A Saga é encerrada em CANCELLED.

Cliente
   ↓
Checkout
   ↓ OrderCreated
Orquestrador
   ↓ ProcessPayment
Pagamento
   ↓ PaymentApproved
Orquestrador
   ↓ ReserveStock
Estoque
   ↓ StockRejected
Orquestrador
   ↓ RefundPayment
Pagamento
   ↓ PaymentRefunded
Orquestrador
   ↓
CANCELLED

A compensação é totalmente automatizada por meio da própria Saga:

O Orquestrador monitora os eventos dos participantes.
Cada falha prevista possui uma ação compensatória associada.
Ao receber StockRejected, o Orquestrador sabe que deve executar RefundPayment.
O serviço de Pagamentos possui uma operação específica de estorno, capaz de desfazer a etapa anteriormente concluída.
Toda a comunicação ocorre de forma assíncrona por meio de eventos em Kafka ou RabbitMQ.
Caso haja falha temporária no estorno, mecanismos de retry, DLQ (Dead Letter Queue) e processamento idempotente garantem novas tentativas sem gerar duplicidade.

Dessa forma, mesmo sob uma carga de 5.000 TPS, o sistema consegue corrigir automaticamente inconsistências de negócio, assegurando que o cliente nunca fique simultaneamente sem o produto e sem o dinheiro, alcançando a consistência eventual proposta pelo padrão Saga.


### 2.2 Resiliência da Compensação e Consistência

- **E se a compensação falhar ou oscilar?** A chamada ou evento de estorno pode sofrer timeout ou instabilidade transitória. Como seu design garante que o cliente não fique no prejuízo se a tentativa de compensação falhar ou for reprocessada?
  > Em um ambiente distribuído com 5.000 TPS, deve-se assumir que falhas transitórias irão ocorrer. Portanto, a arquitetura não pode depender do sucesso imediato do estorno. O objetivo é garantir que a compensação seja executada eventualmente, mesmo diante de timeouts, indisponibilidades temporárias ou reprocessamentos.

  Quando o Orquestrador recebe o evento StockRejected, ele não realiza uma chamada HTTP síncrona aguardando resposta. Em vez disso, publica um comando RefundPayment em um broker (Kafka ou RabbitMQ).

  StockRejected
      ↓
Saga Orchestrator
      ↓
RefundPayment (evento/comando)
      ↓
Fila/Tópico
      ↓
Serviço de Pagamentos

Dessa forma, uma indisponibilidade momentânea do serviço de Pagamentos não interrompe a Saga.

Se o serviço de Pagamentos não conseguir processar o estorno devido a timeout, indisponibilidade do gateway Pix ou falha de rede, o comando é processado novamente automaticamente.

Tentativa 1 → Timeout
Tentativa 2 → Timeout
Tentativa 3 → Sucesso

As retentativas podem utilizar Exponential Backoff para evitar sobrecarga:

5s
10s
30s
60s
300s

Idempotência

Um dos maiores riscos em sistemas distribuídos é o reprocessamento de mensagens.

Por exemplo:

RefundPayment
      ↓
Gateway executa estorno
      ↓
Resposta se perde na rede
      ↓
Mensagem é reenviada

Sem proteção, o cliente poderia receber dois estornos.

Por isso, cada operação de compensação utiliza um identificador único:

refundId = ORDER-12345

Antes de executar o estorno, o serviço verifica:

Este refundId já foi processado?

Sim → retorna sucesso sem executar novamente.
Não → realiza o estorno e registra o processamento.

Assim, múltiplas execuções produzem exatamente o mesmo resultado.

Se todas as tentativas falharem, a mensagem é enviada para uma fila de exceção.

RefundPayment
      ↓
Retry 1
Retry 2
Retry 3
      ↓
DLQ

Isso impede a perda definitiva da solicitação de compensação.

A equipe operacional pode monitorar a DLQ e reprocessar as mensagens posteriormente sem impactar os clientes.

Estado intermediário explícito

Enquanto o estorno não é concluído, o pedido não deve ir diretamente para CANCELLED.

Ele permanece em:

REFUND_REQUESTED

ou

REFUND_IN_PROGRESS

Somente após o evento PaymentRefunded o pedido muda para:

REFUND_COMPLETED
      ↓
CANCELLED

REFUND_COMPLETED
      ↓
CANCELLED

Isso permite que tanto os sistemas internos quanto o cliente saibam exatamente o estágio da compensação.

Garantia de Consistência para o Cliente

O cliente nunca fica no prejuízo porque:

O estado da Saga registra que existe uma compensação pendente.
O comando de estorno é persistido e não se perde em caso de falha do sistema.
O processamento possui retentativas automáticas.
A operação é idempotente, permitindo reprocessamentos seguros.
Mensagens que excedem o número máximo de tentativas são enviadas para DLQ.
O pedido só é considerado encerrado após a confirmação do evento PaymentRefunded.

Decisão Arquitetural

A compensação será implementada com:

Comunicação assíncrona via broker de mensagens.
Retry com Exponential Backoff.
Operações idempotentes.
Dead Letter Queue (DLQ).
Máquina de estados explícita para rastrear compensações pendentes.

Dessa forma, mesmo diante de falhas temporárias, quedas de serviço ou reprocessamentos, a arquitetura garante consistência eventual, não perda de mensagens e proteção financeira do cliente, que são requisitos essenciais para um marketplace operando em escala de Black Friday.

- **Atomicidade entre Estado e Mensageria (Dual Write):** Se a sua arquitetura envolve publicação de eventos ou mensagens entre passos, como você garante que a persistência no banco local e o despacho da mensagem ocorram sem risco de perda silenciosa se a rede ou o broker falharem entre as duas operações?
  > Um dos principais riscos da arquitetura é o chamado Dual Write Problem:

  1. Atualiza o banco local ✅
2. Publica evento no broker ❌

ou

1. Publica evento no broker ✅
2. Atualiza o banco local ❌

Nesses casos, o sistema fica inconsistente porque uma operação ocorreu e a outra não.

Por exemplo:

Pedido → PAYMENT_CONFIRMED

foi persistido no banco, mas o evento PaymentApproved nunca chegou ao broker porque houve uma falha de rede imediatamente após o commit.

O resultado seria uma perda silenciosa da mensagem, fazendo a Saga ficar travada.

Solução: Transactional Outbox Pattern

Para evitar esse problema, utilizaria o padrão Transactional Outbox.

Em vez de publicar diretamente no Kafka/RabbitMQ dentro da transação de negócio, o serviço grava duas informações na mesma transação local:

BEGIN TRANSACTION

1. Atualiza Pedido
2. Insere evento na tabela OUTBOX

COMMIT

Exemplo:

Tabela Orders
------------------
order_id = 123
status = PAYMENT_CONFIRMED

Tabela Outbox
------------------
event_id = abc-123
event_type = PaymentApproved
status = PENDING

Como ambas as gravações acontecem no mesmo banco e na mesma transação ACID local, temos:

Pedido salvo + evento salvo.
Ou nada é salvo.

Nunca existe o cenário em que o pedido foi persistido e o evento desapareceu.

Após o commit, um componente dedicado (Outbox Publisher) lê continuamente os registros pendentes:

Outbox
   ↓
Publisher
   ↓
Kafka/RabbitMQ

Quando a publicação é confirmada:

status = SENT

Se o broker estiver indisponível:

status = PENDING

e novas tentativas serão realizadas posteriormente.

Pagamento aprovado
        ↓
Transação local
        ↓
+ Atualiza Pedido
+ Grava PaymentApproved na Outbox
        ↓
COMMIT
        ↓
Publisher
        ↓
Kafka
        ↓
Orquestrador

Mesmo que o Kafka fique indisponível por alguns minutos, o evento continua armazenado na Outbox e será publicado quando o broker retornar.

Como evitar eventos duplicados

Como o Publisher pode reenviar mensagens após falhas, todos os consumidores devem ser idempotentes.

Cada evento possui um identificador único:

eventId = abc-123

Antes de processar um evento, o consumidor verifica:

Já processei abc-123?

Sim → ignora.
Não → processa e registra.

Dessa forma, o sistema suporta a semântica at-least-once delivery, comum em Kafka e RabbitMQ.

Decisão Arquitetural

Para garantir atomicidade entre persistência do estado e publicação de mensagens, utilizaria:

Transactional Outbox Pattern para eliminar o Dual Write.
Publisher assíncrono para envio ao broker.
Retries automáticos em caso de indisponibilidade do broker.
Idempotência nos consumidores para lidar com reprocessamentos.
Persistência durável da Outbox até a confirmação do envio.

Essa abordagem garante que nenhuma transição crítica da Saga seja perdida, mesmo perante falhas de rede, reinício de serviços ou indisponibilidade temporária do broker, requisito essencial para um marketplace operando a 5.000 TPS durante a Black Friday.
---

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em Alta Vazão com CQRS

- **O Problema de Leitura:** Durante o pico de vendas, milhões de usuários atualizam a página de "Meus Pedidos" a cada poucos segundos.
- **Desenho da Solução:** Como você separa a responsabilidade de escrita (Command) da consulta de status (Query)? Como o modelo de leitura é alimentado e como sua arquitetura lida com a consistência eventual (o delay entre o pagamento e a visualização do status)?
  > Durante a Black Friday, o fluxo de escrita (criação de pedidos, pagamento e reserva de estoque) representa apenas uma parte da carga. O maior volume costuma vir das consultas dos clientes atualizando constantemente a tela "Meus Pedidos".

Se as consultas forem realizadas diretamente no banco transacional da Saga, teremos alguns problemas:

Competição entre leitura e escrita.
Aumento da latência das transações de checkout.
Sobrecarga dos bancos responsáveis por Pagamento, Estoque e Checkout.
Dificuldade para escalar leitura e escrita de forma independente.

Por isso, adotaria o padrão CQRS (Command Query Responsibility Segregation).

Separação entre Command e Query
Lado de Escrita (Command)

Responsável por alterar o estado do sistema.

Serviços envolvidos:

Checkout
Saga Orchestrator
Pagamento
Estoque

Fluxo:

Cliente
    ↓
Checkout
    ↓
Saga
    ↓
Pagamento
    ↓
Estoque

Cada serviço possui seu próprio banco transacional e executa apenas operações de negócio.

Lado de Leitura (Query)

Responsável exclusivamente pelas consultas realizadas pelos clientes.

Em vez de consultar os serviços transacionais, a aplicação consulta um banco otimizado para leitura.

Cliente
    ↓
API Meus Pedidos
    ↓
Read Database

Esse banco contém uma visão materializada do pedido:

{
  "orderId": "123",
  "status": "PAYMENT_CONFIRMED",
  "product": "Smart TV 65",
  "paymentStatus": "APPROVED",
  "lastUpdate": "2026-11-27T10:30:00"
}

Como o Modelo de Leitura é Alimentado

A atualização do modelo de leitura ocorre por eventos produzidos pela Saga.

Exemplo:

OrderCreated
PaymentApproved
StockReserved
PaymentRefunded
OrderCancelled

Esses eventos são publicados no broker.

Um serviço específico de projeção (Query Projection Service) consome os eventos e atualiza o banco de leitura.

Saga
    ↓
Kafka
    ↓
Projection Service
    ↓
Read Database

Exemplo:

PaymentApproved
    ↓
Atualiza Read Model:
status = PAYMENT_CONFIRMED

Posteriormente:

StockReserved
    ↓
Atualiza Read Model:
status = COMPLETED

Banco Otimizado para Consulta

Como milhões de clientes consultarão o status simultaneamente, o banco de leitura deve ser independente do banco transacional.

Possíveis opções:

PostgreSQL Read Replica
Elasticsearch
DynamoDB
Redis (para estados mais acessados)

A escolha depende dos requisitos não funcionais, mas o princípio permanece o mesmo: leitura desacoplada da escrita.

Como Lidar com a Consistência Eventual

Como o modelo de leitura é atualizado através de eventos assíncronos, existe um pequeno atraso entre a alteração do estado e sua visualização pelo usuário.

Exemplo:

10:00:00
Pagamento aprovado

10:00:01
Evento publicado

10:00:01
Evento publicado

Durante esse intervalo, o cliente pode visualizar o status anterior.

Essa é uma consequência natural da consistência eventual e é aceitável no contexto de marketplace, desde que o atraso seja pequeno (segundos).

Estratégias para Melhor Experiência do Usuário

Para reduzir a percepção do atraso:

Exibir estados intermediários claros:

Pagamento em processamento
Reserva de estoque em andamento
Estorno em processamento

Atualizar a tela por polling ou WebSocket.

Exibir a data/hora da última atualização.

Garantir que o Read Model seja atualizado em poucos segundos após cada evento.

Fluxo Arquitetural Completo

[COMMAND SIDE]

Cliente
    ↓
Checkout
    ↓
Saga Orchestrator
    ↓
Pagamento
    ↓
Estoque
    ↓
Eventos

==========================

[QUERY SIDE]

Kafka/RabbitMQ
    ↓
Projection Service
    ↓
Read Database
    ↓
API Meus Pedidos
    ↓
Cliente

Decisão Arquitetural

A arquitetura utilizará CQRS com modelo de leitura materializado, no qual:

Escritas são processadas pelos microsserviços da Saga.
Eventos de negócio alimentam um banco dedicado para consultas.
Leitura e escrita escalam independentemente.
Milhões de consultas à funcionalidade "Meus Pedidos" não impactam o checkout.
A consistência eventual é aceita e mitigada por estados intermediários e atualização rápida do modelo de leitura.

Essa abordagem atende ao requisito de 5.000 TPS de checkout e ao mesmo tempo suporta milhões de consultas concorrentes, preservando desempenho, escalabilidade e desacoplamento entre operações de leitura e escrita.

### 3.2 Estratégia de Testes da Transação Distribuída

- **Validação de Resiliência:** Como comprovar em ambiente de testes e pipeline que a Saga funciona de verdade?
- **Plano de Testes:** Descreva como você estruturaria o teste automatizado para simular a rejeição na reserva de estoque após a cobrança e validar que a compensação automática transiciona o pedido para cancelado com sucesso.
  > Validação de Resiliência

Apenas testes unitários não são suficientes para comprovar que a Saga funciona em produção. Como o comportamento depende da interação entre múltiplos serviços e mensagens assíncronas, é necessário validar cenários completos de ponta a ponta.

A estratégia de testes deve ser composta por múltiplas camadas:

Testes Unitários

Validam regras isoladas de cada serviço:

Aprovação e rejeição de pagamento.
Reserva e liberação de estoque.
Transições da máquina de estados da Saga.
Regras de compensação.

STOCK_REJECTED
    ↓
gera comando
RefundPayment

Testes de Integração

Validam a comunicação com componentes reais ou containers temporários:

Banco de dados.
Kafka/RabbitMQ.
Outbox.
Consumidores de eventos.

Ferramentas comuns:

Testcontainers
Docker
Kafka Container
PostgreSQL Container

O objetivo é garantir que os eventos sejam efetivamente publicados e consumidos.

3. Testes End-to-End da Saga

São os testes mais importantes para esse cenário.

Eles simulam o fluxo completo:

Checkout
↓
Pagamento
↓
Estoque
↓
Compensação

Validando que a Saga alcança o estado final esperado.

Testes de Falhas (Chaos/Resilience)

Simulam problemas reais:

Timeout no serviço de Pagamentos.
Broker indisponível.
Duplicação de mensagens.
Queda do Orquestrador durante a execução.
Falha temporária no estorno.

Objetivo:

Verificar se retries, idempotência e recuperação automática funcionam corretamente.

Cenário

Validar o incidente descrito no enunciado:

Pagamento aprovado com sucesso, seguida de rejeição na reserva de estoque.

Resultado esperado:

Pedido = CANCELLED
Pagamento = REFUNDED

Passo 1: Preparação do Ambiente

Subir todos os componentes necessários:

Checkout Service
Saga Orchestrator
Payment Service
Inventory Service
Kafka/RabbitMQ
Banco de Dados

Criar um produto sem estoque disponível:

Plain Text

TV_65
Quantidade = 0

Passo 2: Início da Compra

Enviar um comando:

{
  "orderId": "ORDER-123",
  "productId": "TV_65",
  "value": 3000
}

Verificar:

Estado = CREATED

Passo 3: Simular Pagamento Aprovado

Mockar ou configurar o serviço de pagamento para retornar sucesso.

Evento esperado:

PaymentApproved

Verificações:

paymentStatus = APPROVED
orderStatus = PAYMENT_CONFIRMED

Passo 4: Simular Falha no Estoque

Configurar o Inventory Service para rejeitar a reserva.

Evento publicado:

StockRejected

Verificações:

orderStatus = STOCK_REJECTED

Passo 5: Validar Disparo da Compensação

Confirmar que o Orquestrador publica automaticamente:

RefundPayment

O teste deve verificar:

Mensagem enviada ao broker.
Mensagem persistida na Outbox.
Consumo pelo serviço de Pagamentos.

Passo 6: Simular Estorno

O serviço de pagamento processa a compensação.

Evento esperado:

PaymentRefunded

Validações:

refundStatus = COMPLETED

Passo 7: Verificar Estado Final

O teste aguarda a conclusão assíncrona da Saga.

Estado esperado:

orderStatus = CANCELLED

Além disso:

paymentStatus = REFUNDED
stockReserved = false

Critérios de Sucesso

O teste deve validar simultaneamente:

✅ O pagamento foi aprovado.

✅ A reserva de estoque falhou.

✅ O comando de compensação foi disparado.

✅ O estorno foi executado.

✅ Nenhum item ficou reservado.

✅ O pedido terminou em CANCELLED.

✅ Não houve intervenção humana.

Teste Adicional de Resiliência

Um segundo teste deve simular a falha do estorno na primeira tentativa:

RefundPayment
    ↓
Timeout

Resultado esperado:

Retry automático
    ↓
PaymentRefunded
    ↓
CANCELLED


Esse cenário comprova que a arquitetura suporta falhas transitórias e continua garantindo a consistência eventual.

Decisão Arquitetural

A validação da Saga será realizada através de testes End-to-End automatizados executados no pipeline CI/CD, simulando tanto o fluxo feliz quanto os cenários de falha. O principal teste reproduz exatamente o incidente da Black Friday: pagamento aprovado seguido de rejeição de estoque, comprovando que a compensação automática executa o estorno e finaliza o pedido corretamente em CANCELLED, garantindo que nenhum cliente fique sem produto e sem dinheiro.


