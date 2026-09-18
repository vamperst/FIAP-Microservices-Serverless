# 02.1 - Lambda

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todo comando de terminal deste laboratório roda no terminal do IDE criado pelo Codespaces.

> [!WARNING]
> **Pré-requisitos**
> - [ ] Codespaces do repositório criado e credenciais AWS configuradas (link acima) — valide com `aws sts get-caller-identity`
> - [ ] Serverless Framework disponível no terminal — valide com `sls --version`
> - [ ] Python 3 disponível no terminal — valide com `python3 --version`
>
> **Tempo estimado:** execução pura ~5 min + leitura/observação dos blocos `💡` ~15 min (**~20 min no total**).

Este é o primeiro contato do módulo com o **AWS Lambda**. A ideia é sair de zero código para uma função rodando de verdade na AWS no menor número de passos possível, usando o **Serverless Framework** como ferramenta de empacotamento e deploy — a mesma peça que vai aparecer em praticamente todo laboratório de Lambda daqui em diante.

## Principais pontos de aprendizagem

- Criar um serviço serverless a partir de um template (`sls create`).
- Entender a estrutura mínima de um serviço: `serverless.yml` (infraestrutura) + `handler.py` (código).
- Fazer deploy de uma função Lambda com um único comando.
- Invocar a função dos dois jeitos possíveis: remotamente (na AWS) e localmente (sem tocar a AWS).
- Remover toda a infraestrutura criada ao final, sem deixar nada "pra trás" na conta.

## O que você terá ao final

Uma função Lambda publicada na AWS, testada remotamente e localmente, e removida — a conta volta ao estado em que estava antes do laboratório.

> [!TIP]
> Os blocos `<details>💡 Clique para entender` ao longo do laboratório são opcionais: explicam o que o comando faz por baixo dos panos, com link para a documentação oficial. Não são necessários para completar o passo — servem para quem quer entender a mecânica, não só copiar o comando.

## Mapa do lab

| Parte | Descrição | Tempo | Passos |
|---|---|---|---|
| 1 - Criando e implantando a função | `sls create`, configurar `serverless.yml` e primeiro `sls deploy` | ~10 min | [1](#passo-1) a [5](#passo-5) |
| 2 - Testando, iterando e limpando | Invocar remoto e local, alterar o código e remover | ~10 min | [6](#passo-6) a [9](#passo-9) |

## Contexto

O **AWS Lambda** executa código sob demanda sem que você precise provisionar ou gerenciar servidor nenhum — você entrega a função, a AWS cuida do resto (escalonamento, disponibilidade, patch do runtime). O ganho para quem desenvolve é não pensar em infraestrutura; o custo é que o deploy e o empacotamento do código passam a ser um passo a mais no fluxo de trabalho.

É exatamente esse passo a mais que o **Serverless Framework** (comando `sls`) resolve neste laboratório: ele lê um arquivo `serverless.yml` declarando a função, empacota o código, cria (ou atualiza) uma stack do CloudFormation com tudo que a função precisa (a própria função, o log group e, quando você não indica uma role existente, também a role de execução) e permite invocar e remover essa stack com um comando cada. No Learner Lab você vai sempre indicar a `LabRole`, porque a conta não permite criar roles novas. Você vai ver essa mesma sequência — criar, configurar, `deploy`, `invoke`, `remove` — se repetir nos próximos laboratórios do módulo.

## Parte 1 - Criando e implantando a função

### Resultado esperado desta parte

Uma função Lambda chamada `hello`, publicada na AWS via CloudFormation, pronta para ser invocada.

<a id="passo-1"></a>

---

<dl>
<dt>

**1. Entre na pasta do exercício**

</dt>
<dd>

No terminal do IDE criado pelo Codespaces, execute:

```bash
cd /workspaces/FIAP-Microservices-Serverless/02-Lambda/01-Intro/
```

</dd>
</dl>

<a id="passo-2"></a>

---

<dl>
<dt>

**2. Inicie o repositório de trabalho**

</dt>
<dd>

```bash
sls create --template "aws-python3"
```

![img/slscreate.png](img/slscreate.png)

<details>
<summary>💡 Clique para entender: o que o <code>sls create</code> faz de verdade</summary>
<blockquote>

Este comando **não fala com a AWS** — é 100% local. O Serverless Framework copia o template `aws-python3` (mantido no repositório oficial de templates do framework) para a pasta atual, gerando dois arquivos: `serverless.yml` (a declaração da infraestrutura) e `handler.py` (o código da função, já com uma função `hello` de exemplo). Nenhuma credencial é usada neste passo porque nada é criado na sua conta ainda — isso só acontece no `sls deploy` (passo 5).

📚 Documentação oficial: [Serverless CLI reference - create](https://www.serverless.com/framework/docs/providers/aws/cli-reference/create) — lista os templates disponíveis além do `aws-python3` usado aqui.

</blockquote>
</details>

</dd>
</dl>

<a id="passo-3"></a>

---

<dl>
<dt>

**3. Abra o arquivo `serverless.yml`**

</dt>
<dd>

```bash
code serverless.yml
```

</dd>
</dl>

<a id="passo-4"></a>

---

<dl>
<dt>

**4. Configure a função conforme a imagem**

</dt>
<dd>

Altere o arquivo para que fique como na imagem abaixo. Para salvar, use **CTRL+S**.

![](img/yml1.png)

</dd>
</dl>

<a id="passo-5"></a>

---

<dl>
<dt>

**5. Faça o deploy da função**

</dt>
<dd>

No terminal do IDE:

```bash
sls deploy --verbose
```

A flag `--verbose` mostra cada etapa do deploy em detalhes, incluindo o status do CloudFormation sendo criado.

![img/slsdeploy.png](img/slsdeploy.png)

Este comando é **idempotente**: você pode rodá-lo quantas vezes precisar. Se editar `serverless.yml` ou `handler.py` depois deste passo, basta rodar `sls deploy` de novo para atualizar a função — não precisa remover nada antes.

<details>
<summary>💡 Clique para entender: o que o <code>sls deploy</code> faz por baixo dos panos</summary>
<blockquote>

O `sls deploy` empacota o conteúdo da pasta (o `handler.py`) em um `.zip`, envia esse pacote para um bucket S3 de deployment (criado automaticamente no primeiro deploy da região/conta) e então cria — ou atualiza, se a stack já existir — uma stack do **AWS CloudFormation** contendo os recursos `AWS::Lambda::Function`, `AWS::IAM::Role` e o log group correspondente no CloudWatch. O `--verbose` imprime os eventos da stack (o equivalente a chamar `DescribeStackEvents` em loop) enquanto o CloudFormation provisiona cada recurso.

📚 Documentação oficial: [Deploying - Serverless Framework](https://www.serverless.com/framework/docs/providers/aws/guide/deploying) — explica a diferença entre `sls deploy` (stack completa) e `sls deploy function` (só o código, mais rápido para iteração).
📚 Documentação oficial: [Understanding stacks - AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html) — detalha o ciclo de vida `CREATE_COMPLETE`/`UPDATE_COMPLETE` que aparece na saída do `--verbose`.

</blockquote>
</details>

</dd>
</dl>

### Checkpoint

- [ ] `sls deploy --verbose` terminou sem erro, com a stack em `CREATE_COMPLETE` (ou `UPDATE_COMPLETE` em execuções seguintes).
- [ ] A saída do deploy mostra o nome da função `hello` e o endpoint/ARN correspondente.

## Parte 2 - Testando, iterando e limpando

### Resultado esperado desta parte

A função testada dos dois jeitos (remoto e local), uma alteração de código redeployada, e a infraestrutura removida ao final.

<a id="passo-6"></a>

---

<dl>
<dt>

**6. Teste a função remotamente**

</dt>
<dd>

```bash
sls invoke -f hello
```

![img/slsinvoke.png](img/slsinvoke.png)

<details>
<summary>💡 Clique para entender: o que o <code>sls invoke</code> faz por baixo dos panos</summary>
<blockquote>

Sem a flag `local`, este comando chama a API **`Invoke`** do AWS Lambda (tipo de invocação `RequestResponse`) contra a função já publicada na AWS — é uma chamada real, que aparece no CloudWatch Logs da função. A resposta impressa no terminal é o valor de retorno (`return`) do handler, serializado em JSON.

📚 Documentação oficial: [Invoke - AWS Lambda API Reference](https://docs.aws.amazon.com/lambda/latest/api/API_Invoke.html) — parâmetro `InvocationType` é o que diferencia uma chamada síncrona (usada aqui) de uma assíncrona.

</blockquote>
</details>

<details>
<summary>⚠ Se der erro: onde ver o log da execução</summary>
<blockquote>

Se a invocação retornar erro, o traceback completo fica no CloudWatch Logs da função, não necessariamente no terminal. Para acompanhar em tempo real, abra um segundo terminal e rode:

```bash
aws logs tail /aws/lambda/<nome-da-funcao> --follow --since 10m --format short
```

O nome exato da função (com o sufixo gerado pelo Serverless Framework) aparece na saída do `sls deploy` do passo 5.

</blockquote>
</details>

</dd>
</dl>

<a id="passo-7"></a>

---

<dl>
<dt>

**7. Altere a versão da função**

</dt>
<dd>

```bash
code handler.py
```

Altere a versão do retorno da função para `1.1` no arquivo `handler.py`, como na imagem, e salve com **CTRL+S**.

![img/altereversao.png](img/altereversao.png)

</dd>
</dl>

<a id="passo-8"></a>

---

<dl>
<dt>

**8. Teste a função localmente**

</dt>
<dd>

```bash
sls invoke local -f hello
```

![img/slsinvokelocal.png](img/slsinvokelocal.png)

<details>
<summary>💡 Clique para entender: em que o <code>invoke local</code> difere do <code>invoke</code></summary>
<blockquote>

O `invoke local` **não faz nenhuma chamada à AWS** — ele importa e executa o handler Python diretamente no ambiente do Codespaces, simulando o contrato de entrada/saída do runtime Lambda. Por não depender de deploy nem de rede, é o ciclo mais rápido para validar uma alteração de código antes de gastar tempo com um `sls deploy` completo. Note que a alteração feita no passo 7 já aparece aqui sem precisar reempacotar nem redeploy — só é visível remotamente (passo 6) depois de um novo `sls deploy`.

📚 Documentação oficial: [Invoke Local - Serverless CLI reference](https://www.serverless.com/framework/docs/providers/aws/cli-reference/invoke-local) — lista as flags para simular variáveis de ambiente e payloads de evento sem precisar deployar.

</blockquote>
</details>

</dd>
</dl>

<a id="passo-9"></a>

---

<dl>
<dt>

**9. Remova a função criada**

</dt>
<dd>

```bash
sls remove
```

![img/slsremove.png](img/slsremove.png)

Este é o passo de limpeza: sem ele, a função e os recursos associados (log group) continuam na conta depois que o laboratório termina. O comando é seguro de rodar mais de uma vez — se a stack já tiver sido removida, ele apenas confirma que não há nada para remover.

<details>
<summary>💡 Clique para entender: o que o <code>sls remove</code> faz por baixo dos panos</summary>
<blockquote>

Este comando chama **`DeleteStack`** na stack do CloudFormation criada no passo 5, o que remove em cascata a função Lambda e o log group no CloudWatch. A `LabRole` **não** é removida: ela não pertence à stack, já existia na conta antes do deploy e é reusada pelos outros laboratórios. O bucket S3 de deployment (compartilhado entre laboratórios da mesma região/conta) também não é removido por este comando.

📚 Documentação oficial: [Remove - Serverless CLI reference](https://www.serverless.com/framework/docs/providers/aws/cli-reference/remove) — explica a ordem de remoção dos recursos e como o Serverless Framework decide o que pertence à stack.

</blockquote>
</details>

</dd>
</dl>

### Checkpoint

- [ ] `sls invoke -f hello` e `sls invoke local -f hello` retornaram a resposta da função sem erro.
- [ ] A versão `1.1` alterada no passo 7 aparece na saída do `invoke local`.
- [ ] `sls remove` terminou sem erro e a stack não aparece mais no CloudFormation.

## Conclusão

Você criou um serviço serverless do zero, fez o deploy de uma função Lambda real na AWS, testou essa função dos dois jeitos possíveis — remoto e local — e removeu toda a infraestrutura ao final. Esse ciclo (`create` → configurar → `deploy` → `invoke`/`invoke local` → `remove`) é a base de todos os laboratórios de Lambda que vêm a seguir.

## Próximo passo

[02.2 - Lambda Layers](../02-Layers/README.md): mesmo ciclo de deploy, mas agora com uma dependência externa (`boto3`) empacotada separadamente como uma **Layer**, em vez de junto com o código da função.

<details>
<summary>💡 Glossário rápido</summary>

| Termo | Significado |
|---|---|
| Lambda | Serviço da AWS que executa código sob demanda, sem servidor para gerenciar |
| Serverless Framework (`sls`) | Ferramenta de linha de comando que empacota o código e gera a stack de infraestrutura |
| `serverless.yml` | Arquivo que declara a função e seus recursos associados |
| Stack (CloudFormation) | Conjunto de recursos AWS criados/gerenciados como uma unidade |
| Invoke remoto | Chamada real à função já publicada na AWS |
| Invoke local | Execução do handler no próprio ambiente de desenvolvimento, sem tocar a AWS |

</details>

<details>
<summary>💡 Como pedir ajuda se travou</summary>

Antes de chamar o professor, tenha em mãos:

1. Em qual passo travou (o número, ex: "passo 5").
2. O comando exato que rodou.
3. A mensagem de erro completa (copie do terminal, não resuma).
4. O que já tentou (ex: "roubei `sls deploy` de novo e deu o mesmo erro").

Canais, em ordem de prioridade: sinalize em sala para o professor → grupo da turma → monitoria.

</details>
