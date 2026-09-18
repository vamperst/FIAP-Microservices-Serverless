# 04.1 - Standard Queue

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todos os comandos de terminal deste laboratório rodam no terminal do **Codespaces**. As filas SQS são criadas diretamente no **console AWS** (aba SQS), pois é o jeito mais rápido de visualizar a configuração default de uma fila pela primeira vez.

> [!WARNING]
> **Antes de começar, confirme os pré-requisitos abaixo (≈ 2 min). Tempo total estimado do laboratório: 25-30 min.**
> - [ ] Credenciais AWS configuradas no Codespaces → valide com `aws sts get-caller-identity`
> - [ ] Python 3 e pip3 disponíveis no Codespaces → valide com `python3 --version`
> - [ ] Serverless Framework instalado (já vem no devcontainer) → valide com `sls --version`
> - [ ] `jq` disponível para extrair valores de JSON → valide com `jq --version`

Neste laboratório você vai criar sua primeira fila **SQS Standard** pelo console AWS, popular ela com 3.000 mensagens usando `boto3`, e depois consumir essa fila com uma função **Lambda** que você invoca manualmente e que lê a fila por conta própria com `boto3`. É o fluxo produtor → fila → consumidor mais simples possível em arquitetura serverless, e serve de base para os dois próximos laboratórios (DLQ e Lambda avançado) — no 04.3 você vai trocar essa leitura manual por um gatilho automático.

## Principais pontos de aprendizagem

- Criar e configurar uma fila SQS Standard pelo console AWS
- Enviar mensagens em lote (`SendMessageBatch`) via `boto3`
- Ler e apagar mensagens de uma fila (`ReceiveMessage` / `DeleteMessage`) dentro de uma Lambda
- Fazer deploy e remoção de infraestrutura com o Serverless Framework

## O que você terá ao final

Duas filas SQS (`demoqueue` e `demoqueue_dest`) e uma função Lambda publicada, capaz de consumir mensagens da primeira fila e reenviá-las para a segunda, com toda a stack removível por um único comando.

> [!TIP]
> Ao longo do texto você vai ver blocos `<details>💡 Clique para entender`. Eles são opcionais — aprofundam o "por quê" e o "como funciona por baixo dos panos", mas não são necessários para concluir o passo a passo. Abra quando quiser entender a mecânica, pule se só quiser avançar.

## Mapa do lab

| Parte | Descrição | Tempo | Passos |
|---|---|---|---|
| [Parte 1 - Criando a fila SQS](#parte-1---criando-a-fila-sqs) | Criar a fila `demoqueue` pelo console | ~5 min | [1](#passo-1) · [2](#passo-2) |
| [Parte 2 - Enviando dados para a fila](#parte-2---enviando-dados-para-a-fila) | Popular a fila com 3.000 mensagens via Python | ~8 min | [3](#passo-3) · [4](#passo-4) · [5](#passo-5) · [6](#passo-6) · [7](#passo-7) |
| [Parte 3 - Consumindo com Lambda](#parte-3---consumindo-com-lambda) | Lambda que lê a fila, deploy e limpeza | ~15 min | [8](#passo-8) · [9](#passo-9) · [10](#passo-10) · [11](#passo-11) · [12](#passo-12) · [13](#passo-13) · [14](#passo-14) · [15](#passo-15) · [16](#passo-16) · [17](#passo-17) · [18](#passo-18) |

<details>
<summary><b>💡 Clique para entender: o que é uma fila SQS Standard</b></summary>
<blockquote>

O **Amazon SQS (Simple Queue Service)** é um serviço de fila de mensagens totalmente gerenciado. No modo **Standard** (o default), ele garante *at-least-once delivery* — ou seja, uma mensagem pode, em raras situações, ser entregue mais de uma vez, e a ordem de entrega não é garantida. Em troca disso, o Standard tem *throughput* praticamente ilimitado, o que o torna a escolha padrão para desacoplar produtores de consumidores em arquiteturas de microsserviços. O outro modo, FIFO, garante ordem e entrega única, mas com limites de throughput mais baixos — não é usado neste laboratório.

</blockquote>
</details>

## Contexto

Em uma arquitetura de microsserviços, componentes que produzem dados (produtores) raramente devem chamar diretamente os componentes que os consomem. Esse acoplamento direto faz o produtor esperar o consumidor responder, e se o consumidor cair, o produtor trava. O SQS resolve isso: o produtor apenas deposita mensagens na fila e segue seu fluxo; o consumidor (aqui, uma função Lambda) processa no seu próprio ritmo, inclusive escalando automaticamente conforme o tamanho da fila. Este laboratório reproduz esse fluxo do zero, com o menor número de peças possível.

<a id="parte-1---criando-a-fila-sqs"></a>

## Parte 1 - Criando a fila SQS

### Resultado esperado desta parte

Ao final desta parte você terá uma fila SQS Standard chamada `demoqueue` criada no console AWS, com a URL da fila copiada para uso nos próximos passos.

<a id="passo-1"></a>

**1. Crie a fila `demoqueue` no console SQS**

<dl>
<dt></dt>
<dd>

[Clique aqui para criar uma fila](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/create-queue), digite o nome `demoqueue`, deixe todos os demais valores no default e clique em **Criar Fila**.

![img/sqs01.png](img/sqs01.png)

![img/sqs01.png](img/sqs03.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: por que os defaults servem para este exercício</b></summary>
<blockquote>

O default de `VisibilityTimeout` é 30 segundos e o de retenção de mensagem é 4 dias — mais do que suficiente para os testes deste laboratório. Você vai alterar o `VisibilityTimeout` propositalmente no laboratório 04.2 (DLQ) para forçar um comportamento de erro; aqui, mantenha tudo como está.

📚 Documentação oficial: [CreateQueue - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_CreateQueue.html) — lista todos os atributos configuráveis na criação da fila, incluindo os defaults aplicados.

</blockquote>
</details>

---

<a id="passo-2"></a>

**2. Copie a URL da fila criada**

<dl>
<dt></dt>
<dd>

A URL fica disponível na tela de detalhes da fila, como no destaque da imagem abaixo. Guarde-a — ela será usada nos passos 5 e 11.

![](img/sqs02.png)

</dd>
</dl>

### Checkpoint

- [ ] Fila `demoqueue` aparece na [listagem de filas do SQS](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/queues)
- [ ] Você tem a URL da fila copiada em algum lugar acessível (bloco de notas, ou já colada no `put.py`)

---

<a id="parte-2---enviando-dados-para-a-fila"></a>

## Parte 2 - Enviando dados para a fila

### Resultado esperado desta parte

Ao final desta parte, 3.000 mensagens terão sido enviadas para a fila `demoqueue` em lotes de 10, usando o script `put.py` com `boto3`.

<a id="passo-3"></a>

**3. Entre na pasta do exercício**

<dl>
<dt></dt>
<dd>

```shell
cd /workspaces/FIAP-Microservices-Serverless/04-SQS/01-Standard-Queue/
```

</dd>
</dl>

---

<a id="passo-4"></a>

**4. Abra o arquivo `put.py`**

<dl>
<dt></dt>
<dd>

```shell
code put.py
```

</dd>
</dl>

---

<a id="passo-5"></a>

**5. Altere o `put.py` com a URL da fila**

<dl>
<dt></dt>
<dd>

Cole a URL da fila `demoqueue` (copiada no passo 2) no lugar indicado do arquivo.

![img/sendtoqueue01.png](img/sendtoqueue01.png)

</dd>
</dl>

---

<a id="passo-6"></a>

**6. Prepare o ambiente virtual Python com boto3**

<dl>
<dt></dt>
<dd>

```shell
pip3 install virtualenv && python3 -m venv ~/venv
source ~/venv/bin/activate
pip3 install boto3
```

</dd>
</dl>

<details>
<summary><b>⚠ Se der erro: <code>command not found: python3</code> ou falha ao criar o venv</b></summary>
<blockquote>

Confira se o Codespaces terminou de montar o devcontainer (a feature de Python é instalada automaticamente na criação do ambiente). Rode `python3 --version` isoladamente; se ainda falhar, feche e reabra o terminal do Codespaces.

</blockquote>
</details>

---

<a id="passo-7"></a>

**7. Envie 3.000 mensagens para a fila**

<dl>
<dt></dt>
<dd>

```shell
python3 put.py
```

Verifique no [console SQS](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/queues) o resultado — o campo "Mensagens disponíveis" da `demoqueue` deve subir para 3.000.

![alt](img/sendtoqueue02.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: como o put.py envia 3.000 mensagens</b></summary>
<blockquote>

O script monta uma lista de 3.000 mensagens e as divide em lotes de 10 (o máximo permitido por chamada), enviando cada lote com o método `sendBatch` de `sqsHandler.py`, que por baixo dos panos chama a API `SendMessageBatch`. Enviar em lote em vez de mensagem a mensagem reduz o número de chamadas de rede de 3.000 para 300.

📚 Documentação oficial: [SendMessageBatch - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html) — detalha o limite de 10 mensagens por lote e o formato de cada entrada.

</blockquote>
</details>

### Checkpoint

- [ ] `python3 put.py` terminou sem erro
- [ ] O console SQS mostra ~3.000 mensagens disponíveis em `demoqueue`

---

<a id="parte-3---consumindo-com-lambda"></a>

## Parte 3 - Consumindo com Lambda

### Resultado esperado desta parte

Ao final desta parte, uma função Lambda estará publicada, lendo mensagens da `demoqueue` e reenviando-as para a fila `demoqueue_dest` — e depois removida, deixando a conta limpa.

<a id="passo-8"></a>

**8. Crie a fila de destino `demoqueue_dest`**

<dl>
<dt></dt>
<dd>

[Crie mais uma fila](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/create-queue) usando o mesmo procedimento do passo 1, com o nome `demoqueue_dest` (mesmo nome da anterior, com o sufixo `_dest`).

</dd>
</dl>

---

<a id="passo-9"></a>

**9. Crie o projeto Serverless**

<dl>
<dt></dt>
<dd>

```shell
sls create --template "aws-python3"
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o que o sls create faz por baixo dos panos</b></summary>
<blockquote>

O comando gera, na pasta atual, o esqueleto padrão de um projeto Serverless Framework para Python na AWS: um `serverless.yml` (definição de infraestrutura como código) e um `handler.py` (código da função). Nenhuma chamada à AWS acontece aqui — é só *scaffolding* local, por isso é seguro rodar quantas vezes quiser.

📚 Documentação oficial: [Serverless Framework — create](https://www.serverless.com/framework/docs/providers/aws/cli-reference/create) — lista os templates disponíveis, incluindo `aws-python3`.

</blockquote>
</details>

---

<a id="passo-10"></a>

**10. Abra o `serverless.yml`**

<dl>
<dt></dt>
<dd>

```shell
code serverless.yml
```

</dd>
</dl>

---

<a id="passo-11"></a>

**11. Configure o `serverless.yml` com as duas URLs de fila**

<dl>
<dt></dt>
<dd>

Substitua o conteúdo do arquivo pelo mostrado na imagem, preenchendo a URL da `demoqueue` (origem) e da `demoqueue_dest` (destino) exatamente como indicado.

![img/lambda-01.png](img/lambda-01.png)

Confirme que a seção `provider` aponta a função para a role `LabRole`, que já existe na conta do Learner Lab. Sem essa linha o deploy falha, porque a conta do AWS Academy não permite criar roles novas:

```yaml
provider:
  name: aws
  runtime: python3.11
  iam:
    role: !Sub arn:aws:iam::${AWS::AccountId}:role/LabRole
```

O `${AWS::AccountId}` é resolvido pelo próprio CloudFormation no momento do deploy, com o número da conta em que você está. Você não precisa descobrir nem digitar o seu — o bloco acima é igual para todos.

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: por que aqui a Lambda recebe as URLs das filas como variável de ambiente</b></summary>
<blockquote>

Neste laboratório a função **não** tem gatilho: ela não é acionada pela fila, e sim invocada por você no passo 15. Por isso o `serverless.yml` não declara nenhum `events` — só passa as duas URLs em `provider.environment` (`sqs_url` e `sqs_url_dest`), e é o próprio `handler.py` que chama `ReceiveMessage` na fila de origem e `SendMessage` na de destino usando `boto3`.

É também por isso que o `timeout` está em **300 segundos**: o handler faz até 100 rodadas de leitura dentro da mesma invocação, e precisa de tempo de execução para isso. Como não existe gatilho de SQS, o `VisibilityTimeout` de 30 segundos da fila (o default que você manteve no passo 1) não interfere no deploy.

No laboratório 04.3 você vai fazer o contrário: declarar a fila como gatilho de verdade e deixar a AWS invocar a função — e lá o `serverless.yml` vai precisar do **ARN** da fila, não da URL.

📚 Documentação oficial: [Serverless Framework — environment variables](https://www.serverless.com/framework/docs/providers/aws/guide/variables) — como o `provider.environment` chega ao código da função.

</blockquote>
</details>

---

<a id="passo-12"></a>

**12. Crie o `handler.py`**

<dl>
<dt></dt>
<dd>

Crie o arquivo com o conteúdo mostrado na imagem. Abra com:

```shell
code handler.py
```

![img/lambda-02.png](img/lambda-02.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: de onde vem o limite de 1.000 mensagens por execução</b></summary>
<blockquote>

O handler tem um laço `for i in range(100)` e, em cada rodada, chama `getMessage(10)` — ou seja, `ReceiveMessage` pedindo no máximo 10 mensagens (o limite da API por chamada). São, no pior caso, 100 × 10 = **1.000 mensagens por invocação**. Se a fila esvaziar antes, o `break` encerra o laço.

Cada mensagem lida é reenviada para a `demoqueue_dest` e só então apagada da origem com `deleteMessage(ReceiptHandle)`. O `ReceiptHandle` é um identificador temporário devolvido pelo `ReceiveMessage` — é ele, e não o `MessageId`, que a SQS exige para apagar uma mensagem.

📚 Documentação oficial: [ReceiveMessage - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html) — documenta o limite de 10 mensagens por chamada e o papel do `ReceiptHandle`.

</blockquote>
</details>

---

<a id="passo-13"></a>

**13. Faça o deploy da função Lambda**

<dl>
<dt></dt>
<dd>

```shell
sls deploy
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o que o sls deploy faz por baixo dos panos</b></summary>
<blockquote>

O `sls deploy` empacota o código, gera um template do CloudFormation a partir do `serverless.yml` e sobe (ou atualiza) uma *stack* na conta AWS, criando a função Lambda. O *role* de execução não é criado: a função reusa a `LabRole` que você indicou no `provider`, e é dela que vem a permissão para ler da SQS. O comando é **idempotente**: rodar `sls deploy` de novo sobre uma stack já publicada apenas aplica o diff, sem duplicar recursos — pode rodar quantas vezes precisar.

📚 Documentação oficial: [Serverless Framework — deploy](https://www.serverless.com/framework/docs/providers/aws/cli-reference/deploy) — detalha o ciclo de empacotamento e atualização de stack via CloudFormation.

</blockquote>
</details>

<details>
<summary><b>⚠ Se der erro: deploy falha por permissão ou credencial expirada</b></summary>
<blockquote>

As credenciais do AWS Academy Learner Lab expiram periodicamente. Refaça o passo de [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md) e rode `sls deploy` novamente — é seguro repetir.

Se o erro citar `iam:CreateRole`, por exemplo:

```
CREATE_FAILED: IamRoleLambdaExecution
User: arn:aws:sts::<conta>:assumed-role/voclabs/... is not authorized to
perform: iam:CreateRole on resource: .../...-dev-us-east-1-lambdaRole
```

a causa é a linha `iam.role` faltando no `provider` do `serverless.yml`. O Learner Lab bloqueia a criação de roles, e sem essa linha o Serverless Framework tenta criar uma role própria para o serviço. Volte ao [passo 11](#passo-11), acrescente o `iam.role` apontando para a `LabRole` e rode `sls deploy` de novo.

</blockquote>
</details>

---

<a id="passo-14"></a>

**14. Reabasteça a fila de origem**

<dl>
<dt></dt>
<dd>

```shell
python3 put.py
```

Lembre-se: cada execução do Lambda consome até 1.000 posições da fila SQS, por causa do laço do `handler.py` criado no passo 12.

</dd>
</dl>

---

<a id="passo-15"></a>

**15. Invoque o Lambda manualmente**

<dl>
<dt></dt>
<dd>

```shell
sls invoke -l -f sqshandler
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o que o sls invoke -l faz por baixo dos panos</b></summary>
<blockquote>

O `sls invoke` chama a API `Invoke` do Lambda de forma síncrona (`RequestResponse`) contra a função já publicada na AWS — não é execução local. A flag `-l` (`--log`) busca e imprime no terminal o log gerado pela execução, direto do CloudWatch Logs.

📚 Documentação oficial: [Invoke - AWS Lambda API Reference](https://docs.aws.amazon.com/lambda/latest/dg/API_Invoke.html) — descreve o tipo de invocação síncrona usado aqui.

</blockquote>
</details>

---

<a id="passo-16"></a>

**16. Observe as mensagens se movendo no painel SQS**

<dl>
<dt></dt>
<dd>

Enquanto o comando anterior espera, acompanhe no [painel do SQS](https://console.aws.amazon.com/sqs/v2/home?region=us-east-1#/queues) as mensagens saindo de `demoqueue` e chegando em `demoqueue_dest`. Atualize manualmente pelo ícone no canto superior direito do painel — cada execução do Lambda move até 1.000 mensagens, por definição do laço no código.

![alt](img/lambda-02-1.png)

</dd>
</dl>

---

<a id="passo-17"></a>

**17. Confirme que a fila principal esvazia**

<dl>
<dt></dt>
<dd>

Repita o [passo 15](#passo-15) algumas vezes — com 3.000 mensagens na fila e até 1.000 por execução, são cerca de três invocações até a `demoqueue` zerar as mensagens disponíveis, todas movidas para `demoqueue_dest`.

</dd>
</dl>

### Checkpoint

- [ ] `sls deploy` terminou sem erro e a função aparece no console Lambda
- [ ] `demoqueue` chegou a zero mensagens disponíveis
- [ ] `demoqueue_dest` acumulou as mensagens reenviadas

---

<a id="passo-18"></a>

**18. Remova a stack criada**

<dl>
<dt></dt>
<dd>

```shell
sls remove
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o que o sls remove faz por baixo dos panos</b></summary>
<blockquote>

O comando deleta a *stack* do CloudFormation criada no passo 13, removendo a função Lambda e os recursos que a stack criou. A `LabRole` **não** é removida: ela não pertence à stack, já existia na conta antes do deploy e continua lá para os próximos laboratórios. É **idempotente**: rodar `sls remove` numa stack que já não existe apenas retorna sem erro relevante, então não há problema em rodar de novo caso tenha dúvida se já removeu.

📚 Documentação oficial: [Serverless Framework — remove](https://www.serverless.com/framework/docs/providers/aws/cli-reference/remove) — detalha o processo de remoção da stack.

</blockquote>
</details>

## Conclusão

Você criou uma fila SQS Standard, populou ela via `boto3` em lotes, e publicou uma função Lambda que lê a fila de origem e reenvia as mensagens para a fila de destino — o padrão produtor/fila/consumidor mais comum em arquiteturas serverless. Aqui o consumo foi disparado por você, na mão; no laboratório 04.3 a própria fila vai passar a invocar a função. As filas `demoqueue` e `demoqueue_dest` continuam existindo (só a stack Lambda foi removida); você vai reaproveitá-las no próximo laboratório.

## Próximo passo

Siga para [04.2 - DLQ](../02-DLQ/README.md): você vai forçar falhas de entrega propositalmente e observar como uma Dead Letter Queue captura as mensagens que não conseguem ser processadas.

<details>
<summary><b>💡 Glossário rápido</b></summary>

| Termo | Significado |
|---|---|
| SQS Standard Queue | Fila com entrega *at-least-once* e sem garantia de ordem, throughput praticamente ilimitado |
| VisibilityTimeout | Tempo em que uma mensagem lida fica invisível para outros consumidores |
| ReceiptHandle | Identificador temporário devolvido pelo `ReceiveMessage`, exigido para apagar a mensagem |
| SendMessageBatch | Operação que envia até 10 mensagens em uma única chamada à SQS |
| boto3 | SDK oficial da AWS para Python |
| venv | Ambiente virtual Python, isola dependências do projeto do restante do sistema |
| Serverless Framework | Ferramenta de infraestrutura como código para funções serverless (Lambda, API Gateway, etc.) |

</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>

Antes de chamar o professor ou monitor, tenha em mãos:

- [ ] O número exato do passo onde travou (ex: "travei no passo 13")
- [ ] O comando exato que executou (copiado do terminal, não de memória)
- [ ] A mensagem de erro completa (copiada do terminal, não resumida)
- [ ] O que você já tentou (reler o passo, rodar `aws sts get-caller-identity`, etc.)

Canais por prioridade:

1. Releia o passo e o bloco `⚠ Se der erro` correspondente, se houver
2. Pergunte no canal da turma, com as 4 informações acima
3. Chame o professor/monitor presencialmente, já com o contexto reunido

</details>
