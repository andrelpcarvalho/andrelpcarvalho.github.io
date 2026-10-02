---
layout: post
title: "Postman para APIs Salesforce: autenticação automática com Pre-request Scripts"
date: 2026-10-02
author: André Carvalho
tags: [salesforce, postman, oauth, devops, api]
description: "Como centralizar a obtenção e validação do token OAuth da Salesforce num único Pre-request Script de collection, eliminando o copia-e-cola de tokens entre requests."
---

## Resumo

Quem trabalha com APIs da Salesforce no Postman conhece o ritual: um request separado só para gerar o token, copiar o `access_token`, colar no header de outro request, descobrir meia hora depois que o token expirou e repetir tudo. Multiplique isso por REST, GraphQL, Bulk API, Composite e por três ambientes (DEV, UAT, PRD) e o tempo perdido deixa de ser pequeno.

Este artigo propõe um padrão simples: **um único Pre-request Script no nível da collection** que obtém o token via OAuth 2.0 Client Credentials, valida se ele ainda está ativo na própria Salesforce e só gera um novo quando necessário. Todos os requests da collection herdam a autenticação. Nenhum request de "login" precisa ser executado manualmente.

No final, a collection é versionada no Git com o Native Git do Postman e executada em CI com o Postman CLI, sem nenhuma credencial no repositório.

---

## 1. O problema

A abordagem mais comum em times que consomem APIs Salesforce é ter, dentro da collection, um request chamado algo como `00 - Get Token`. O fluxo fica assim:

1. Executar `Get Token` manualmente.
2. Um script de teste salva o `access_token` numa variável.
3. Executar o request que realmente interessa.
4. Receber `401 INVALID_SESSION_ID` algum tempo depois.
5. Voltar ao passo 1.

Os problemas desse modelo:

- **Dependência de ordem de execução.** Quem abre a collection pela primeira vez não sabe que precisa rodar o request de token antes.
- **Expiração invisível.** O timeout de sessão é configurado na org (Session Settings) e pode ser revogado a qualquer momento. O Postman não sabe disso.
- **Duplicação.** Cada pasta ou collection acaba com seu próprio request de token, cada um com uma variação diferente.
- **Atrito em CI.** No pipeline, é preciso garantir que o request de token rode primeiro, o que acopla a ordem dos testes à autenticação.

## 2. A proposta

Mover toda a responsabilidade de autenticação para o **Pre-request Script da collection**. O Postman executa esse script automaticamente antes de **cada** request da collection, então qualquer request, executado em qualquer ordem, sempre sai com um token válido.

```
Request disparado
      │
      ▼
Pre-request da collection
      │
      ├── Existe token em cache e foi validado há pouco? ──► segue
      │
      ├── Existe token? ──► GET /services/oauth2/userinfo
      │                         ├── 200 ──► marca validado, segue
      │                         └── 401 ──► gera novo token
      │
      └── Não existe token ──► POST /services/oauth2/token
                                     └── salva token e instance_url
      │
      ▼
Request sai com Authorization: Bearer {{sf_access_token}}
```

Três decisões de design sustentam esse fluxo:

1. **Validação na fonte, não por relógio.** Em vez de assumir que o token vale por X minutos, o script pergunta à Salesforce. Isso cobre sessões revogadas, políticas de timeout diferentes por org e tokens invalidados por troca de senha ou de configuração do app.
2. **Janela de validação.** Chamar o `userinfo` antes de todo request dobraria o número de chamadas. O script só revalida se a última validação tiver mais de alguns minutos. Dentro da janela, confia no cache.
3. **Ambiente parametrizado.** O mesmo script serve para DEV, UAT e PRD. Uma única variável (`sf_env`) define de onde vêm as credenciais.

## 3. Pré-requisitos na Salesforce

O fluxo Client Credentials exige um app de integração configurado na org:

- Um **External Client App** (ou Connected App) com OAuth habilitado e o fluxo **Client Credentials** ativo.
- Um usuário definido em **Run As**. Toda chamada feita com o token roda com as permissões desse usuário, então use um usuário de integração com o mínimo de acesso necessário.
- Os scopes adequados para o que a collection vai consumir (no mínimo `api`).

Um detalhe que costuma custar tempo: o Client Credentials Flow **exige a URL do My Domain** da org. Chamar `login.salesforce.com` ou `test.salesforce.com` retorna erro.

## 4. Variáveis de ambiente

Cada environment do Postman define o prefixo do ambiente e as credenciais correspondentes:

| Variável | Exemplo | Observação |
|---|---|---|
| `sf_env` | `uat` | Prefixo usado para buscar as demais |
| `uat_url` | `https://suaorg--uat.sandbox.my.salesforce.com` | My Domain da org |
| `uat_key` | *(vazio no repositório)* | Consumer Key |
| `uat_sec` | *(vazio no repositório)* | Consumer Secret |
| `Version` | `v62.0` | Versão da API |

O script cria e mantém sozinho estas três:

| Variável | Conteúdo |
|---|---|
| `sf_access_token` | Token atual |
| `sf_instance_url` | URL da instância retornada pela Salesforce |
| `sf_token_checked_at` | Timestamp da última validação |

Para adicionar PRD, basta um novo environment com `sf_env = prd` e as variáveis `prd_url`, `prd_key` e `prd_sec`. O script não muda.

## 5. O script

Cole no Pre-request Script da **collection** (não do request):

```javascript
/**
 * Salesforce OAuth 2.0 - Client Credentials
 * Obtém, reaproveita e valida o token antes de cada request da collection.
 */

const VALIDATION_WINDOW_MS = 5 * 60 * 1000; // revalida no máximo a cada 5 min

const env    = pm.environment.get("sf_env") || "uat";
const url    = pm.environment.get(`${env}_url`);
const key    = pm.environment.get(`${env}_key`);
const secret = pm.environment.get(`${env}_sec`);

if (!url || !key || !secret) {
    throw new Error(`Defina ${env}_url, ${env}_key e ${env}_sec no environment.`);
}

async function tokenValido(token, instanceUrl) {
    const res = await pm.sendRequest({
        url: `${instanceUrl}/services/oauth2/userinfo`,
        method: "GET",
        header: { Authorization: `Bearer ${token}` }
    });
    return res.code === 200;
}

async function novoToken() {
    const res = await pm.sendRequest({
        url: `${url}/services/oauth2/token`,
        method: "POST",
        header: { "Content-Type": "application/x-www-form-urlencoded" },
        body: {
            mode: "urlencoded",
            urlencoded: [
                { key: "grant_type",    value: "client_credentials" },
                { key: "client_id",     value: key },
                { key: "client_secret", value: secret }
            ]
        }
    });

    const body = res.json();
    if (res.code !== 200) {
        throw new Error(`[${env}] Falha no token: ${body.error} - ${body.error_description}`);
    }

    pm.environment.set("sf_access_token", body.access_token);
    pm.environment.set("sf_instance_url", body.instance_url);
    pm.environment.set("sf_token_checked_at", Date.now());
    console.log(`[${env}] Novo token gerado.`);
}

async function garantirToken() {
    const token       = pm.environment.get("sf_access_token");
    const instanceUrl = pm.environment.get("sf_instance_url");
    const checkedAt   = Number(pm.environment.get("sf_token_checked_at") || 0);

    if (!token || !instanceUrl) {
        return novoToken();
    }

    if (Date.now() - checkedAt < VALIDATION_WINDOW_MS) {
        return; // validado recentemente, confia no cache
    }

    if (await tokenValido(token, instanceUrl)) {
        pm.environment.set("sf_token_checked_at", Date.now());
        console.log(`[${env}] Token válido, reaproveitando.`);
        return;
    }

    console.log(`[${env}] Token expirado ou revogado.`);
    return novoToken();
}

await garantirToken();
```

### Por que `async/await`

`pm.sendRequest` com callback não bloqueia o request principal: o Postman pode disparar a chamada antes de o token chegar. Com `await` (suportado a partir do Postman v11), o request só sai depois que a autenticação terminou.

### Por que o `userinfo`

É o endpoint mais barato para checar um token: responde rápido, retorna `401` quando a sessão não é mais válida e não depende de nenhum objeto ou permissão específica. Funciona como um "ping autenticado".

### Trocando ambiente

Como o token fica salvo no environment, trocar de UAT para PRD no seletor do Postman troca automaticamente o conjunto `sf_access_token` / `sf_instance_url`. Não há risco de mandar um token de UAT para PRD.

## 6. Configurando os requests

Com o script na collection, os requests ficam limpos.

**Na collection**, aba *Authorization*:

- Tipo: **Bearer Token**
- Token: `{{sf_access_token}}`
- *Apply Auth to*: o domínio da org, por exemplo `*.my.salesforce.com/*`. Esse campo impede que o token seja enviado se algum request apontar para um host fora da Salesforce.

**Em cada request**, aba *Authorization*: **Inherit auth from parent**.

**URLs** sempre a partir da instância retornada pela Salesforce:

```
{{sf_instance_url}}/services/data/{{Version}}/sobjects/Account/describe
```

### Exemplo: GraphQL

```
POST {{sf_instance_url}}/services/data/{{Version}}/graphql
```

Body do tipo GraphQL:

```graphql
query buscarCaso($numeroCaso: String) {
  uiapi {
    query {
      Case(where: { CaseNumber: { eq: $numeroCaso } }) {
        edges {
          node {
            Id
            CaseNumber { value }
            Status { value }
            Subject { value }
          }
        }
      }
    }
  }
}
```

Variables:

```json
{ "numeroCaso": "{{numeroCaso}}" }
```

Um teste importante no *Post-response*: o endpoint GraphQL da Salesforce responde **HTTP 200 mesmo quando a query falha**, por exemplo com um campo inexistente ou sem permissão de leitura. O erro vem dentro do corpo:

```javascript
const body = pm.response.json();
pm.test("Sem erros GraphQL", () =>
    pm.expect(body.errors, JSON.stringify(body.errors)).to.be.undefined
);
```

Sem esse teste, um request quebrado passa como verde no runner e no CI.

## 7. Versionando a collection

Com o Native Git do Postman (v12+), a collection deixa de viver só na nuvem e passa a ser um conjunto de arquivos YAML dentro do repositório. Ao conectar o workspace a uma pasta, o Postman cria:

| Pasta | Conteúdo | Versionar |
|---|---|---|
| `postman/` | Collections e environments em YAML | Sim |
| `.postman/resources.yaml` | Mapa entre arquivos locais e IDs no Postman Cloud | Sim |

O fluxo de trabalho vira o mesmo do código:

```bash
git checkout -b feature/novo-endpoint
# edita a collection no Postman, em Local View
git add postman/ .postman/
git commit -m "Adiciona request de consulta de Case"
git push -u origin feature/novo-endpoint
```

O Pre-request Script vai junto no arquivo de definição da collection, então quem clonar o repositório já recebe a autenticação pronta.

### Credenciais fora do repositório

Os environments também viram arquivos. A regra é: **as chaves vão para o Git, os valores não.** Antes de cada commit:

```bash
grep -rnE "_key|_sec|sf_access_token" postman/environments/
```

Se aparecer qualquer valor preenchido, limpe no environment ou mova o segredo para o **Postman Vault** antes do `git add`.

## 8. Executando em CI

O Postman CLI roda a mesma collection no pipeline. As credenciais entram como secrets do CI, injetadas em tempo de execução:

```yaml
name: Salesforce API Tests

on:
  pull_request:
    paths: ["postman/**"]

jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Instalar Postman CLI
        run: curl -o- "https://dl-cli.pstmn.io/install/linux64.sh" | sh

      - name: Rodar collection em UAT
        env:
          POSTMAN_API_KEY: ${{ secrets.POSTMAN_API_KEY }}
          UAT_KEY: ${{ secrets.SF_UAT_KEY }}
          UAT_SEC: ${{ secrets.SF_UAT_SEC }}
        run: |
          postman login --with-api-key "$POSTMAN_API_KEY"
          postman collection run postman/collections/Salesforce \
            -e postman/environments/UAT.yaml \
            --env-var "uat_key=$UAT_KEY" \
            --env-var "uat_sec=$UAT_SEC"
```

Repare que o pipeline **não tem nenhum passo de autenticação na Salesforce**. O Pre-request Script cuida disso na primeira chamada, exatamente como na máquina do desenvolvedor.

> O Newman não é compatível com o formato v3 de collections usado pelo Native Git. Pipelines antigos que usam `newman run` precisam migrar para o Postman CLI.

## 9. Boas práticas e armadilhas

- **Script na collection, nunca no request.** Script no request reintroduz a duplicação que este padrão quer eliminar.
- **Usuário de integração com menor privilégio.** O token herda todas as permissões do usuário *Run As*. Um token vazado de um admin é um incidente; de um usuário restrito, é um incômodo.
- **My Domain sempre.** `test.salesforce.com` não funciona com Client Credentials.
- **Não edite em Cloud View.** Com Native Git, alterações feitas direto na nuvem são sobrescritas no próximo `postman workspace push`.
- **Ajuste a janela de validação ao uso.** Para uso interativo, 5 minutos é confortável. Para runners com centenas de requests, uma janela maior reduz chamadas ao `userinfo`; o `401` ainda é tratado na próxima revalidação.
- **Teste `errors` em GraphQL.** Status 200 não significa sucesso.

## 10. Conclusão

Centralizar a autenticação num Pre-request Script de collection resolve um problema pequeno que se repete o dia inteiro. O ganho não está só nos segundos economizados por request: está em ter uma collection que **funciona na primeira execução**, em qualquer ordem, em qualquer ambiente, na máquina de qualquer pessoa do time e no pipeline, sem nenhuma instrução extra.

Combinado ao Native Git, o resultado é uma collection tratada como código: versionada, revisada em PR, testada em CI e sem nenhum segredo no repositório.

---

*Código completo e collection de exemplo: [github.com/andrelpcarvalho/salesforcce-collections](https://github.com/andrelpcarvalho/salesforcce-collections)*
