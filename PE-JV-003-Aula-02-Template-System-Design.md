# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** ( ) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)? Spring MVC com Virtual Threads
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > A escolha por Spring MVC com Virtual Threads foi feita porque entrega escala massiva (10k usuários simultâneos) com simplicidade de implementação e manutenção, sem exigir a curva de aprendizado da programação reativa. É a solução mais prática e eficiente para o cenário de Checkout 

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > Circuit Breaker → abre quando há falhas consecutivas, evitando sobrecarga e liberando rapidamente respostas de erro para o cliente.

TimeLimiter → define um tempo máximo de resposta (ex.: 2 segundos). Se o parceiro demorar mais, a requisição é cancelada e a thread liberada.

Retry com backoff exponencial → reenvia chamadas em falhas transitórias (ex.: timeout ou erro de rede), mas com intervalos crescentes para não bombardear o serviço externo.

Bulkhead → isola recursos (threads/conexões) para que falhas no parceiro não afetem todo o sistema de checkout.

RateLimiter → controla a taxa de chamadas, respeitando limites de API e evitando bloqueios por excesso de requisições.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > A Idempotency-Key será gerada no início da transação de checkout (ex.: UUID).

Essa chave é enviada junto com a requisição ao parceiro de pagamentos.

O parceiro armazena a chave e associa ao resultado da operação (aprovada, recusada ou expirada).

Se a mesma requisição for enviada novamente com a mesma chave, o parceiro retorna o mesmo resultado, sem processar um novo débito.

Ciclo de vida da chave:

Criação → gerada no início do checkout.

Persistência → armazenada temporariamente no parceiro ou em cache.

Validade → dura até a conclusão da operação (ex.: até 24h).

Expiração → após o tempo configurado, a chave é descartada.

Não reutilizável → cada pagamento deve ter sua própria chave.

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. **E se o banco de dados cair na hora de registrar a transação de pagamento?** [Ex: o banco de dados cair na hora de atualizar o estado do Circuit Breaker?]
   - **Resposta do Sistema:**O sistema deve usar fila de mensagens (Kafka/SQS) para garantir persistência assíncrona. Se o banco estiver indisponível, a transação fica na fila e será processada quando o banco voltar, evitando perda de dados.
2. **E se o parceiro de pagamentos ficar indisponível por vários minutos durante o pico de acessos?** 
   - **Resposta do Sistema:** O Circuit Breaker deve abrir após falhas consecutivas, bloqueando novas chamadas e retornando uma resposta rápida ao cliente (ex.: “Serviço temporariamente indisponível”). Enquanto isso, o sistema pode registrar a tentativa em fila para reprocessar quando o parceiro voltar.
3. **E se o cliente clicar várias vezes em “Pagar” ou a requisição for reenviada por causa de um retry automático?** 
   - **Resposta do Sistema:** A Idempotency-Key garante que apenas uma cobrança real seja registrada. Chamadas repetidas com a mesma chave retornam o mesmo resultado, evitando duplicidade de débito.

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > Simulação de falhas consecutivas

Configurar o ambiente de teste para que o parceiro de pagamentos retorne erros (HTTP 500) em sequência.

Monitorar se o Circuit Breaker abre após o número configurado de falhas (ex.: 5 falhas seguidas).

Validar que, enquanto o circuito está aberto, novas requisições são bloqueadas rapidamente, sem tentar chamar o serviço externo.

Simulação de lentidão extrema

Introduzir latência artificial no parceiro (ex.: resposta em 10 segundos).

Configurar o TimeLimiter para 2 segundos.

Garantir que a requisição seja interrompida após o tempo limite e que a thread seja liberada para atender outras requisições, evitando bloqueio.

Testes de carga e concorrência

Usar ferramentas como JMeter ou Gatling para simular 10.000 requisições simultâneas.

Observar se o Circuit Breaker abre em momentos de falha massiva e se o Timeout evita acúmulo de threads presas.

Observabilidade e métricas

Integrar Micrometer + Prometheus + Grafana para monitorar:

Estado do Circuit Breaker (fechado, aberto, meio-aberto).

Número de timeouts disparados.

Latência média e taxa de erro.

Configurar alertas automáticos para detectar quando o circuito abre ou quando há excesso de timeouts.
