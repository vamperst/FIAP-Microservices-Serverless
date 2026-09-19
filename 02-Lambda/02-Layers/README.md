# 02.2 - Lambda Layers

**Antes de começar, execute os passos abaixo para configurar o ambiente caso não tenha feito isso ainda na aula de HOJE: [Preparando Credenciais](../../01-create-codespaces/Inicio-de-aula.md)**

Todo comando de terminal deste laboratório roda no terminal do IDE criado pelo Codespaces; os passos de validação no console rodam no navegador, na conta AWS da FIAP.

> [!WARNING]
> **Pré-requisitos**
> - [ ] Laboratório [02.1 - Lambda](../01-Intro/README.md) concluído (mesmo ciclo `create`/`deploy`/`remove`, usado aqui sem repetir a explicação)
> - [ ] Serverless Framework disponível no terminal — valide com `sls --version`
> - [ ] Python 3 e `pip3` disponíveis no terminal — valide com `python3 --version` e `pip3 --version`
> - [ ] Acesso ao console AWS (região `us-east-1`) já configurado
>
> **Tempo estimado:** execução pura ~10 min + leitura/observação dos blocos `💡` ~15 min (**~25 min no total**).

No laboratório anterior, o código da função inteira (`handler.py`) foi empacotado junto no deploy. Na prática, é raro uma função depender só do próprio código — bibliotecas externas (como `boto3`) também precisam ir no pacote. Este laboratório mostra a alternativa: empacotar a dependência **separadamente**, como uma **Lambda Layer**, e anexá-la à função em vez de subir tudo junto a cada deploy.

## Principais pontos de aprendizagem

- Diferença entre o código da função e uma **Layer** (dependência compartilhada, empacotada separadamente).
- Como montar o pacote de uma Layer localmente com `pip3 install -t` dentro de um ambiente virtual isolado.
- Como declarar uma Layer no `serverless.yml` e referenciá-la na função.
- Onde confirmar, no console AWS, que a função e a Layer foram criadas e estão associadas.

## O que você terá ao final

Uma função Lambda com uma Layer própria anexada (contendo `boto3`), publicada, validada no console e testada por invocação — depois removida.

> [!TIP]
> Os blocos `<details>💡 Clique para entender` ao longo do laboratório são opcionais: explicam o que o comando faz por baixo dos panos, com link para a documentação oficial. Não são necessários para completar o passo — servem para quem quer entender a mecânica, não só copiar o comando.

## Mapa do lab

| Parte | Descrição | Tempo | Passos |
|---|---|---|---|
| 1 - Empacotando a dependência como Layer | Criar o serviço, isolar o `boto3` num ambiente virtual e montar o pacote da Layer | ~15 min | [1](#passo-1) a [9](#passo-9) |
| 2 - Deploy, validação no console e teste | `sls deploy`, confirmar Layer no console e invocar a função | ~10 min | [10](#passo-10) a [15](#passo-15) |

## Contexto

Uma **Lambda Layer** é um pacote `.zip` versionado, independente do código da função, que é anexado a uma ou mais funções em tempo de execução. O caso de uso mais comum é exatamente este: bibliotecas de terceiros (como o `boto3`) que não mudam a cada deploy do código não precisam ser reempacotadas junto com a função — ficam numa Layer separada, reaproveitável por outras funções do mesmo time.

O trade-off é que montar essa Layer exige um passo manual antes do deploy: instalar a dependência num diretório específico, no formato que o runtime da Lambda espera encontrar (`python/` na raiz do zip, para runtimes Python). É esse passo manual — isolado num ambiente virtual para não misturar com pacotes globais do Codespaces — que ocupa a primeira parte deste laboratório.

## Parte 1 - Empacotando a dependência como Layer

### Resultado esperado desta parte

Um diretório `layer/` contendo o `boto3` instalado no formato esperado pela Lambda, e o `serverless.yml`/`handler.py` já configurados para usá-lo.

<a id="passo-1"></a>

---

<dl>
<dt>

**1. Entre na pasta do exercício**

</dt>
<dd>

```bash
cd /workspaces/FIAP-Microservices-Serverless/02-Lambda/02-Layers/
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

Comando 100% local (sem tocar a AWS): copia o template `aws-python3` do repositório oficial de templates do Serverless Framework para a pasta atual, gerando `serverless.yml` e `handler.py`.

📚 Documentação oficial: [Serverless CLI reference - create](https://www.serverless.com/framework/docs/providers/aws/cli-reference/create) — lista os templates disponíveis além do `aws-python3` usado aqui.

</blockquote>
</details>

</dd>
</dl>

<a id="passo-3"></a>

---

<dl>
<dt>

**3. Declare a dependência da Layer**

</dt>
<dd>

Crie o arquivo `requirements.txt`:

```bash
code requirements.txt
```

Adicione a linha `boto3` e salve com **CTRL+S**, como na imagem:

![img/boto3.png](img/boto3.png)

</dd>
</dl>

<a id="passo-4"></a>

---

<dl>
<dt>

**4. Instale o `virtualenv` e crie um ambiente virtual**

</dt>
<dd>

```bash
pip3 install virtualenv && python3 -m venv ~/venv
```

O ambiente virtual isola os pacotes desta Layer do Python global do Codespaces — assim o pacote da Layer fica só com o que foi instalado explicitamente no passo 7.

</dd>
</dl>

<a id="passo-5"></a>

---

<dl>
<dt>

**5. Ative o ambiente virtual**

</dt>
<dd>

```bash
source ~/venv/bin/activate
```

O prompt do terminal passa a mostrar `(venv)` no início da linha — é o sinal de que os próximos comandos `pip3` instalam dentro deste ambiente isolado, não no Python global.

</dd>
</dl>

<a id="passo-6"></a>

---

<dl>
<dt>

**6. Crie o diretório da Layer**

</dt>
<dd>

```bash
mkdir layer
```

</dd>
</dl>

<a id="passo-7"></a>

---

<dl>
<dt>

**7. Instale a dependência dentro do diretório da Layer**

</dt>
<dd>

```bash
pip3 install -r requirements.txt -t layer
```

A flag `-t layer` instala o pacote listado em `requirements.txt` (o `boto3`) diretamente dentro de `layer/`, em vez do local padrão do ambiente virtual — é esse diretório que vai ser empacotado como Layer no deploy.

![img/pipinstall.png](img/pipinstall.png)

</dd>
</dl>

<a id="passo-8"></a>

---

<dl>
<dt>

**8. Ajuste o topo do `handler.py`**

</dt>
<dd>

```bash
code handler.py
```

Altere o início do arquivo conforme a imagem e salve com **CTRL+S**:

![img/topoarquivopython.png](img/topoarquivopython.png)

</dd>
</dl>

<a id="passo-9"></a>

---

<dl>
<dt>

**9. Configure a Layer no `serverless.yml`**

</dt>
<dd>

```bash
code serverless.yml
```

Substitua o conteúdo do arquivo pelo da imagem abaixo — é aqui que a Layer é declarada e associada à função — e salve com **CTRL+S**:

![img/yamllayers.png](img/yamllayers.png)

</dd>
</dl>

<details>
<summary><b>💡 Clique para entender: o <code>package.exclude</code> e o aviso de deprecação que vai aparecer</b></summary>
<blockquote>

O bloco `package.exclude` com `- layer/**` evita que o conteúdo da pasta `layer/` entre também no pacote da função: as dependências já vão para a AWS dentro da Layer, e incluí-las de novo no `.zip` da função dobraria o tamanho do deploy sem nenhum ganho.

Essa é a sintaxe antiga. A partir do Serverless Framework 2, ela foi substituída por `package.patterns`, onde a exclusão é escrita com `!` na frente do padrão:

```yaml
package:
  patterns:
    - '!layer/**'
```

As duas formas funcionam na versão 3 usada neste laboratório, mas o `exclude` imprime um aviso no terminal durante o `sls deploy`:

```
Support for "package.include" and "package.exclude" will be removed in the next
major release. Please use "package.patterns" instead
```

É só um aviso de deprecação — **o deploy conclui normalmente** e não há nada a corrigir para o laboratório funcionar. Se quiser deixar o arquivo na sintaxe atual, troque o bloco pelo `patterns` acima; o resultado é o mesmo.

📚 Documentação oficial: [Serverless Framework — package patterns](https://www.serverless.com/framework/docs/providers/aws/guide/packaging) — explica a sintaxe de `patterns` e a ordem em que os padrões são aplicados.

</blockquote>
</details>

### Checkpoint

- [ ] `layer/` contém o `boto3` instalado (confirme com `ls layer`).
- [ ] `serverless.yml` declara a Layer apontando para o diretório `layer` e a função referencia essa Layer.

## Parte 2 - Deploy, validação no console e teste

### Resultado esperado desta parte

Função e Layer publicadas na AWS, confirmadas visualmente no console e testadas por invocação — depois removidas.

<a id="passo-10"></a>

---

<dl>
<dt>

**10. Faça o deploy**

</dt>
<dd>

```bash
sls deploy
```

![img/slsdeploy.png](img/slsdeploy.png)

Este comando é **idempotente**: pode ser executado quantas vezes for necessário. Se ajustar `serverless.yml`, `handler.py` ou o conteúdo de `layer/` depois deste passo, basta rodar `sls deploy` novamente.

<details>
<summary>💡 Clique para entender: o que muda no deploy quando existe uma Layer</summary>
<blockquote>

Além de empacotar e enviar o código da função como no laboratório anterior, o `sls deploy` agora também empacota o conteúdo de `layer/` num `.zip` separado, publica esse pacote como uma nova versão de `AWS::Lambda::LayerVersion` e inclui o ARN dessa versão na configuração da função (`AWS::Lambda::Function`), tudo dentro da mesma stack do CloudFormation.

📚 Documentação oficial: [Lambda Layers - AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) — detalha a estrutura de diretórios esperada dentro do zip para cada runtime (no caso do Python, a pasta `python/`).

</blockquote>
</details>

</dd>
</dl>

<a id="passo-11"></a>

---

<dl>
<dt>

**11. Abra o console do Lambda**

</dt>
<dd>

No navegador, acesse o console AWS Lambda (funções) na região `us-east-1`:

<https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions>

</dd>
</dl>

<a id="passo-12"></a>

---

<dl>
<dt>

**12. Confirme a função criada e abra a aba "Camadas"**

</dt>
<dd>

Localize a função criada na lista e clique nela. Na página da função, clique em **Camadas** (Layers) para confirmar que a Layer criada no passo 9 aparece anexada:

![img/funcoescriadas.png](img/funcoescriadas.png)

</dd>
</dl>

<a id="passo-13"></a>

---

<dl>
<dt>

**13. Confirme a Layer no console de Layers**

</dt>
<dd>

Acesse o console de Layers (mesma região) e confirme que a Layer criada no deploy aparece listada, com a versão publicada:

![img/camadascriadas.png](img/camadascriadas.png)

</dd>
</dl>

<a id="passo-14"></a>

---

<dl>
<dt>

**14. Invoque a função**

</dt>
<dd>

```bash
sls invoke -f hello
```

![img/slsinvoke.png](img/slsinvoke.png)

<details>
<summary>💡 Clique para entender: por que a invocação só funciona com a Layer anexada</summary>
<blockquote>

Como o `boto3` não está mais empacotado junto com o código da função (só na Layer), o `import boto3` no `handler.py` só resolve em tempo de execução porque a Lambda monta o conteúdo da Layer em `/opt/python` — um dos caminhos que o runtime Python já inclui no `sys.path` por padrão. Sem a Layer anexada, esta mesma invocação falharia com `ModuleNotFoundError`.

📚 Documentação oficial: [Invoke - AWS Lambda API Reference](https://docs.aws.amazon.com/lambda/latest/api/API_Invoke.html) — parâmetro `InvocationType` (aqui, síncrono).

</blockquote>
</details>

</dd>
</dl>

<a id="passo-15"></a>

---

<dl>
<dt>

**15. Remova a infraestrutura criada**

</dt>
<dd>

```bash
sls remove
```

Comando seguro de repetir: se a stack já tiver sido removida, ele apenas confirma que não há nada para remover.

<details>
<summary>💡 Clique para entender: o que o <code>sls remove</code> apaga (e o que não apaga)</summary>
<blockquote>

O comando chama **`DeleteStack`** na stack do CloudFormation, removendo a função, a role de execução, o log group e a versão da Layer publicada por este deploy. O diretório local `layer/` e o ambiente virtual `~/venv` **não** são apagados por este comando — são artefatos locais, fora da stack.

📚 Documentação oficial: [Remove - Serverless CLI reference](https://www.serverless.com/framework/docs/providers/aws/cli-reference/remove).

</blockquote>
</details>

</dd>
</dl>

### Checkpoint

- [ ] A aba "Camadas" da função (passo 12) mostra a Layer anexada.
- [ ] `sls invoke -f hello` (passo 14) retorna sem `ModuleNotFoundError`.
- [ ] `sls remove` terminou sem erro e a stack não aparece mais no CloudFormation.

## Conclusão

Você separou uma dependência externa do código da função, empacotando-a como uma Lambda Layer isolada num ambiente virtual, declarou essa Layer no `serverless.yml`, confirmou no console que ela foi criada e anexada corretamente, e validou o resultado com uma invocação real — antes de remover tudo.

<details>
<summary>💡 Glossário rápido</summary>

| Termo | Significado |
|---|---|
| Layer | Pacote `.zip` versionado, anexado a uma função Lambda, separado do código dela |
| `virtualenv` / `venv` | Ambiente Python isolado, usado aqui para não misturar dependências da Layer com o Python global |
| `pip3 install -t <dir>` | Instala o pacote diretamente num diretório específico, em vez do local padrão |
| ARN | Identificador único de um recurso na AWS (aqui, da versão da Layer) |
| `/opt` | Caminho onde a Lambda monta o conteúdo de todas as Layers anexadas, em tempo de execução |

</details>

<details>
<summary>💡 Como pedir ajuda se travou</summary>

Antes de chamar o professor, tenha em mãos:

1. Em qual passo travou (o número, ex: "passo 9").
2. O comando exato que rodou.
3. A mensagem de erro completa (copie do terminal, não resuma).
4. Se o erro foi na invocação, o resultado de `ls layer` e a confirmação de que a Layer aparece anexada no console (passo 12).

Canais, em ordem de prioridade: sinalize em sala para o professor → grupo da turma → monitoria.

</details>
