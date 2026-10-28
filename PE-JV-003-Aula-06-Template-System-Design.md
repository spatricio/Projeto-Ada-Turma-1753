# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.
  > Para identificar rapidamente uma requisição problemática entre milhares de linhas de log, todos os serviços (Checkout, Pagamento e Estoque) devem produzir logs estruturados contendo os seguintes campos:

Campo	Finalidadetimestamp	Indica o momento exato do evento, permitindo reconstruir a linha do tempo do incidente.
level (INFO, WARN, ERROR)	Facilita a filtragem por criticidade.
service_name	Identifica qual serviço gerou o log (Checkout, Pagamento ou Estoque).
environment	Diferencia ambientes como produção, homologação ou canário.
version	Permite identificar se a ocorrência está relacionada à versão nova implantada no Canary.
trace_id	Identificador único da requisição ponta a ponta. É o principal campo para rastrear o fluxo completo.
span_id	Identifica uma operação específica dentro do trace distribuído.
parent_span_id	Permite reconstruir a árvore de chamadas entre serviços.
http_method	Método utilizado na requisição (GET, POST, etc.).
endpoint	Endpoint acessado.
status_code	Código de resposta retornado.
duration_ms	Tempo de processamento da operação. Fundamental para identificar gargalos.
error_message	Descrição do erro quando houver falha.
host/pod_id	Identifica a instância que processou a requisição.
canary ou deployment_group	Permite distinguir tráfego da versão estável e da versão Canary.

Justificativa: Como os painéis mostram métricas agregadas saudáveis, a investigação precisa acontecer no nível de requisição individual. O campo mais importante nesse cenário é o trace_id, que conecta todos os eventos relacionados ao mesmo pedido.

- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?
  > O identificador de rastreamento deve ser criado no início da requisição no Checkout e propagado automaticamente para os demais serviços através dos cabeçalhos HTTP.

Fluxo:

O cliente chama o serviço Checkout.
O Checkout cria (ou recebe) um trace_id.
Ao chamar Pagamento, o Checkout envia o trace_id nos headers HTTP.
O serviço de Pagamento registra o mesmo trace_id em seus logs e traces.
Se Pagamento chamar outro serviço, o mesmo identificador continua sendo propagado.
O mesmo processo ocorre na chamada para Estoque.

Exemplo de header seguindo o padrão OpenTelemetry:

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

Com isso, torna-se possível visualizar uma única transação atravessando Checkout → Pagamento → Estoque e identificar exatamente em qual etapa ocorreu o travamento ou timeout.

- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?
  > Dados pessoais ou sensíveis não devem ser registrados em logs nem traces.

Devem ficar fora dos logs:

Nome completo do cliente
CPF
RG
Endereço
Telefone
E-mail
Número do cartão
CVV
Dados bancários
Senhas
Tokens de autenticação
Cookies de sessão
Informações de pagamento completas

Dados que podem aparecer mediante tratamento adequado:

Dado	TratamentoID do cliente	Hash ou pseudonimização
ID do pedido	Identificador interno sem informação pessoal
E-mail	Mascaramento parcial (jo***@dominio.com) quando realmente necessário
IP	Mascaramento ou anonimização conforme política da empresa

Controle de acesso:

Logs e traces devem ficar armazenados em plataforma centralizada com controle de acesso baseado em papéis (RBAC).
Apenas equipes autorizadas (SRE, Operações e Engenharia responsável pelo serviço) podem visualizar registros completos.
Todo acesso deve ser auditado.
Deve existir retenção limitada e descarte dos dados conforme a política de LGPD.

Justificativa: O cenário exige observabilidade para diagnóstico, mas também informa explicitamente que os dados de cliente permanecem sujeitos à LGPD. Portanto, é necessário adotar minimização de dados, mascaramento e controle rigoroso de acesso, garantindo rastreabilidade sem exposição de informações pessoais.

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.
  > Visão do Cliente (RED)

Para o serviço de Checkout, o painel deve seguir o modelo RED (Rate, Errors, Duration), focando na experiência do cliente e não apenas nos recursos da infraestrutura.

Rate (Taxa)

Quantidade de requisições por segundo (TPS).
Quantidade de pedidos iniciados.
Quantidade de pedidos concluídos com sucesso.
Taxa de conversão do checkout.

Errors (Erros)

Taxa de erros HTTP 5xx.
Taxa de erros HTTP 4xx.
Quantidade de timeouts.
Taxa de falhas nas chamadas para Pagamento e Estoque.

Duration (Duração)

Tempo de resposta do endpoint "Finalizar Pedido".
Latência das chamadas para Pagamento.
Latência das chamadas para Estoque.
Tempo total da jornada do checkout.
Como medir a duração sem esconder o problema na média

Neste cenário, utilizar apenas a latência média é insuficiente e foi justamente o que mascarou o incidente.

Exemplo:

95% das requisições respondem em 120 ms.
5% ficam travadas por 30 segundos até ocorrer timeout.

A média pode continuar parecendo saudável, enquanto milhares de clientes estão enfrentando uma experiência ruim.

Por isso, a duração deve ser medida utilizando histogramas e percentis, não apenas médias.

Métricas recomendadas:

p50 (mediana)
p90
p95
p99
máximo observado (max)

Exemplo de painel:

Métrica	ValorMédia	120 ms
p50	110 ms
p95	250 ms
p99	8 s
Máximo	30 s

Nesse cenário, mesmo com média de 120 ms, o p99 de 8 segundos evidenciaria imediatamente que parte dos usuários está enfrentando lentidão severa.

Além disso, os percentis devem ser segmentados por:

Versão da aplicação (Stable x Canary)
Endpoint
Região
Serviço dependente (Pagamento ou Estoque)

Dessa forma, seria possível perceber rapidamente que a nova versão Canary introduziu uma degradação que afeta apenas uma parcela das requisições, algo que a média isoladamente não consegue revelar.

Justificativa: Como o incidente ocorreu durante uma implantação Canary com 5.000 TPS e os dashboards permaneceram verdes mesmo com clientes enfrentando timeouts, a observabilidade deve priorizar percentis de latência (especialmente p95 e p99) e métricas de timeout. Isso garante que problemas que afetam apenas parte do tráfego não sejam escondidos por valores médios aparentemente saudáveis.

- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?
  > Visão dos Recursos (USE)

Para complementar a visão do cliente (RED), o monitoramento deve seguir o método USE (Utilization, Saturation, Errors), focando nos recursos dos serviços Checkout, Pagamento e Estoque.

Recursos monitorados

CPU

Utilização total de CPU (%).
Tempo de CPU por pod/instância.
Throttling de CPU em containers.

Memória

Consumo de memória.
Percentual da memória disponível.
Ocorrências de OOM (Out Of Memory).
Crescimento contínuo do heap (possível vazamento de memória).

Rede

Taxa de transferência (entrada e saída).
Latência de comunicação entre serviços.
Perda de pacotes.
Erros de conexão.

Threads e Conexões

Quantidade de threads ativas.
Pool de conexões HTTP.
Pool de conexões com banco de dados.
Requisições aguardando recurso disponível.

Disco (quando aplicável)

Tempo de I/O.
Fila de I/O.
Espaço disponível.
Taxa de leitura e escrita.

Aplicação

Tamanho das filas internas.
Quantidade de requisições em andamento.
Taxa de timeouts em dependências.
Circuit breakers abertos.
Que sinal acusaria saturação antes do primeiro HTTP 500?

Os sinais mais importantes são os de saturação, pois normalmente aparecem antes dos erros visíveis para o cliente.

Exemplos:

Crescimento contínuo da fila de requisições pendentes.
Aumento do tempo de espera em pools de conexão.
Elevação dos percentis de latência (p95 e p99).
Muitas threads bloqueadas aguardando resposta de Pagamento ou Estoque.
Pool de conexões próximo de 100% de utilização.
Throttling de CPU.
Aumento da quantidade de requisições em timeout.
Crescimento do número de requisições simultâneas em andamento.

No cenário apresentado, o sinal mais relevante seria o aumento de requisições presas aguardando respostas de dependências, acompanhado da elevação do p95/p99 de latência e do uso do pool de conexões, mesmo com CPU abaixo de 40% e sem HTTP 500.

Isso ocorre porque um serviço pode ficar saturado por espera (threads, conexões ou chamadas remotas bloqueadas) sem necessariamente consumir muita CPU. Dessa forma, a plataforma começaria a apresentar filas, lentidão e timeouts muito antes de gerar erros HTTP 500, permitindo detectar o problema da versão Canary de forma antecipada.


- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.
  > Labels

As métricas devem utilizar apenas labels com baixa cardinalidade, ou seja, valores que pertencem a um conjunto pequeno e controlado. Isso garante que o servidor de métricas (como Prometheus) continue eficiente mesmo durante picos de tráfego como os 5.000 TPS da Black Friday.

Labels Permitidas
Label	Motivoservice (checkout, pagamento, estoque)	Permite separar métricas por serviço.
endpoint	Identifica qual operação está sendo monitorada.
method (GET, POST, etc.)	Ajuda na análise por tipo de requisição.
status_code (200, 404, 500...)	Permite acompanhar erros e sucessos.
version	Diferencia Stable e Canary durante implantações.
environment (prod, staging)	Separa ambientes.
region	Permite identificar problemas localizados.
dependency (pagamento, estoque)	Mostra qual dependência está impactando a operação.
Labels Proibidas
Label	Problemacustomer_id	Gera milhões de séries diferentes e pode violar LGPD.
order_id	Cria uma série por pedido.
trace_id	Cada requisição cria uma nova série temporal.
session_id	Cardinalidade extremamente alta.
email	Além da alta cardinalidade, expõe dado pessoal.
cpf	Viola LGPD e gera cardinalidade ilimitada.
ip	Alta cardinalidade e possível dado pessoal.
timestamp	Cria uma série única para cada coleta.
Efeito dessas escolhas no servidor de métricas

Baixa cardinalidade (permitidas)

Menor consumo de memória.
Consultas rápidas.
Menor uso de CPU no servidor de métricas.
Retenção de dados mais eficiente.
Dashboards e alertas respondem rapidamente.

Alta cardinalidade (proibidas)

Explosão do número de séries temporais.
Alto consumo de memória e armazenamento.
Consultas lentas.
Aumento do tempo de coleta das métricas.
Possível indisponibilidade ou degradação do próprio sistema de observabilidade.

No cenário apresentado, o trace_id deve existir em logs e traces distribuídos, mas não em métricas, porque geraria uma série temporal para cada requisição. Já métricas devem permanecer agregadas e utilizar apenas labels controladas, enquanto a investigação detalhada de uma requisição específica deve ser feita através dos logs correlacionados e dos traces distribuídos.
---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  ( ) Tail Sampling  ( ) Combinação das duas
- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?
  > A Escolha

✅ ( X ) Combinação das duas

Utilizaria uma combinação de Head Sampling + Tail Sampling para equilibrar custo e capacidade de investigação.

Head Sampling: reduz o volume de traces logo na entrada do sistema, diminuindo custos de processamento e armazenamento.
Tail Sampling: analisa o resultado final do trace antes de decidir se ele será armazenado, garantindo a retenção dos traces mais importantes.
Justificativa

No cenário da Black Friday, com 5.000 TPS, armazenar 100% dos traces geraria um custo elevado de infraestrutura, armazenamento e consulta. Por outro lado, uma amostragem simples por Head Sampling poderia descartar justamente as requisições problemáticas.

Por isso, adotaria a seguinte política:

Regra de retenção

Guardar sempre (100%):

Traces com erro (4xx e 5xx relevantes).
Traces com timeout.
Traces cuja duração ultrapasse um limite definido (por exemplo, > 2 segundos).
Traces da versão Canary.
Traces que acionem retries ou circuit breakers.

Amostragem parcial para o restante:

Requisições saudáveis: armazenar entre 1% e 10%, ajustando conforme o volume de tráfego.
Como isso evita perder as requisições travadas?

O problema descrito no cenário não gera necessariamente HTTP 500. O cliente apenas fica esperando até ocorrer timeout.

Se fosse utilizado apenas Head Sampling, uma requisição lenta poderia ser descartada aleatoriamente antes de sabermos que ela apresentou problema.

Com Tail Sampling, a decisão é tomada após a conclusão do trace. Assim, quando uma requisição:

ultrapassar o tempo esperado;
terminar em timeout;
apresentar latência elevada (p95/p99);
ou falhar em alguma dependência (Pagamento ou Estoque);

o trace completo será preservado automaticamente.

Dessa forma, mesmo que apenas uma pequena parcela dos clientes seja afetada, os traces mais relevantes para investigação continuarão disponíveis, permitindo identificar exatamente onde ocorreu o travamento no fluxo Checkout → Pagamento → Estoque.

Justificativa para o cenário: como os dashboards permaneceram verdes e não houve aumento de HTTP 500, o principal sinal do incidente é a lentidão excessiva e os timeouts. A combinação de Head Sampling e Tail Sampling reduz custos sem comprometer a captura dos traces que realmente importam para o diagnóstico.
---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?
  > Estratégia de Testes
Qual a nossa estratégia para testar isso e garantir que funciona?

Antes da próxima Black Friday, a observabilidade deve ser validada por meio de testes de carga, testes de falha e simulações controladas de incidentes (Game Days/Chaos Engineering). O objetivo não é apenas verificar se o sistema funciona, mas provar que logs, métricas, traces e alertas conseguem detectar o problema rapidamente.

1. Teste de carga com tráfego semelhante ao real

Simular um cenário próximo ao esperado para a Black Friday, por exemplo:

5.000 TPS no endpoint de Checkout.
Tráfego distribuído entre versão Stable e Canary.
Chamadas reais para os serviços de Pagamento e Estoque em ambiente de teste.

Durante o teste, verificar se:

Os dashboards apresentam corretamente Rate, Errors e Duration.
Os percentis p95 e p99 refletem o comportamento do sistema.
Os traces são coletados e correlacionados ponta a ponta.
2. Injeção controlada do mesmo tipo de falha do incidente

Reproduzir deliberadamente o problema observado:

Adicionar latência artificial no serviço de Pagamento ou Estoque.
Fazer parte das requisições demorar vários segundos.
Simular timeouts sem retornar HTTP 500.

Objetivo:

Confirmar que a média continua aparentemente saudável.
Validar que os percentis (p95/p99) aumentam e evidenciam o problema.
Garantir que os traces lentos sejam capturados pela estratégia de Tail Sampling.
Confirmar que os logs apresentam o mesmo trace_id em todos os serviços.
3. Validação dos alertas

Executar testes específicos para garantir que os alertas disparem antes do impacto generalizado.

Alertas esperados:

p95 ou p99 acima do limite definido.
Taxa de timeout acima do SLO.
Crescimento anormal de requisições em andamento.
Saturação de pool de conexões.
Canary apresentando desempenho pior que a versão estável.

O teste só é considerado aprovado quando o alerta dispara dentro do tempo estabelecido pela operação.

4. Exercício de investigação

A equipe deve receber apenas um cenário semelhante ao incidente real:

"Clientes reclamam que o checkout fica carregando até expirar."

A partir dos dashboards, logs e traces, os engenheiros devem conseguir:

Identificar que o problema existe.
Localizar o serviço responsável.
Encontrar os traces afetados.
Verificar se o problema está associado à versão Canary.
Determinar a causa raiz.
Como provar que os painéis mostrariam este incidente?

A prova é reproduzir exatamente as características do incidente:

Sem aumento de HTTP 500.
CPU abaixo de 40%.
Uma pequena parcela das requisições sofrendo timeout.
Queda de conversão.
Latência média aparentemente saudável.

Os testes serão considerados bem-sucedidos se, mesmo nessas condições:

O painel mostrar aumento do p95 e p99.
Os traces lentos forem preservados.
O trace_id permitir seguir o fluxo Checkout → Pagamento → Estoque.
Os alertas forem acionados automaticamente.
A equipe conseguir identificar a causa sem acessar servidores ou utilizar SSH, respeitando a restrição das imagens Distroless.

Justificativa: Como o incidente real passou despercebido pelos dashboards tradicionais, a única forma de garantir que ele não se repita é reproduzir o mesmo padrão de falha em ambiente controlado e demonstrar que a nova estratégia de observabilidade consegue detectá-lo e diagnosticar sua origem rapidamente.


