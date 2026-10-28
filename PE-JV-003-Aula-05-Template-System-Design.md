# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  > Construção vs Execução

Para o serviço de Checkout, adotaria uma estratégia de multi-stage build, separando completamente o ambiente de compilação do ambiente de execução.

Etapa de compilação (builder):

Imagem completa contendo:
JDK
Maven ou Gradle
Dependências de build
Ferramentas de testes
Ferramentas de análise estática
Executa:
Compilação
Testes unitários
Empacotamento do artefato (JAR)

Exemplo conceitual:

Construção vs Execução

FROM maven:3.9-eclipse-temurin-21 AS builder

COPY . .
RUN mvn clean package

Etapa de execução (runtime):

A imagem final deve conter apenas:

✅ Aplicação compilada (JAR)

✅ Runtime Java necessário

✅ Certificados TLS

✅ Configurações mínimas necessárias para execução

Não devem entrar:

❌ Maven

❌ Gradle

❌ Código-fonte

❌ Testes

❌ Ferramentas de compilação

❌ SSH

❌ Curl, wget, gcc e outras ferramentas administrativas sem necessidade operacional

❌ Credenciais embarcadas

Exemplo conceitual:

FROM gcr.io/distroless/java21

COPY --from=builder app.jar /app/app.jar

USER 10001

ENTRYPOINT ["java","-jar","/app/app.jar"]

Permissões do Processo

O container deve executar com um usuário não privilegiado:

USER 10001

ou

USER checkout

Nunca utilizar:

USER root

ustificativa:

Em um ambiente exposto à internet e processando pagamentos, uma vulnerabilidade na aplicação poderia permitir execução de código dentro do container. Executando como usuário não privilegiado, o impacto de uma eventual invasão é reduzido.

Benefícios:

Menor risco de escalonamento de privilégios.
Menor possibilidade de alteração do sistema de arquivos do container.
Aderência ao princípio do menor privilégio.
Escolha da Imagem Base

Para produção escolheria uma imagem mínima, como:

Distroless
Alpine endurecida
Chainguard

Preferencialmente:

gcr.io/distroless/java21

Segurança

A escolha de uma imagem mínima reduz:

Quantidade de bibliotecas instaladas.
Quantidade de binários disponíveis.
Quantidade de CVEs conhecidas.
Superfície total de ataque.

Isso é particularmente importante no cenário apresentado, onde uma falha durante uma Black Friday afeta um serviço crítico de Checkout operando a 5.000 TPS.

Trade-off: Segurança vs Diagnóstico
Aspecto	Imagem DistrolessSegurança	Excelente
CVEs	Muito poucas
Tamanho	Muito pequeno
Tempo de pull/deploy	Menor
Shell (bash/sh)	Não possui
Ferramentas de debug	Não possui
Diagnóstico emergencial	Mais difícil
Como lidar com o diagnóstico

Em vez de incluir ferramentas administrativas na imagem de produção:

Logs centralizados (ELK, Datadog, Grafana Loki).
Métricas via Prometheus.
Tracing distribuído (OpenTelemetry).
Containers efêmeros de debug quando necessário.
Ambiente de staging reproduzindo a mesma imagem.

Dessa forma mantemos a segurança da imagem produtiva sem sacrificar a capacidade de investigação de incidentes.

Decisão Arquitetural

Decisão: utilizar multi-stage build + imagem runtime Distroless + execução como usuário não root.

Benefícios para o cenário:

Redução da superfície de ataque do serviço de Checkout.
Menor quantidade de vulnerabilidades na imagem.
Imagens menores e mais rápidas de distribuir durante deploys.
Separação clara entre compilação e execução.
Maior conformidade com práticas de produção em ambientes de alta criticidade financeira.

Trade-off aceito: diagnósticos dentro do container tornam-se mais difíceis, exigindo observabilidade externa e mecanismos específicos de debug, porém o ganho de segurança e previsibilidade operacional justifica a escolha para um serviço crítico de Checkout em produção.


### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.

  > 1.2 Rastreabilidade, Identidade do Artefato e Parametrização
Da Homologação à Produção

Para evitar exatamente o problema ocorrido no incidente, a esteira deve seguir o princípio de Build Once, Deploy Many.

O artefato (JAR e imagem Docker) é compilado uma única vez a partir de um commit específico da branch principal. Após ser aprovado nos testes automatizados, a mesma imagem é promovida entre os ambientes de Dev, Staging e Produção.

Fluxo:

Commit
   ↓
Build
   ↓
Testes
   ↓
Imagem gerada
   ↓
Registry
   ↓
Deploy em Staging
   ↓
Validação
   ↓
Promoção da MESMA imagem
   ↓
Produção

Dessa forma:

✅ O binário testado em homologação é exatamente o mesmo executado em produção.

✅ Elimina diferenças causadas por recompilações posteriores.

✅ Garante reprodutibilidade.

No cenário do Checkout, o erro ocorreu porque a produção recompilou novamente a main, criando um artefato potencialmente diferente daquele validado anteriormente.

Configurações sem Recompilar

As configurações de ambiente não devem estar embutidas no código nem na imagem.

A imagem deve ser agnóstica ao ambiente:

checkout:v1.8.3

A diferença entre Dev, Staging e Produção é fornecida externamente por:

Variáveis de ambiente
ConfigMaps
Secrets
Secret Manager / Vault

Exemplo:

SPRING_PROFILES_ACTIVE=prod

DB_HOST=checkout-prod-db
DB_PORT=5432

KAFKA_BOOTSTRAP_SERVERS=kafka-prod:9092

A mesma imagem pode ser executada assim:

checkout:v1.8.3

em:

Dev
Staging
Produção

mudando apenas os parâmetros de execução.

Benefícios:

Não existe uma imagem para cada ambiente.
Reduz possibilidade de divergência.
Simplifica auditoria e troubleshooting.
Facilita promoção entre ambientes.
Identidade Imutável do Artefato

O incidente mostra um erro clássico:

checkout:latest

A tag foi sobrescrita e a versão anterior deixou de existir.

Para evitar isso, cada imagem deve possuir identificadores imutáveis:

checkout:1.8.3
checkout:2025.11.24
checkout:git-a4f91c2

Idealmente utilizar:

checkout:1.8.3
checkout:git-a4f91c2

onde:

1.8.3 representa a versão da aplicação.
a4f91c2 representa o SHA do commit.

Além disso, o registry mantém o digest único:

sha256:7ab3f...

O deploy deve referenciar preferencialmente o digest:

YAML

image: checkout@sha256:7ab3f...

pois o digest é imutável por definição.

Estratégia de Promoção

Em vez de:

latest -> staging
latest -> prod

a pipeline promove exatamente o mesmo artefato:

Plain Text
checkout:git-a4f91c2

ou

Plain Text
checkout@sha256:7ab3f...

de:

Plain Text
Staging
   ↓
Produção

sem qualquer rebuild.

Rollback Seguro

Em caso de falha:

Plain Text
Versão atual:
checkout:1.9.0

Versão anterior:
checkout:1.8.3

o rollback consiste apenas em alterar a referência da imagem:

Plain Text
checkout:1.8.3

ou

Plain Text
checkout@sha256:anterior

Não há risco de recuperar um artefato diferente porque:

Tags versionadas nunca são sobrescritas.
Digests são imutáveis.
O registry mantém histórico das versões publicadas.

Decisão Arquitetural

Decisão: adotar Build Once, Deploy Many, parametrização externa por variáveis/Secrets e versionamento imutável das imagens através de tags semânticas + SHA de commit + digest do registry.

Benefícios para o cenário:

O artefato executado em produção é exatamente o mesmo validado em homologação.
Elimina recompilações durante o deploy.
Remove dependência da tag latest.
Permite auditoria completa entre commit, pipeline, imagem e deploy.
Possibilita rollback rápido e confiável, sem o risco de recuperar uma imagem sobrescrita por engano.
---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** ( ) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  
  2.1 O Modelo de Rollout em Produção
Escolha

✅ Canary Release (Liberação Gradual por Percentual)

Justificativa

No cenário apresentado, o principal problema não foi apenas o erro na migração do banco, mas o fato de que uma alteração incompatível chegou rapidamente a um ambiente que processava 5.000 TPS. Para um sistema crítico de Checkout, o objetivo é reduzir o blast radius de qualquer falha.

Com Blue-Green, uma vez realizada a troca de tráfego, toda a carga passa a utilizar a nova versão. Caso exista um defeito não detectado nos testes, o impacto pode atingir imediatamente os 5.000 TPS.

Já com Canary Release, a nova versão é exposta gradualmente para uma pequena parcela dos usuários, permitindo detectar problemas antes que eles afetem toda a plataforma.

Evolução do Tráfego

Proposta de rollout:

Versão Atual (v1): 100%
Nova Versão (v2):   0%

Etapas:

Plain Text
5%   → validação
10%  → validação
25%  → validação
50%  → validação
100% → conclusão

Considerando 5.000 TPS:

Etapa	Tráfego5%	250 TPS
10%	500 TPS
25%	1.250 TPS
50%	2.500 TPS
100%	5.000 TPS

Dessa forma, um defeito é inicialmente exposto a apenas uma fração dos clientes.

Critérios para Avançar o Canary

O avanço não deve ocorrer apenas por tempo decorrido, mas por indicadores objetivos de saúde.

Monitoraria:

Taxa de Erro
Plain Text
HTTP 5xx
Timeouts
Exceções da aplicação

Exemplo:

Plain Text
Erro < 0,5%

Latência

Comparação entre:

Plain Text
P95
P99

da versão atual e da versão canário.

Exemplo:

Plain Text
Aumento máximo aceitável: 10%

Métricas de Negócio

Como se trata de Checkout:

Plain Text
Taxa de pagamento aprovado
Taxa de criação de pedido
Taxa de conclusão da Saga

Essas métricas são ainda mais importantes que CPU ou memória, pois detectam falhas funcionais.

Infraestrutura
Plain Text
Uso de CPU
Uso de memória
Pool de conexões PostgreSQL
Consumer lag do Kafka

Critérios de Rollback Automático

O rollback deve ser automático caso qualquer limite seja ultrapassado.

Exemplos:

Plain Text
Taxa de erro > 2%

ou

P99 > 2x da versão estável

ou

Plain Text
Queda da conversão de checkout > 5%

Nesse caso:

Plain Text
Canary → 0%
Versão estável → 100%

Como as imagens são imutáveis e versionadas por tag/digest (decisão da questão 1.2), o rollback é imediato e previsível.

Convivência de Duas Versões Simultaneamente

Este é o principal desafio do Canary.

Durante parte do rollout teremos:

Plain Text
v1 atendendo requisições
v2 atendendo requisições

simultaneamente.

Por isso, a arquitetura deve exigir compatibilidade entre versões.

Compatibilidade da API

Mudanças incompatíveis não podem ser implantadas diretamente.

Exemplo incorreto:

JSON
{
  "status_pagamento": "APROVADO"
}

Compatibilidade do Banco

O incidente demonstra exatamente o motivo.

A migração:

SQL
ALTER TABLE pedidos
RENAME COLUMN status_pagamento TO estado_saga;

é incompatível com rollout gradual.

Durante o Canary, versões antigas continuam executando.

Portanto, deve-se adotar o padrão Expand and Contract:

Expand

Adicionar nova estrutura sem remover a antiga.

SQL
ALTER TABLE pedidos
ADD COLUMN estado_saga VARCHAR(50);

Aplicações novas passam a gravar:

Plain Text
status_pagamento
estado_saga

Migração

Copiar os dados gradualmente.

Contract

Somente após:

Plain Text
100% do tráfego em v2
Rollback não ser mais necessário

remover a coluna antiga.

SQL
ALTER TABLE pedidos
DROP COLUMN status_pagamento;

Mitigação das Desvantagens do Canary
Desvantagem	MitigaçãoDuas versões simultâneas	Compatibilidade de API e banco
Rollout mais complexo	Automação via Service Mesh/Kubernetes
Observabilidade mais difícil	Métricas separadas por versão
Necessidade de múltiplas validações	Gates automáticos baseados em SLOs
Migrações incompatíveis	Estratégia Expand-and-Contract
Decisão Arquitetural

Escolho Canary Release, pois ele reduz significativamente o impacto potencial de falhas em um serviço de Checkout operando a 5.000 TPS, permite validação progressiva em produção, suporta rollback rápido e, combinado com migrações backward-compatible (Expand-and-Contract), evita a repetição do incidente em que versões antigas e novas ficaram incompatíveis durante o deploy.


### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  
  2.2 Evolução de Schema Sem Downtime e Plano de Rollback

Para permitir um Canary Release com coexistência entre v1 e v2, não podemos utilizar:

SQL
ALTER TABLE pedidos
RENAME COLUMN status_pagamento TO estado_saga;

Essa abordagem foi justamente a causa do incidente, pois a versão antiga continuou procurando a coluna status_pagamento enquanto o banco já possuía apenas estado_saga.

A estratégia correta é utilizar o padrão Expand → Migrate → Contract, garantindo compatibilidade entre versões durante todo o rollout.

Etapa 1 - Expandir o Schema (Compatível com v1)

Adicionar a nova coluna sem remover a antiga:

SQL
ALTER TABLE pedidos
ADD COLUMN estado_saga VARCHAR(50);

Estado da tabela:

status_pagamento	estado_sagaAPROVADO	NULL

Resultado:

A versão v1 continua funcionando normalmente.
Nenhuma consulta SQL existente é quebrada.
Rollback continua possível.
Etapa 2 - Backfill dos Dados Existentes

Copiar os valores históricos para a nova coluna:

SQL
UPDATE pedidos
SET estado_saga = status_pagamento
WHERE estado_saga IS NULL;

Após a migração:

status_pagamento	estado_sagaAPROVADO	APROVADO
PENDENTE	PENDENTE

Resultado:

Dados antigos passam a existir nas duas colunas.
Nenhuma versão da aplicação é impactada.
Etapa 3 - Deploy da Versão v2 (Dual Write)

A nova versão deve gravar em ambas as colunas.

Exemplo:

java
pedido.setStatusPagamento(status);
pedido.setEstadoSaga(status);

Toda atualização executa:

SQL
UPDATE pedidos
SET status_pagamento = ?,
    estado_saga = ?

Resultado:

v1 continua lendo e escrevendo status_pagamento.
v2 escreve em ambas.
Não há inconsistência durante o rollout gradual.

Etapa 4 - Canary da v2

Iniciar o rollout gradual:

Plain Text
v1 = 95%
v2 = 5%

Depois:

Plain Text
v1 = 90%
v2 = 10%

Até:

Plain Text
v1 = 0%
v2 = 100%

Durante esse período:

v1 utiliza status_pagamento.
v2 utiliza ambos os campos.
Nenhum erro de SQL ocorre.

Etapa 5 - Migrar Leituras para a Nova Coluna

Quando a v2 já estiver estável, alterar a aplicação para que a leitura passe a utilizar:

SQL
estado_saga

Mantendo ainda o dual write.

Exemplo:

Java
String estado = pedido.getEstadoSaga();

Resultado:

Toda a lógica passa a depender da nova coluna.
A antiga permanece apenas para compatibilidade e rollback.
Etapa 6 - Validar Ausência de Tráfego da v1

Antes de remover a coluna antiga, confirmar:

✅ 100% das instâncias executando v2

✅ Nenhum pod da v1 em produção

✅ Janela de rollback encerrada

✅ Métricas e logs sem acessos ao campo antigo

Etapa 7 - Encerrar o Dual Write

Após a confirmação:

Java
pedido.setEstadoSaga(status);

Removendo:

Java
pedido.setStatusPagamento(status);

Agora somente a nova coluna é atualizada.

Etapa 8 - Contract (Remoção do Campo Legado)

Somente depois de concluídas todas as etapas anteriores:

SQL
ALTER TABLE pedidos
DROP COLUMN status_pagamento;

Estado final:

estado_sagaAPROVADO
PENDENTE
Plano de Rollback
Rollback durante as Etapas 1 a 6

É simples e imediato.

Basta redirecionar o tráfego para a versão anterior:

Plain Text
v2 → 0%
v1 → 100%

Como a coluna:

SQL
status_pagamento
 

ainda existe e continua sendo atualizada, a v1 funciona normalmente.

Nenhuma migração reversa é necessária.

Rollback após a Etapa 8

Após executar:

SQL
DROP COLUMN status_pagamento;

o rollback para a v1 deixa de ser possível.

Por isso a remoção da coluna antiga deve ocorrer em uma release posterior, quando:

A nova versão estiver comprovadamente estável.
Não houver necessidade operacional de retornar para a v1.
O período de observação do Canary já tiver sido concluído.
Sequência Resumida
Plain Text
1. ADD COLUMN estado_saga
2. Backfill dos dados antigos
3. Deploy v2 com dual write
4. Canary Release
5. v2 passa a ler estado_saga
6. Validar 100% do tráfego em v2
7. Remover dual write
8. DROP COLUMN status_pagamento

Essa estratégia garante zero downtime, permite que v1 e v2 coexistam durante todo o rollout, suporta rollback imediato e evita exatamente o cenário do incidente em que a renomeação direta da coluna tornou as versões incompatíveis.

- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  
  Garantia de Rollback Imediato

Se a v2 apresentar degradação grave de performance durante a Black Friday quando estiver recebendo 20% dos 5.000 TPS, o rollback deve ser exclusivamente uma operação de roteamento de tráfego, sem qualquer alteração emergencial no banco de dados.

Procedimento Exato de Rollback

Estado no momento da falha:

Plain Text
v1 = 80% do tráfego
v2 = 20% do tráfego

Ao detectar que a v2 ultrapassou os limites definidos de erro, latência ou métricas de negócio:

Plain Text
Taxa de erro > limite
ou
P99 > limite
ou
Queda na conversão do checkout

executamos:

Plain Text
1. Interromper imediatamente o avanço do Canary.
2. Redirecionar o tráfego da v2 para 0%.
3. Redirecionar o tráfego da v1 para 100%.
4. Encerrar/remover os pods da v2.
5. Manter o banco exatamente como está.

Resultado:

Plain Text
v1 = 100%
v2 = 0%

O retorno é praticamente imediato porque não depende de:

Restore de banco.
Reversão de migração.
Rebuild de imagem.
Reimplantação manual de código.

A versão v1 já está em produção recebendo 80% da carga e apenas volta a assumir 100%.

Por Que a v1 Não Quebra?

Porque a migração foi projetada usando o padrão Expand and Contract.

Em vez de fazer:

SQL
ALTER TABLE pedidos
RENAME COLUMN status_pagamento TO estado_saga;

foi executado:

SQL
ALTER TABLE pedidos
ADD COLUMN estado_saga;

Mantendo simultaneamente:

Plain Text
status_pagamento
estado_saga

na tabela.

Assim, durante todo o Canary:

Plain Text
v1 continua utilizando status_pagamento
v2 utiliza estado_saga

As duas versões enxergam as estruturas que esperam encontrar.

Por Que os Dados Gravados Pela v2 Não São Perdidos?

A v2 foi implementada com dual write.

Sempre que um pedido é atualizado:

SQL
UPDATE pedidos
SET
status_pagamento = 'APROVADO',
estado_saga = 'APROVADO'

Ou seja, o mesmo valor é persistido nas duas colunas.

Exemplo:

id_pedido	status_pagamento	estado_saga101	APROVADO	APROVADO

Mesmo que o pedido tenha sido criado ou alterado pela v2 segundos antes do rollback, a v1 continuará enxergando os dados corretamente através da coluna:

Plain Text
status_pagamento
O Que Teria Acontecido no Incidente Original?

No incidente descrito, o Flyway executou:

SQL
RENAME COLUMN status_pagamento TO estado_saga;
 

Então a situação ficou:

estado_sagaAPROVADO

Quando a v1 tentou executar:

SQL
SELECT status_pagamento

ocorreu:

Plain Text
column "status_pagamento" does not exist

Tornando o rollback impossível.

Na estratégia proposta isso não acontece porque a coluna antiga só é removida muito depois, quando:

100% do tráfego já está na v2.
O período de observação foi concluído.
Não existe mais possibilidade de retorno para a v1.
Conclusão

O rollback é imediato porque consiste apenas em redirecionar o tráfego de 20% → 0% para a v2 e de 80% → 100% para a v1, sem tocar no banco de dados. O uso do padrão Expand and Contract, combinado com dual write, garante que tanto status_pagamento quanto estado_saga coexistam durante o rollout. Dessa forma, a v1 continua funcionando normalmente e consegue ler todos os dados gravados pela v2, eliminando o risco de erro de SQL ou perda de informação durante a reversão.
---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  
  3.1 Gates de Segurança e Guardrails no Processo de Entrega

Considerando que o incidente ocorreu por uma combinação de rebuild em produção, uso da tag latest e migração incompatível de banco, a esteira de CI/CD deve possuir múltiplos guardrails automáticos que impeçam uma mudança perigosa de chegar à produção.

1. Pull Request (Pré-Merge)

Antes da aprovação do PR:

Revisão Obrigatória de Código
Aprovação mínima de 2 revisores.
Proibição de auto-merge.
Branch Protection na main.

Objetivo:

Detectar mudanças perigosas.
Revisar alterações de banco e contratos de API.
Testes Automatizados

Execução obrigatória de:

Plain Text
Testes unitários
Testes de integração
Testes de contrato

O merge é bloqueado caso qualquer teste falhe.

Análise Estática (SAST)

Ferramentas como:

Plain Text
SonarQube
Semgrep
Checkmarx

Verificam:

SQL Injection.
Vulnerabilidades conhecidas.
Código inseguro.
Débito técnico crítico.

O pipeline falha caso existam achados de alta severidade.

Verificação de Cobertura

Exemplo:

Plain Text
Cobertura mínima de 80%

Mudanças sem cobertura adequada não são promovidas.

2. Build

Após o merge aprovado:

Build Único (Build Once)

A aplicação é compilada uma única vez.

Plain Text
Commit
↓
Build
↓
Imagem Docker

Nenhuma recompilação é permitida nos ambientes seguintes.

Isso elimina o problema ocorrido no incidente.

Assinatura do Artefato

A imagem gerada deve ser assinada digitalmente.

Exemplo:

Plain Text
Cosign
Notary

Garantia:

A imagem não foi alterada após o build.
A origem do artefato é verificável.
3. Segurança da Imagem

Antes de publicar no registry:

Scan de Vulnerabilidades

Ferramentas:

Plain Text
Trivy
Grype
Clair

Verificações:

CVEs do sistema operacional.
Bibliotecas vulneráveis.
Dependências inseguras.

Regra:

Plain Text
CVSS Crítico → Bloqueia o pipeline
CVSS Alto → Exige aprovação formal
Verificação de Secrets

Ferramentas:

Plain Text
GitLeaks
TruffleHog

Detectam:

Plain Text
Senhas
Tokens
Chaves AWS
Certificados

Nenhum segredo pode seguir para produção.

4. Publicação no Registry
Tags Imutáveis

Proibido:

Plain Text
checkout:latest

Obrigatório:

Plain Text
checkout:1.8.3
checkout:git-a4f91c2

ou

Plain Text
sha256:digest

Guardrail:

Plain Text
Tags não podem ser sobrescritas.

Isso garante rollback confiável.

5. Validação em Staging

Antes da promoção para produção:

Testes de Integração End-to-End

Validação completa do fluxo:

Plain Text
Checkout
↓
Pagamento
↓
Saga
↓
Pedido
Smoke Tests

Validação básica:

Plain Text
Health Check
Criação de pedido
Consulta de pedido
Pagamento

Caso falhe:

Plain Text
Promoção bloqueada
Testes de Performance

Como o sistema suporta 5.000 TPS, cada release deve ser testada sob carga próxima ao cenário real.

Validação de:

Plain Text
P95
P99
Uso de CPU
Uso de Memória
Pool de Conexões
6. Guardrails Específicos para Banco de Dados

Este é o ponto mais crítico do incidente.

Validação Automática das Migrações Flyway

Bloquear migrações destrutivas como:

SQL
DROP COLUMN
DROP TABLE
RENAME COLUMN

quando houver rollout gradual.

Toda alteração deve seguir:

Plain Text
Expand
→ Migrate
→ Contract
Compatibility Check

O pipeline deve validar:

Plain Text
Banco novo + aplicação antiga
Banco novo + aplicação nova

Objetivo:

Garantir que v1 e v2 consigam coexistir durante o Canary.

7. Promoção para Produção
Aprovação Manual (Change Gate)

Para um serviço crítico de Checkout:

Plain Text
Staging aprovado
↓
Aprovação do responsável
↓
Produção

Não realizar deploy totalmente automático em períodos críticos como Black Friday.

Deploy Canary Automatizado

Progressão:

Plain Text
5%
10%
25%
50%
100%

Com observabilidade em cada etapa.

8. Guardrails Pós-Deploy

A promoção do Canary só avança se os SLOs forem atendidos.

Monitorar:

Técnicos
Plain Text
HTTP 5xx
Timeouts
Latência P95
Latência P99
CPU
Memória
 
Negócio
Plain Text
Taxa de aprovação de pagamento
Taxa de conclusão da Saga
Conversão do Checkout
9. Rollback Automático

Caso qualquer limite seja excedido:

Exemplo:

Plain Text
Erro > 2%
 

ou

Plain Text
P99 > 2x baseline

ou

Plain Text
Conversão -5%

A plataforma executa:

Plain Text
Canary → 0%
Versão estável → 100%

automaticamente.

Fluxo Final de Guardrails
Plain Text
Pull Request
↓
Code Review
↓
Testes Unitários
↓
Testes de Integração
↓
SAST
↓
Scan de Secrets
↓
Build Único
↓
Assinatura da Imagem
↓
Scan de Vulnerabilidades
↓
Publicação com Tag Imutável
↓
Deploy em Staging
↓
Smoke Tests
↓
Testes E2E
↓
Validação de Migração Flyway
↓
Aprovação de Mudança
↓
Canary Release
↓
Monitoramento de SLOs
↓
Promoção ou Rollback Automático
Decisão Arquitetural

A esteira deve implementar guardrails em cada etapa do ciclo de entrega, impedindo que código inseguro, imagens vulneráveis, migrações incompatíveis ou artefatos não rastreáveis cheguem à produção. Dessa forma, evitamos exatamente os fatores que causaram o incidente: uso de latest, recompilação em produção, ausência de validação de compatibilidade e impossibilidade de rollback seguro.

- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  
  3.1 Governança e Controle em Produção

O objetivo dos guardrails de governança é impedir que uma situação semelhante ao incidente aconteça novamente, onde houve rebuild em produção, sobrescrita da tag latest e alteração irreversível do banco, dificultando o rollback. Para isso, a produção deve ser tratada como um ambiente imutável, auditável e reproduzível.

Guardrails de Processo
1. Git como Fonte Única da Verdade (GitOps)

Nenhuma alteração deve ser realizada manualmente nos servidores ou no cluster.

Todo ajuste deve ocorrer através de:

Plain Text
Pull Request
→ Aprovação
→ Merge
→ Pipeline
→ Deploy

Benefícios:

Todas as mudanças ficam registradas.
Existe histórico de autoria.
É possível saber quem aprovou e quando.
Facilita auditorias e análises pós-incidente.
2. Branch Protection

A branch principal deve possuir:

Plain Text
Pull Request obrigatório
Aprovação por pares
Pipeline verde obrigatório
Proibição de push direto

Dessa forma, ninguém consegue enviar código diretamente para produção.

3. Separação de Funções

Nenhum desenvolvedor deve possuir permissão para:

Plain Text
Alterar código
Aprovar o próprio código
Realizar deploy manual

Princípio:

Plain Text
Four Eyes Principle

Toda alteração crítica passa por pelo menos dois responsáveis.

4. Aprovação Formal para Produção

Após aprovação técnica:

Plain Text
Staging aprovado
↓
Mudança autorizada
↓
Deploy em Produção

Especialmente durante eventos de alto risco como Black Friday.

Guardrails de Arquitetura
1. Produção Imutável

Um dos erros do cenário foi recompilar a aplicação durante o deploy.

A regra deve ser:

Plain Text
Build Once
Deploy Many

A produção recebe apenas imagens previamente criadas e testadas.

Proibido:

Plain Text
Compilar em produção

Permitido:

Plain Text
Promover imagem já validada
2. Tags Imutáveis

Proibido:

Plain Text
checkout:latest

Obrigatório:

Plain Text
checkout:1.8.3
checkout:git-a4f91c2
checkout@sha256:...

Além disso:

Plain Text
Nenhuma tag pode ser sobrescrita.

Isso garante rastreabilidade completa e rollback confiável.

3. Bloqueio de Acesso Direto ao Ambiente

Operadores não devem executar:

Shell
kubectl edit
kubectl patch
kubectl apply manual

em produção rotineiramente.

As alterações devem passar pelo pipeline.

Isso evita:

Drift de configuração.
Mudanças não documentadas.
Ambientes diferentes do esperado.
4. Flyway Controlado pela Pipeline

Migrações de banco não podem ser executadas manualmente.

Devem passar por:

Plain Text
Validação
↓
Teste
↓
Aprovação
↓
Execução Automatizada

Além disso, regras automáticas devem bloquear comandos destrutivos incompatíveis com Canary:

SQL
DROP COLUMN
DROP TABLE
RENAME COLUMN

sem análise prévia.

Como Equilibrar Segurança e Agilidade em Incidentes

Um processo excessivamente burocrático pode atrasar a resposta a um incidente crítico. Por isso, é necessário possuir um mecanismo de exceção controlada.

1. Procedimento de Emergência (Break Glass)

Durante um incidente grave:

Plain Text
Indisponibilidade do Checkout
Queda de receita
Falha de pagamento

alguns operadores autorizados podem executar ações emergenciais.

Exemplos:

Plain Text
Rollback
Escalonamento horizontal
Bloqueio de uma feature

Mas toda ação deve ser:

Plain Text
Autenticada
Autorizada
Auditada
Registrada
2. Rollback Não Deve Precisar de Aprovação Complexa

Para um serviço crítico de Checkout:

✅ Deploy pode exigir aprovações.

✅ Rollback deve ser rápido.

Exemplo:

Plain Text
Canary detecta anomalia
↓
Rollback automático
↓
Tráfego retorna para v1

Sem esperar reunião ou aprovação manual.

3. Feature Flags

Mudanças de comportamento podem ser desligadas sem novo deploy.

Exemplo:

Plain Text
Nova lógica da Saga

Pode ser:

Plain Text
Ativada
Desativada

via configuração controlada.

Benefícios:

Resposta rápida.
Menor risco operacional.
Auditoria preservada.
4. Auditoria Centralizada

Toda ação deve gerar evidências:

Plain Text
Quem executou
O que foi alterado
Quando ocorreu
Qual ticket justificou
Qual versão foi implantada
Qual versão foi revertida

Isso permite reconstruir completamente a linha do tempo de um incidente.

Decisão Arquitetural

Adotarei uma combinação de GitOps, artefatos imutáveis, aprovação controlada, restrição a alterações manuais e auditoria completa, tornando impossível que mudanças não rastreadas cheguem à produção. Ao mesmo tempo, para incidentes críticos, utilizarei rollback automatizado, feature flags e procedimentos de emergência auditados (break glass), garantindo rapidez operacional sem abrir mão da governança e da rastreabilidade exigidas para um sistema de Checkout que processa 5.000 TPS durante a Black Friday.

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  
  3.2 Estratégia de Testes da Imagem e da Migração
Validação Pré-Tráfego

Concordo com a premissa. Passar em testes unitários apenas prova que partes isoladas do código funcionam. No incidente apresentado, o problema ocorreu na interação entre:

A imagem efetivamente implantada.
A migração Flyway.
O PostgreSQL compartilhado.
A coexistência entre v1 e v2.

Portanto, antes de liberar qualquer tráfego real, a pipeline deve validar o artefato final e a compatibilidade da evolução do banco.

Plano de Testes na Pipeline
1. Testes Unitários

Ainda são executados durante o build:

Plain Text
Compile
→ Testes Unitários
→ Testes de Integração da Aplicação

Objetivo:

Validar regras de negócio.
Detectar regressões rápidas.

Contudo, a promoção não depende apenas desta etapa.

2. Construção da Imagem Final

Após o build:

Plain Text
JAR
↓
Docker Build
↓
Imagem Final de Produção

A partir daqui, todos os testes passam a utilizar a imagem real que será promovida para produção.

Isso garante que estamos validando exatamente o mesmo artefato que será executado no Canary.

3. Container Startup Test

A pipeline deve iniciar a imagem em um ambiente temporário.

Exemplo:

Plain Text
docker run checkout:1.9.0

Validando:

Container sobe corretamente.
ENTRYPOINT funciona.
Usuário não-root consegue iniciar a aplicação.
Health Check retorna sucesso.

Exemplo:

HTTP
GET /actuator/health

Resultado esperado:

JSON
{
"status": "UP"
}

Falha nesta etapa bloqueia a promoção.

4. Ambiente Efêmero para Integração

A pipeline cria automaticamente:

Plain Text
Checkout v1
Checkout v2
PostgreSQL limpo
Kafka (ou equivalente)

Utilizando Docker Compose ou ambiente Kubernetes temporário.

Objetivo:

Simular a arquitetura real antes de qualquer deploy.

5. Teste da Migração Flyway

Executar todas as migrations em um PostgreSQL novo.

Fluxo:

Plain Text
Banco vazio
↓
Flyway migrate
↓
Validação do schema

Verificar:

Migrações executam sem erro.
Estrutura final está correta.
Índices foram criados.
Constraints permanecem válidas.
6. Teste de Retrocompatibilidade (Principal Teste)

Este é o guardrail que teria evitado o incidente.

Cenário A

Aplicação antiga sobre banco novo.

Fluxo:

Plain Text
Banco atualizado pelo Flyway
↓
Subir v1
↓
Executar testes funcionais

Validar:

Plain Text
Criar pedido
Consultar pedido
Atualizar pagamento
Executar Saga

Se a v1 apresentar erro:

Plain Text
column status_pagamento does not exist

a pipeline falha imediatamente.

7. Teste de Forward Compatibility
Cenário B

Aplicação nova sobre banco atualizado.

Fluxo:

Plain Text
Banco atualizado
↓
Subir v2
↓
Executar suíte E2E

Validar:

Plain Text
Checkout
Pagamento
Saga
Persistência
Consultas
8. Teste de Coexistência v1 + v2

Este é o cenário mais próximo da produção.

Ambiente:

Plain Text
PostgreSQL
v1
v2

Executando simultaneamente.

Carga de teste:

Plain Text
50% requisições para v1
50% requisições para v2

Validações:

Ambas respondem corretamente.
Não existem erros SQL.
Não existem inconsistências de dados.
Ambas conseguem ler registros produzidos pela outra versão.
9. Teste de Dual Write

Considerando:

Plain Text
status_pagamento
estado_saga

A pipeline deve validar automaticamente:

Escrita pela v2
SQL
INSERT/UPDATE

Verificar:

Plain Text
status_pagamento = APROVADO
estado_saga = APROVADO

As duas colunas devem permanecer sincronizadas.

10. Teste de Rollback

Este é um dos testes mais importantes para o cenário.

Fluxo:

Plain Text
1. Subir v1
2. Aplicar migração
3. Subir v2
4. Gerar dados
5. Desligar v2
6. Voltar para v1
 

Validações:

Plain Text
v1 continua funcionando
v1 consegue ler pedidos gravados pela v2
nenhum dado foi perdido
nenhum erro SQL ocorre
 

Se o rollback falhar, a release é bloqueada.

11. Smoke Test da Imagem Promovida

Antes de receber tráfego real:

Plain Text
Deploy em Staging
↓
Smoke Tests

Validações:

Health Check.
Criação de pedido.
Autorização de pagamento.
Criação da Saga.
Consulta do pedido.
12. Teste de Performance Básico

Executar carga controlada:

Plain Text
500 TPS
1000 TPS

ou volume compatível com o ambiente de homologação.

Avaliar:

Plain Text
P95
P99
CPU
Memória
Conexões PostgreSQL
 

Comparando com a versão anterior.

Fluxo Final da Pipeline
Plain Text
PR
↓
Testes Unitários
↓
Build
↓
Criação da Imagem
↓
Container Startup Test
↓
Flyway Migration Test
↓
Banco Novo + v1
↓
Banco Novo + v2
↓
Coexistência v1 + v2
↓
Teste de Dual Write
↓
Teste de Rollback
↓
Smoke Test
↓
Teste de Performance
↓
Staging
↓
Canary Release
Decisão Arquitetural

A validação não deve parar no código-fonte. A pipeline deve testar a imagem final exatamente como será executada em produção, validar a migração do PostgreSQL e, principalmente, provar automaticamente que v1 e v2 conseguem coexistir e que um rollback continua funcional após a evolução do schema. Esse conjunto de testes teria evitado o incidente descrito, pois detectaria a incompatibilidade causada pela renomeação direta da coluna antes que qualquer tráfego real fosse impactado.
