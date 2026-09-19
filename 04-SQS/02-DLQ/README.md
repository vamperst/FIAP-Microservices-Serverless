# 04.2 - DLQ Queue

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todos os comandos de terminal deste laboratório rodam no terminal do **Codespaces**. A criação da fila e as alterações de configuração da `demoqueue` são feitas no **console AWS** (aba SQS).

> [!WARNING]
> **Antes de começar, confirme os pré-requisitos abaixo (≈ 2 min). Tempo total estimado do laboratório: 20-25 min.**
> - [ ] Laboratório [04.1 - Standard Queue](../01-Standard-Queue/README.md) concluído, com a fila `demoqueue` já criada → valide com `aws sqs get-queue-url --queue-name demoqueue`
> - [ ] Credenciais AWS ativas no Codespaces → valide com `aws sts get-caller-identity`
> - [ ] `jq` disponível para extrair valores de JSON → valide com `jq --version`
> - [ ] Ambiente virtual Python com `boto3` do laboratório anterior ainda ativo (ou pronto para reativar com `source ~/venv/bin/activate`)

Neste laboratório você vai forçar, de propósito, uma falha de entrega: um consumidor que recebe mensagens mas nunca as remove da fila. Com o **Visibility Timeout** reduzido para 1 segundo e uma política de **redrive** configurada, você vai observar mensagens sendo redirecionadas automaticamente para uma **Dead Letter Queue (DLQ)** — o mecanismo que a AWS usa para isolar mensagens "problemáticas" sem travar o restante da fila.

## Principais pontos de aprendizagem

- Criar e associar uma Dead Letter Queue a uma fila SQS existente
- Configurar `RedrivePolicy` / `maxReceiveCount` para redirecionar mensagens automaticamente
- Entender o papel do `VisibilityTimeout` no reprocessamento de mensagens não confirmadas
- Purgar filas SQS de forma segura antes de repetir um teste

## O que você terá ao final

A fila `demoqueue` com redrive policy configurada, uma fila `demoqueue_DLQ` populada com as mensagens que o consumidor recebeu mas nunca confirmou, e a visão de como esse comportamento aparece no console SQS.

> [!TIP]
> Ao longo do texto você vai ver blocos `<details>💡 Clique para entender`. Eles são opcionais — aprofundam o "por quê" e o "como funciona por baixo dos panos", mas não são necessários para concluir o passo a passo. Abra quando quiser entender a mecânica, pule se só quiser avançar.

## Mapa do lab

| Parte | Descrição | Tempo | Passos |
|---|---|---|---|
| [Parte 1 - Preparando a fila e a DLQ](#parte-1---preparando-a-fila-e-a-dlq) | Criar a `demoqueue_DLQ` e configurar redrive na `demoqueue` | ~8 min | [1](#passo-1) · [2](#passo-2) · [3](#passo-3) · [4](#passo-4) |
| [Parte 2 - Populando e purgando](#parte-2---populando-e-purgando) | Ajustar o `put.py`, limpar as filas e enviar mensagens | ~7 min | [5](#passo-5) · [6](#passo-6) · [7](#passo-7) |
| [Parte 3 - Observando o redirecionamento](#parte-3---observando-o-redirecionamento) | Consumir sem confirmar e ver a DLQ ser populada | ~8 min | [8](#passo-8) · [9](#passo-9) · [10](#passo-10) |

<details>
<summary><b>💡 Clique para entender: o que é uma Dead Letter Queue</b></summary>
<blockquote>

Uma **Dead Letter Queue (DLQ)** é uma fila SQS comum, usada como destino de mensagens que uma fila de origem não conseguiu ter processadas com sucesso após um número configurado de tentativas (`maxReceiveCount`). Ela existe para isolar mensagens "problemáticas" — que causariam erro repetido no consumidor — sem deixar que elas bloqueiem o processamento das demais mensagens da fila principal. É um padrão de resiliência comum em qualquer arquitetura orientada a filas, não só na AWS.

</blockquote>
</details>

## Contexto

No laboratório anterior, o consumidor (Lambda) processava e removia as mensagens com sucesso. Mas, no mundo real, consumidores falham: um bug, uma dependência fora do ar, um payload inesperado. Se a fila simplesmente devolvesse a mensagem para o topo indefinidamente, uma mensagem "envenenada" travaria o processamento das demais para sempre. A DLQ resolve isso: depois de N tentativas sem sucesso, a mensagem é automaticamente movida para uma fila separada, onde pode ser inspecionada manualmente sem impactar o fluxo principal.

<a id="parte-1---preparando-a-fila-e-a-dlq"></a>

## Parte 1 - Preparando a fila e a DLQ

### Resultado esperado desta parte

Ao final desta parte, a `demoqueue` terá `VisibilityTimeout` de 1 segundo e uma `RedrivePolicy` apontando para a nova fila `demoqueue_DLQ`.

<a id="passo-1"></a>

**1. Entre na pasta do exercício**

<dl>
<dt></dt>
<dd>

```shell
cd /workspaces/FIAP-Microservices-Serverless/04-SQS/02-DLQ/
```

</dd>
</dl>

---

<a id="passo-2"></a>

**2. Crie a fila `demoqueue_DLQ`**

<dl>
<dt></dt>
<dd>

Na [aba do SQS](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/create-queue), crie uma fila com o mesmo nome da fila já criada acrescido do sufixo `_DLQ`, ficando `demoqueue_DLQ`. Mantenha todo o restante das informações com o que está pré-preenchido.

</dd>
</dl>

---

<a id="passo-3"></a>

**3. Abra a `demoqueue` para edição**

<dl>
<dt></dt>
<dd>

De volta ao painel de [listagem de filas do SQS](https://us-east-1.console.aws.amazon.com/sqs/v3/home?region=us-east-1#/queues), selecione a fila `demoqueue` e clique em **Editar** no canto superior direito.

![img/dlq-01.png](img/dlq-01.png)

</dd>
</dl>

---

<a id="passo-4"></a>

**4. Configure Visibility Timeout e Redrive Policy**

<dl>
<dt></dt>
<dd>

Preencha as informações como nas imagens e clique em **Salvar**. Na primeira imagem você altera o tempo de visibilidade para 1 segundo, para que a mensagem volte para a fila 1 segundo após ser entregue a um consumidor, mesmo que não tenha sido removida nesse meio tempo. Na segunda, você adiciona a `demoqueue_DLQ` como fila de mensagens mortas, configurando para que mensagens entregues mais de uma vez sejam enviadas para ela.

![img/dlq-02.png](img/dlq-02.png)

![img/dlq-02-1.png](img/dlq-02-1.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: VisibilityTimeout e RedrivePolicy por baixo dos panos</b></summary>
<blockquote>

Quando um consumidor chama `ReceiveMessage`, a mensagem não é removida — ela fica **invisível** para outros consumidores por `VisibilityTimeout` segundos. Se, nesse intervalo, ninguém chamar `DeleteMessage` para confirmar o processamento, a mensagem volta a ficar visível e pode ser entregue novamente. A `RedrivePolicy` monitora quantas vezes (`maxReceiveCount`) uma mesma mensagem foi recebida sem ser confirmada; ao ultrapassar esse número, a SQS move a mensagem automaticamente para a fila configurada como `deadLetterTargetArn`, em vez de devolvê-la para a fila de origem.

📚 Documentação oficial: [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/dead-letter-queues.html) — explica `RedrivePolicy` e `maxReceiveCount` em detalhe. 📚 [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html) — explica o comportamento do `VisibilityTimeout` usado aqui.

</blockquote>
</details>

### Checkpoint

- [ ] Fila `demoqueue_DLQ` criada e visível no console
- [ ] `demoqueue` com `VisibilityTimeout = 1` segundo e `RedrivePolicy` apontando para `demoqueue_DLQ`

---

<a id="parte-2---populando-e-purgando"></a>

## Parte 2 - Populando e purgando

### Resultado esperado desta parte

Ao final desta parte, as filas terão sido purgadas e a `demoqueue` estará novamente com 3.000 mensagens, prontas para o teste de redirecionamento.

<a id="passo-5"></a>

**5. Aponte o `put.py` para a `demoqueue`**

<dl>
<dt></dt>
<dd>

Altere o arquivo `put.py` colocando a URL da fila `demoqueue`. Para abrir, use:

```shell
code put.py
```

Para pegar a URL, você pode entrar no console do SQS ou usar o comando:

```shell
aws sqs get-queue-url --queue-name demoqueue | jq .QueueUrl
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o comando aws sqs get-queue-url</b></summary>
<blockquote>

Esse comando chama a API `GetQueueUrl`, que resolve o nome de uma fila para sua URL completa, sem você precisar copiar e colar manualmente do console. O `| jq .QueueUrl` apenas extrai o campo `QueueUrl` do JSON retornado.

📚 Documentação oficial: [GetQueueUrl - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_GetQueueUrl.html) — descreve o parâmetro `--queue-name` e o formato de retorno.

</blockquote>
</details>

---

<a id="passo-6"></a>

**6. Purgue todas as filas antes de testar**

<dl>
<dt></dt>
<dd>

```shell
for queue_url in $(aws sqs list-queues --query 'QueueUrls[*]' --output text); do
    aws sqs purge-queue --queue-url "$queue_url"
    echo "Purged queue: $queue_url"
done
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o que este comando faz por baixo dos panos</b></summary>
<blockquote>

O comando primeiro lista as URLs de todas as filas da conta com `aws sqs list-queues --query 'QueueUrls[*]' --output text`, depois itera sobre cada uma (`for queue_url in $(...); do ... done`) chamando `aws sqs purge-queue --queue-url "$queue_url"` para apagar todas as mensagens daquela fila específica, imprimindo uma confirmação a cada iteração. Rodar antes de cada teste garante que mensagens de execuções anteriores não interfiram na contagem que você vai observar. O comando é **idempotente**: purgar uma fila já vazia não tem efeito colateral, então é seguro rodar quantas vezes precisar.

📚 Documentação oficial: [PurgeQueue - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_PurgeQueue.html) — nota o limite de uma purga a cada 60 segundos por fila.

</blockquote>
</details>

---

<a id="passo-7"></a>

**7. Envie as mensagens de teste**

<dl>
<dt></dt>
<dd>

```shell
python3 put.py
```

</dd>
</dl>

### Checkpoint

- [ ] `demoqueue` e `demoqueue_DLQ` purgadas (0 mensagens em ambas antes do envio)
- [ ] `demoqueue` com mensagens disponíveis após `python3 put.py`

---

<a id="parte-3---observando-o-redirecionamento"></a>

## Parte 3 - Observando o redirecionamento

### Resultado esperado desta parte

Ao final desta parte, você terá rodado um consumidor que recebe mensagens sem confirmá-las, e verá essas mensagens sendo movidas automaticamente para a `demoqueue_DLQ`.

<a id="passo-8"></a>

**8. Aponte o `consumer.py` para a `demoqueue`**

<dl>
<dt></dt>
<dd>

O `consumer.py` já está no repositório e só precisa de uma alteração: trocar o `<url da sua fila>` pela URL da `demoqueue`. Para abrir:

```shell
code consumer.py
```

O código do arquivo, com os comentários omitidos, é este:

```python
from sqsHandler import SqsHandler

sqs = SqsHandler('<url da sua fila>')

while(True):
    response = sqs.getMessage(10, 20)

    if('Messages' not in response):
        break

    for msg in response['Messages']:
        print(msg['MessageId'])
```

A imagem abaixo mostra a mesma tela no vídeo da aula. Use o bloco de código acima como referência, não a imagem: a URL dela tem o número da conta da gravação (a sua é diferente) e o `if` dela usa `len(response['Messages'])`, que quebra com `KeyError` quando a fila fica sem mensagens visíveis.

![img/dlq-03.png](img/dlq-03.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: por que o loop precisa de long polling e do teste com <code>not in</code></b></summary>
<blockquote>

O segundo argumento de `getMessage(10, 20)` é o `WaitTimeSeconds` do `ReceiveMessage` — o **long polling**. Sem ele (`WaitTimeSeconds=0`), a chamada volta imediatamente e, como as mensagens ficam invisíveis por 1 segundo depois de cada leitura, é comum a resposta vir vazia com a fila ainda cheia: o loop encerraria antes de as redelivery levarem as mensagens para a DLQ, e o laboratório não demonstraria nada. Com 20 segundos de espera, a chamada só volta quando há mensagem disponível (ou ao fim do tempo), o que mantém o consumidor rodando até a fila realmente esvaziar.

Já o teste `if('Messages' not in response)` existe porque a SQS **omite a chave `Messages`** da resposta quando não há nada visível na fila — ela não devolve uma lista vazia. Por isso `len(response['Messages']) == 0` levanta `KeyError` em vez de encerrar o loop.

📚 Documentação oficial: [Amazon SQS short and long polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html) — compara os dois modos e explica quando a resposta volta vazia.

</blockquote>
</details>

---

<a id="passo-9"></a>

**9. Execute o consumidor**

<dl>
<dt></dt>
<dd>

```shell
python3 consumer.py
```

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: por que este consumidor causa o redirecionamento</b></summary>
<blockquote>

O `consumer.py` chama `ReceiveMessage` (via `getMessage` de `sqsHandler.py`) em loop, mas **nunca chama `DeleteMessage`**. Cada mensagem recebida fica invisível por 1 segundo (o `VisibilityTimeout` configurado no passo 4) e depois volta a ficar disponível para ser recebida de novo — até ultrapassar o `maxReceiveCount` da `RedrivePolicy`, momento em que a SQS a move para a `demoqueue_DLQ` em vez de devolvê-la para a `demoqueue`.

📚 Documentação oficial: [ReceiveMessage - Amazon SQS API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html) — descreve o comportamento de visibilidade após a leitura de uma mensagem.

</blockquote>
</details>

---

<a id="passo-10"></a>

**10. Observe a DLQ sendo populada**

<dl>
<dt></dt>
<dd>

Enquanto o script roda, acompanhe no [painel do SQS](https://console.aws.amazon.com/sqs/v2/home?region=us-east-1#/queues) a fila `demoqueue_DLQ` recebendo as mensagens que não foram confirmadas pelo `demoqueue`.

![img/dlq-04.png](img/dlq-04.png)

</dd>
</dl>

### Checkpoint

- [ ] `python3 consumer.py` terminou (loop encerra quando a `demoqueue` fica sem mensagens disponíveis)
- [ ] `demoqueue_DLQ` mostra mensagens disponíveis no console

## Conclusão

Você configurou uma Dead Letter Queue associada a uma fila SQS e observou, na prática, como o par `VisibilityTimeout` + `RedrivePolicy`/`maxReceiveCount` isola mensagens que um consumidor não conseguiu confirmar. Esse é o mecanismo de resiliência que evita que uma mensagem problemática trave o processamento de uma fila inteira.

## Próximo passo

Siga para [04.3 - Lambda](../03-Lambda/README.md): você vai reconfigurar a `demoqueue` para o comportamento normal e reforçar o fluxo de consumo via Lambda com o ajuste correto do ARN da fila.

<details>
<summary><b>💡 Glossário rápido</b></summary>

| Termo | Significado |
|---|---|
| Dead Letter Queue (DLQ) | Fila destino de mensagens que excederam o número de tentativas de processamento |
| RedrivePolicy | Configuração que associa uma fila de origem a uma DLQ |
| maxReceiveCount | Número máximo de tentativas antes do redirecionamento para a DLQ |
| VisibilityTimeout | Tempo em que uma mensagem lida fica invisível para outros consumidores |
| ReceiveMessage | Chamada de API que lê mensagens de uma fila sem removê-las |
| PurgeQueue | Chamada de API que remove todas as mensagens de uma fila |

</details>

<details>
<summary><b>💡 Como pedir ajuda se travou</b></summary>

Antes de chamar o professor ou monitor, tenha em mãos:

- [ ] O número exato do passo onde travou (ex: "travei no passo 9")
- [ ] O comando exato que executou (copiado do terminal, não de memória)
- [ ] A mensagem de erro completa (copiada do terminal, não resumida)
- [ ] O que você já tentou (reler o passo, checar a configuração da fila no console, etc.)

Canais por prioridade:

1. Releia o passo e o bloco `⚠ Se der erro` correspondente, se houver
2. Pergunte no canal da turma, com as 4 informações acima
3. Chame o professor/monitor presencialmente, já com o contexto reunido

</details>
