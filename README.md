# Estudo Aprofundado: AWS Step Functions

Este documento detalha meu estudo sobre o AWS Step Functions, com foco em conceitos-chave, padrões de arquitetura e melhores práticas para a criação de workflows serverless robustos.

---

## 1. Conhecendo o AWS Step Functions

### Anotações Detalhadas

O AWS Step Functions é um serviço de orquestração **serverless** que utiliza **Máquinas de Estado Finito (Finite State Machines)** para coordenar componentes de aplicações distribuídas. O "estado" da sua aplicação é gerenciado pelo serviço, em vez de estar implícito no seu código.

* **Linguagem:** As máquinas de estado são definidas em **Amazon States Language (ASL)**, um formato declarativo JSON. Esta definição é o "plano" do seu workflow.
* **Conceitos-Chave:**
    * **State Machine:** A definição do workflow (o template JSON).
    * **Execution:** Uma instância em execução da sua máquina de estado. Cada execução recebe um JSON de entrada e produz uma saída.
    * **State (Estado):** Um passo no seu workflow. Existem diversos tipos, como `Task`, `Choice`, `Parallel`, `Map` e `Wait`.
    * **Transition (Transição):** A ligação entre um estado e o próximo.

#### Tipos de Workflow: Standard vs. Express

Esta é a decisão mais fundamental ao criar uma Step Function:

| Característica | **Standard (Padrão)** | **Express (Expresso)** |
| :--- | :--- | :--- |
| **Duração Máx.** | 1 ano | 5 minutos |
| **Modelo de Execução**| *Exactly-once* (Exatamente uma vez) | *At-least-once* (Pelo menos uma vez) |
| **Preço** | Por transição de estado | Por nº de execuções + Duração (GB/s) |
| **Audiotoria** | Histórico visual completo e detalhado | Apenas logs (via CloudWatch) |
| **Casos de Uso** | Processos de negócio, ETLs longos, orquestração de microsserviços com SAGA. | Ingestão de dados IoT, processamento de streaming, "cola" de microsserviços de alta velocidade. |

### Insights e Melhores Práticas

* **Idempotência é Chave:** Mesmo que workflows *Standard* garantam a execução *exactly-once* da transição, a sua **tarefa** (ex: a função Lambda) pode ser invocada mais de uma vez em cenários de retry. Sempre projete suas funções Lambda para serem **idempotentes** (executar várias vezes com o mesmo input deve gerar o mesmo resultado).
* **Estado vs. Código:** O principal valor é mover a **lógica de fluxo, espera e tratamento de erros** do seu código (imperativo) para a definição da máquina de estado (declarativa). Suas Lambdas devem focar apenas na regra de negócio.

### Fonte Principal

* **[AWS Step Functions Developer Guide](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)**

---

## 2. Benefícios e Projeto Modelo

### Benefícios Detalhados

* **Resiliência Integrada:** Configuração declarativa de `Retry` (retentativas com backoff exponencial) e `Catch` (captura de falhas) para *qualquer* estado `Task`.
* **Rastreabilidade:** O console visual de execuções (para Workflows Standard) é uma ferramenta poderosa de debug, mostrando o JSON de entrada e saída de cada passo.
* **Orquestração Paralela:**
    * `Parallel`: Executa ramos *diferentes* de lógica ao mesmo tempo e espera todos terminarem.
    * `Map`: Executa o *mesmo* passo para cada item de um array de entrada (paralelismo dinâmico). Essencial para processamento em lote.
* **Integrações de Serviço:** O Step Functions pode invocar mais de 200 serviços da AWS diretamente, sem a necessidade de uma Lambda "cola". Existem dois tipos:
    1.  **Integrações Otimizadas:** (Ex: Lambda, SQS, SNS, DynamoDB) - Mais simples de configurar.
    2.  **Integrações AWS SDK:** Permite chamar *qualquer* ação de API da AWS (ex: `ec2:RunInstances` ou `s3:GetObject`).

### Padrão de Arquitetura: O Padrão SAGA

Este é o caso de uso mais importante para Step Functions em microsserviços. SAGA é um padrão para gerenciar falhas em transações distribuídas (onde um "commit" em múltiplos bancos de dados não é possível).

* **Conceito:** Se um passo da transação falha, o SAGA executa ações **compensatórias** para reverter o trabalho já feito.
* **Exemplo (Pedido):**
    1.  `(Task)` Processar Pagamento.
    2.  `(Task)` Reservar Inventário.
    3.  `(Task)` Agendar Envio.
* **O Problema:** O que acontece se o "Agendar Envio" falhar, mas o pagamento e o inventário já foram processados?
* **Solução com Step Functions:**
    * Use um bloco `Catch` no estado "Agendar Envio".
    * Se falhar, o `Catch` redireciona o fluxo para os estados de compensação:
        1.  `(Task)` Reverter Reserva de Inventário.
        2.  `(Task)` Estornar Pagamento.
        3.  `(Fail)` Marcar a execução como falha.

### Fonte Principal

* **[Padrões de Orquestração SAGA com AWS Step Functions](https://aws.amazon.com/pt/blogs/compute/implementing-the-saga-pattern-with-aws-step-functions/)**
* **[Integrações de Serviço do Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-service-integrations.html)**

---

## 3. Realizando Validações (Estado `Choice`)

### Anotações Detalhadas

O estado `Choice` adiciona lógica condicional (branching) a um workflow.

* **Como Funciona:** Ele inspeciona o JSON de estado usando "Regras de Escolha" (`Choice Rules`).
* **Operadores de Comparação:** Suporta um conjunto rico de operadores para avaliar valores no JSON de estado:
    * `StringEquals`, `StringMatches` (suporta wildcards `*`)
    * `NumericEquals`, `NumericGreaterThan`, `NumericLessThan`
    * `BooleanEquals`
    * `TimestampEquals`, `TimestampGreaterThan`
    * `IsNull`, `IsPresent`
    * Operadores lógicos: `And`, `Or`, `Not`.
* **Estrutura:**
    * `Choices` (Array): Uma lista de regras. A *primeira* regra que for `true` tem seu `Next` (próximo estado) executado.
    * `Default` (String): O estado para onde o fluxo segue se *nenhuma* das regras em `Choices` for `true`.

### Insights e Melhores Práticas

* **Anti-Padrão: "Lambda Validadora"**: **Evite** criar uma função Lambda que existe *apenas* para fazer um `if-then-else` e retornar uma string. Isso adiciona custo, latência e complexidade. Sempre prefira usar um estado `Choice` para lógica de fluxo.
* **Motor de Regras Simples:** O `Choice` pode ser usado como um "Motor de Regras de Negócio" (Business Rules Engine - BRE) leve, permitindo que a lógica de negócio seja declarativa, auditável e modificável sem a necessidade de deploy de novo código Lambda.

### Fonte Principal

* **[Amazon States Language (ASL) - Estado `Choice`](https://states-language.com/spec.html#choice-state)**
* **[AWS Docs - Estado `Choice`](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-choice-state.html)**

---

## 4. Criando e Executando Lambda (Estado `Task`)

### Anotações Detalhadas

O estado `Task` é a principal forma de executar trabalho. A integração com Lambda é a mais comum.

* **Invocação:** O campo `Resource` define o ARN da Lambda.
* **Padrões de Invocação:**
    * **`RequestResponse` (Padrão):** O Step Function invoca a Lambda e *espera* (sincronamente) pela sua conclusão (ou falha) antes de ir para o próximo estado.
    * **`Event` (Assíncrono):** O Step Function invoca a Lambda ("fire-and-forget") e segue imediatamente para o próximo estado.
    * **`.waitForTaskToken` (Callback):** O Step Function invoca a Lambda, passa um `taskToken` e *pausa* a execução. O workflow só continua quando a Lambda (ou outro processo) envia esse token de volta via API `SendTaskSuccess` ou `SendTaskFailure`. Ideal para **processos manuais de aprovação** ou jobs assíncronos que demoram mais que o timeout da Lambda.

#### Manipulação de Dados (Input/Output)

Este é o aspecto mais complexo. O Step Functions usa filtros JSON (baseados em JSONPath) para manipular o JSON de estado:

* **`InputPath`:** Seleciona uma *parte* do JSON de estado para enviar como input para a Lambda (ex: `$.customerDetails`).
* **`Parameters`:** **Constrói** um novo objeto JSON para enviar como input, usando partes do JSON de estado (ex: `{"id.$": "$.customerId", "name.$": "$.customerName"}`). É mais flexível que o `InputPath`.
* **`ResultPath`:** Pega o *resultado* (output) da Lambda e o *insere* de volta no JSON de estado. (Ex: `ResultPath: "$.paymentResult"`). Se for `null`, o resultado é descartado.
* **`OutputPath`:** Filtra o JSON de estado *depois* que o resultado foi mesclado, antes de passá-lo para o próximo estado.

### Insights e Melhores Práticas

* **Limite de Payload:** O "estado" (o JSON que passa entre os passos) tem um limite de **32.768 caracteres (32KB)**.
* **Padrão de Referência:** Se você precisa processar dados maiores (ex: um arquivo de 10MB), **não passe o dado no JSON**. Passe uma **referência** para ele (ex: `{"s3Bucket": "meu-bucket", "s3Key": "arquivo.json"}`). A função Lambda usa essa referência para buscar o dado diretamente do S3.
* **Tratamento de Erros Granular:** Use o `Retry` e `Catch` para tratar erros específicos da Lambda. Você pode capturar erros customizados que sua Lambda retorna (ex: `BusinessLogicError`) e direcionar o fluxo de forma diferente de um erro de sistema (ex: `Lambda.ServiceException`).

### Fonte Principal

* **[Manipulação de Input/Output no Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-input-output-filtering.html)**
* **[Tratamento de Erros no Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)**
* **[Padrão de Callback (Wait for Task Token)](https://docs.aws.amazon.com/step-functions/latest/dg/callback-tasks.html)**
