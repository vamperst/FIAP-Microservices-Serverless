# 04.3 - Lambda

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todos os comandos de terminal deste laboratório rodam no terminal do **Codespaces**. O ajuste de configuração da `demoqueue` é feito no **console AWS** (aba SQS).

> [!WARNING]
> **Antes de começar, confirme os pré-requisitos abaixo (≈ 2 min). Tempo total estimado do laboratório: 20-25 min.**
> - [ ] Laboratórios [04.1](../01-Standard-Queue/README.md) e [04.2](../02-DLQ/README.md) concluídos, com `demoqueue` e `demoqueue_dest` já criadas → valide com `aws sqs get-queue-url --queue-name demoqueue`
> - [ ] Credenciais AWS ativas no Codespaces → valide com `aws sts get-caller-identity`
> - [ ] Serverless Framework instalado (já vem no devcontainer) → valide com `sls --version`
> - [ ] `jq` disponível para extrair valores de JSON → valide com `jq --version`

No laboratório 04.2, você deixou a `demoqueue` com `VisibilityTimeout` de 1 segundo de propósito, para forçar mensagens a caírem na DLQ. Aqui você vai devolver essa fila ao comportamento normal e reforçar o fluxo de consumo via Lambda — desta vez com atenção especial ao ARN da fila de origem, que precisa ser referenciado corretamente no `serverless.yml` para o Event Source Mapping funcionar.

## Principais pontos de aprendizagem

- Referenciar o ARN de uma fila SQS existente dentro de um `serverless.yml`
- Reconfigurar atributos de uma fila SQS já em uso por outros laboratórios
- Reforçar o ciclo completo deploy → teste → remoção com Serverless Framework

## O que você terá ao final

Uma função Lambda publicada e validada, consumindo da `demoqueue` (já com `VisibilityTimeout` normalizado) e reenviando para `demoqueue_dest`, com a stack removida ao final do teste.

> [!TIP]
> Ao longo do texto você vai ver blocos `<details>💡 Clique para entender`. Eles são opcionais — aprofundam o "por quê" e o "como funciona por baixo dos panos", mas não são necessários para concluir o passo a passo. Abra quando quiser entender a mecânica, pule se só quiser avançar.

## Mapa do lab

| Parte | Descrição | Tempo | Passos |
|---|---|---|---|
| [Parte 1 - Preparando o projeto Lambda](#parte-1---preparando-o-projeto-lambda) | Criar o projeto Serverless, ajustar `handler.py` e `serverless.yml` | ~10 min | [1](#passo-1) · [2](#passo-2) · [3](#passo-3) · [4](#passo-4) |
| [Parte 2 - Ajustando a fila e publicando](#parte-2---ajustando-a-fila-e-publicando) | Normalizar a `demoqueue` e fazer o deploy | ~7 min | [5](#passo-5) · [6](#passo-6) |
| [Parte 3 - Testando e limpando](#parte-3---testando-e-limpando) | Enviar mensagens, validar o consumo e remover a stack | ~8 min | [7](#passo-7) · [8](#passo-8) · [9](#passo-9) |

<details>
<summary><b>💡 Clique para entender: por que o ARN da fila importa aqui</b></summary>
<blockquote>

Diferente do `serverless.yml` do laboratório 04.1, onde as URLs das filas bastavam como variáveis de ambiente, o gatilho SQS de uma função Lambda no `serverless.yml` precisa do **ARN** da fila (não a URL) para declarar o Event Source Mapping. URL identifica um endpoint de API; ARN identifica um recurso da AWS de forma única entre contas e serviços — é isso que o Lambda usa para se inscrever como consumidor da fila.

</blockquote>
</details>

## Contexto

Este laboratório fecha o ciclo do módulo de SQS: depois de aprender o fluxo básico (04.1) e o mecanismo de resiliência com DLQ (04.2), aqui você exercita o detalhe que mais gera erro na prática — referenciar corretamente o recurso (ARN, não URL) ao conectar um gatilho SQS a uma função Lambda, e lembrar de desfazer configurações de teste (como o `VisibilityTimeout` reduzido) antes de seguir para o próximo cenário.

<a id="parte-1---preparando-o-projeto-lambda"></a>

## Parte 1 - Preparando o projeto Lambda

### Resultado esperado desta parte

Ao final desta parte, você terá um projeto Serverless com `handler.py` e `serverless.yml` configurados para consumir a `demoqueue` e reenviar para a `demoqueue_dest`.

<a id="passo-1"></a>

**1. Entre na pasta do exercício**

<dl>
<dt></dt>
<dd>

```shell
cd /workspaces/FIAP-Microservices-Serverless/04-SQS/03-Lambda/
```

</dd>
</dl>

---

<a id="passo-2"></a>

**2. Crie o projeto Serverless**

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

O comando gera, na pasta atual, o esqueleto padrão de um projeto Serverless Framework para Python na AWS: um `serverless.yml` e um `handler.py`. Nenhuma chamada à AWS acontece aqui — é só *scaffolding* local, por isso é seguro rodar quantas vezes quiser.

📚 Documentação oficial: [Serverless Framework — create](https://www.serverless.com/framework/docs/providers/aws/cli-reference/create) — lista os templates disponíveis, incluindo `aws-python3`.

</blockquote>
</details>

---

<a id="passo-3"></a>

**3. Configure o `handler.py` com a URL da fila de destino**

<dl>
<dt></dt>
<dd>

Altere o `handler.py` para ficar como na imagem, sem esquecer de colocar a URL da sua fila de destino. Abra com:

```shell
code handler.py
```

Para conseguir a URL da fila de destino:

```shell
aws sqs get-queue-url --queue-name demoqueue_dest | jq .QueueUrl
```

![at](img/lambda-01.png)

A URL que aparece na imagem contém o número da conta de quem gravou a aula. A sua é diferente: use a que o comando acima devolveu, não a da imagem.

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o comando aws sqs get-queue-url</b></summary>
<blockquote>

Esse comando chama a API `GetQueueUrl`, que resolve o nome de uma fila para sua URL completa. Aqui ela é usada para preencher a URL de destino no código do handler, evitando erro de digitação ao copiar do console.

📚 Documentação oficial: [GetQueueUrl - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_GetQueueUrl.html) — descreve o parâmetro `--queue-name` e o formato de retorno.

</blockquote>
</details>

---

<a id="passo-4"></a>

**4. Configure o `serverless.yml` com o ARN da fila de origem**

<dl>
<dt></dt>
<dd>

Altere o `serverless.yml` para que fique como na imagem. Para abrir:

```shell
code serverless.yml
```

Para pegar o ARN da `demoqueue`, use o comando abaixo:

```shell
demoqueueURL=`aws sqs get-queue-url --queue-name demoqueue | jq -r .QueueUrl` && aws sqs get-queue-attributes --queue-url $demoqueueURL --attribute-names QueueArn | jq -r .Attributes.QueueArn
```

![at](img/lambda-02.png)

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
<summary><b>💡 Clique para entender: o comando de ARN e o event source mapping</b></summary>
<blockquote>

O comando encadeia duas chamadas: `GetQueueUrl` resolve o nome da fila para sua URL, e `GetQueueAttributes` com `--attribute-names QueueArn` retorna o ARN a partir dessa URL. O `serverless.yml` usa esse ARN na seção `functions.<nome>.events` para declarar a `demoqueue` como gatilho da Lambda, junto com o parâmetro `batchSize`. É esse ARN — não a URL — que o Serverless Framework usa para criar o Event Source Mapping no deploy.

📚 Documentação oficial: [GetQueueAttributes - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_GetQueueAttributes.html) — lista os atributos disponíveis, incluindo `QueueArn`. 📚 [Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html) — explica o papel do ARN e do `batchSize` no event source mapping.

</blockquote>
</details>

### Checkpoint

- [ ] `handler.py` com a URL de `demoqueue_dest` preenchida
- [ ] `serverless.yml` com o ARN correto da `demoqueue` no gatilho SQS

---

<a id="parte-2---ajustando-a-fila-e-publicando"></a>

## Parte 2 - Ajustando a fila e publicando

### Resultado esperado desta parte

Ao final desta parte, a `demoqueue` estará com o `VisibilityTimeout` normalizado e a função Lambda estará publicada na conta AWS.

<a id="passo-5"></a>

**5. Normalize o Visibility Timeout da `demoqueue`**

<dl>
<dt></dt>
<dd>

Vá à sua aba do SQS e configure a `demoqueue` para ficar como na imagem — isso é necessário pois o tempo de visibilidade padrão estava em 1 segundo, ajustado no laboratório anterior para forçar erros no teste da DLQ.

![at](img/lambda-03.png)

</dd>
</dl>

<details>
<summary><b>⚠ Se der erro: mensagens continuam indo para a DLQ neste laboratório</b></summary>
<blockquote>

Se a `demoqueue` ainda estiver com `VisibilityTimeout` de 1 segundo (configuração do laboratório 04.2), a Lambda pode não conseguir confirmar o processamento a tempo, e as mensagens voltarão a ser redirecionadas para a `demoqueue_DLQ`. Confirme no console que o valor foi salvo como na imagem antes de seguir.

</blockquote>
</details>

---

<a id="passo-6"></a>

**6. Faça o deploy da função Lambda**

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

O `sls deploy` empacota o código, gera um template do CloudFormation a partir do `serverless.yml` e sobe (ou atualiza) uma *stack* na conta AWS, criando a função Lambda e o Event Source Mapping com a `demoqueue`. O *role* de execução não é criado: a função reusa a `LabRole` que você indicou no `provider`. O comando é **idempotente**: rodar `sls deploy` de novo sobre uma stack já publicada apenas aplica o diff, sem duplicar recursos.

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

a causa é a linha `iam.role` faltando no `provider` do `serverless.yml`. O Learner Lab bloqueia a criação de roles, e sem essa linha o Serverless Framework tenta criar uma role própria para o serviço. Volte ao [passo 4](#passo-4), acrescente o `iam.role` apontando para a `LabRole` e rode `sls deploy` de novo.

</blockquote>
</details>

### Checkpoint

- [ ] `demoqueue` com `VisibilityTimeout` normalizado (não mais 1 segundo)
- [ ] `sls deploy` terminou sem erro e a função aparece no console Lambda

---

<a id="parte-3---testando-e-limpando"></a>

## Parte 3 - Testando e limpando

### Resultado esperado desta parte

Ao final desta parte, você terá validado que as mensagens fluem de `demoqueue` para `demoqueue_dest` via Lambda, e removido a stack criada.

<a id="passo-7"></a>

**7. Aponte o `put.py` para a `demoqueue`**

<dl>
<dt></dt>
<dd>

Altere o arquivo `put.py` colocando a URL da sua fila `demoqueue`. Abra com:

```shell
code put.py
```

</dd>
</dl>

---

<a id="passo-8"></a>

**8. Envie mensagens e observe o painel SQS**

<dl>
<dt></dt>
<dd>

```shell
python3 put.py
```

Observe no [painel do SQS](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/queues) que as mensagens estão indo para a fila de destino.

![at](img/lambda-04.png)

</dd>
</dl>

### Checkpoint

- [ ] `demoqueue` recebe as mensagens de `python3 put.py`
- [ ] `demoqueue_dest` acumula as mensagens reenviadas pela Lambda

---

<a id="passo-9"></a>

**9. Remova a stack criada**

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

O comando deleta a *stack* do CloudFormation criada no passo 6, removendo a função Lambda e o Event Source Mapping associado. A `LabRole` **não** é removida: ela não pertence à stack e já existia na conta antes do deploy. É **idempotente**: rodar `sls remove` numa stack que já não existe apenas retorna sem erro relevante.

📚 Documentação oficial: [Serverless Framework — remove](https://www.serverless.com/framework/docs/providers/aws/cli-reference/remove) — detalha o processo de remoção da stack.

</blockquote>
</details>

## Conclusão

Você fechou o ciclo do módulo de SQS: referenciou corretamente o ARN de uma fila existente num `serverless.yml`, normalizou uma configuração de teste (`VisibilityTimeout`) antes de seguir para um novo cenário, e validou de ponta a ponta o fluxo produtor → fila → Lambda → fila de destino, com a stack removida ao final.

## Próximo passo

Siga para [05 - Atividade final](../../05-Atividade-final/README.md), onde você vai aplicar o que aprendeu nos três laboratórios de SQS numa atividade avaliativa.

<details>
<summary><b>💡 Glossário rápido</b></summary>

| Termo | Significado |
|---|---|
| ARN | Identificador único de um recurso AWS, usado por outros serviços para referenciá-lo |
| GetQueueAttributes | Chamada de API que retorna atributos de uma fila, como o `QueueArn` |
| Event Source Mapping | Configuração que faz o Lambda fazer *polling* automático de uma fonte de eventos, como SQS |
| batchSize | Quantidade de mensagens entregues por invocação do Lambda |
| VisibilityTimeout | Tempo em que uma mensagem lida fica invisível para outros consumidores |
| Serverless Framework | Ferramenta de infraestrutura como código para funções serverless (Lambda, API Gateway, etc.) |

</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>

Antes de chamar o professor ou monitor, tenha em mãos:

- [ ] O número exato do passo onde travou (ex: "travei no passo 6")
- [ ] O comando exato que executou (copiado do terminal, não de memória)
- [ ] A mensagem de erro completa (copiada do terminal, não resumida)
- [ ] O que você já tentou (reler o passo, checar o ARN colado no `serverless.yml`, etc.)

Canais por prioridade:

1. Releia o passo e o bloco `⚠ Se der erro` correspondente, se houver
2. Pergunte no canal da turma, com as 4 informações acima
3. Chame o professor/monitor presencialmente, já com o contexto reunido

</details>
