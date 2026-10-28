# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) ( ) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > Persistência Poliglota / Híbrida
  Foi escolhido pq o Checkout exige ACID rigoroso para transações financeiras, o que torna o PostgreSQL indispensável. Porém, pelo Teorema CAP, não conseguimos ter consistência forte e alta disponibilidade sob partição ao mesmo tempo. Para dados de catálogo e sessões, podemos abrir mão de consistência imediata e adotar BASE (consistência eventual), garantindo escalabilidade e tolerância a falhas. Assim, a arquitetura poliglota atende ao requisito sem comprometer a integridade financeira.
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  
 Resposta: Key-Value (Redis/DynamoDB): ideal para carrinhos efêmeros e sessões de usuários, pois oferece leitura/escrita ultrarrápida e tolerância a partição.

Time-Series (InfluxDB/TimescaleDB): excelente para telemetria, logs e auditoria temporal, permitindo análise de métricas de performance e detecção de fraude em tempo real.

(Opcional extra) Search Engine (Elasticsearch): para busca textual de produtos, com relevância e filtros avançados.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 ( ) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  
  UUIDv7 / TSID

  Impacto no Armazenamento:

Sequence/Identity: não escala em sistemas distribuídos, pois exige coordenação central e gera gargalo.

UUIDv4: distribui bem, mas causa fragmentação severa em índices B-Tree, aumentando page splits e degradando inserções concorrentes.

UUIDv7/TSID: combina unicidade distribuída com ordenação temporal, reduzindo fragmentação e melhorando a performance de índices sob alta concorrência.

Por isso, UUIDv7 é a melhor escolha para o Checkout em Black Friday, garantindo escalabilidade e eficiência nos índices.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
   Resposta: Lock Otimista (@Version)
   Justificativa: 
Apenas o comprador interage com o status do seu pedido, logo a disputa por registro é mínima. O lock otimista evita bloqueios desnecessários, melhora a escalabilidade e só lança exceção em caso de concorrência rara. Isso reduz o custo operacional e mantém a experiência fluida.

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > reposta:  Lock Pessimista (SELECT FOR UPDATE)
  Justificativa:  
Centenas de clientes disputam o mesmo SKU promocional. Aqui não podemos correr o risco de overselling. O lock pessimista garante integridade, enfileirando transações e evitando que o saldo fique negativo. É mais custoso, mas indispensável para consistência em cenários de alta contenção.


### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > resposta: Lock Otimista:

Risco: excesso de OptimisticLockException pode frustrar o cliente em cenários de concorrência inesperada.

Mitigação: implementar retry logic transparente na aplicação, garantindo que o usuário não perceba falhas ocasionais.

Lock Pessimista:

Risco: ConnectionTimeoutException e deadlocks podem enfileirar threads e saturar o pool de conexões.

Mitigação:

Ajustar timeout de lock para liberar rapidamente conexões travadas.

Usar fila assíncrona para reservas de estoque em picos extremos, desacoplando a pressão direta no banco.

Monitorar com métricas de pool (HikariCP) e aplicar circuit breakers para evitar cascatas de falhas.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > reposta: Verificar métricas de conexão (HikariCP): analisar saturação do pool, tempo médio de checkout de conexões e número de threads bloqueadas.

Rodar EXPLAIN e EXPLAIN ANALYZE: inspecionar o plano de execução da query WHERE customer_id = ? AND order_status = 'processing'.

Confirmar se está ocorrendo Seq Scan (varredura completa) em vez de Index Scan.

Checar estatísticas com pg_stat_activity e pg_stat_statements: identificar queries mais lentas e frequência de execução.

Analisar índices existentes: verificar se há índice composto cobrindo (customer_id, order_status).

Monitorar I/O e buffers: usar ferramentas como Dalibo ou pgAdmin para visualizar gargalos de leitura/escrita.

Testar hipóteses: simular carga com índices novos em ambiente de staging para validar impacto antes de aplicar em produção.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > resposta: Posicionamento sobre o PR: ❌ Não aprovar a criação de índices individuais para todas as colunas.

Avaliação Crítica:

Impactos negativos:

Cada índice adicional aumenta o custo de INSERT e UPDATE, pois todos precisam ser atualizados.

Consumo extra de espaço em disco.

Complexidade maior para o otimizador escolher o plano de execução.

Solução correta:

Criar índice composto em (customer_id, order_status), pois reflete exatamente o padrão de consulta mais frequente.

Avaliar índices parciais (ex.: apenas para order_status = 'processing') se esse filtro for predominante.

Monitorar com EXPLAIN ANALYZE para confirmar que o índice está sendo utilizado.
