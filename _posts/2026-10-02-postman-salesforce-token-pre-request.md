---
layout: post
title: "Postman para APIs Salesforce: autenticação automática com Pre-request Scripts"
date: 2026-10-02
author: André Carvalho
tags: [salesforce, postman, oauth, api]
description: "Como centralizar a obtenção e validação do token OAuth da Salesforce num único Pre-request Script de collection, eliminando o request de 'Get Token' e o copia-e-cola de tokens."
---

## Resumo

Quem trabalha com APIs da Salesforce no Postman conhece o ritual: um request separado só para gerar o token, copiar o `access_token`, colar no header de outro request, descobrir meia hora depois que o token expirou e repetir tudo. Multiplique isso por REST, GraphQL, Bulk API e Composite, e por três ambientes, e o tempo perdido deixa de ser pequeno.

Este artigo propõe um padrão simples: **um único Pre-request Script no nível da collection** que obtém o token via OAuth 2.0 Client Credentials, valida se ele ainda está ativo na própria Salesforce e só gera um novo quando necessário. Todos os requests herdam a autenticação. Nenhum request de "login" precisa ser executado manualmente.

---

## 1. O problema

A abordagem mais comum é ter, dentro da collection, um request chamado algo como `00 - Get Token`. O fluxo fica assim:

1. Executar `Get Token` manualmente.
2. Um script de teste salva o `access_token` numa variável.
3. Executar o request que realmente interessa.
4. Receber `401 INVALID_SESSION_ID` algum tempo depois.
5. Voltar ao passo 1.

Os problemas desse modelo:

- **Dependência de ordem de execução.** Quem abre a collection pela primeira vez não sabe que precisa rodar o request de token antes.
- **Expiração invisível.** O timeout de sessão é configurado na org e a sessão pode ser revogada a qualquer momento. O Postman não sabe disso.
- **Duplicação.** Cada pasta ou collection acaba com seu próprio request de token, cada um com uma variação diferente.
- **Runner frágil.** Para rodar a collection inteira, o request de token precisa ser o primeiro, o que acopla a ordem dos testes à autenticação.

## 2. Por que não usar o OAuth 2.0 nativo do Postman?

É a primeira pergunta razoável. O Postman tem um tipo de autorização **OAuth 2.0** com suporte a Client Credentials, e para muitos provedores ele resolve bem. Com a Salesforce, esbarra em três limitações:

- **Sem renovação automática.** O auto-refresh do Postman depende de um *refresh token*, e o fluxo Client Credentials da Salesforce não emite um. Quando o token expira, alguém precisa clicar em *Get New Access Token*.
- **Não detecta sessão revogada.** O helper não consulta a Salesforce para saber se o token ainda vale. Se a sessão cair antes do esperado, o primeiro sinal é um `401` no meio do trabalho.
- **O `instance_url` fica de fora.** A resposta do token da Salesforce traz a URL da instância, que é a base correta para as chamadas seguintes. O helper nativo não expõe esse valor de forma simples como variável.

O Pre-request Script resolve as três com poucas linhas de JavaScript, sem perder a configuração centralizada.

## 3. A proposta

Mover toda a responsabilidade de autenticação para o **Pre-request Script da collection**. O Postman executa esse script automaticamente antes de **cada** request da collection, então qualquer request, executado em qualquer ordem, sempre sai com um token válido.

```
Request disparado
      │
      ▼
Pre-request da collection
      │
      ├── Existe token e foi validado há pouco? ──► segue
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

1. **Validação na fonte, não por relógio.** Em vez de assumir que o token vale por X minutos, o script pergunta à Salesforce. Isso cobre sessões revogadas, políticas de timeout diferentes por org e tokens invalidados por mudanças no app.
2. **Janela de validação.** Chamar o `userinfo` antes de todo request dobraria o número de chamadas. O script só revalida se a última validação tiver mais de alguns minutos.
3. **Ambiente parametrizado.** O mesmo script serve para DEV, UAT e PRD. Uma única variável (`sf_env`) define de onde vêm as credenciais.

## 4. Pré-requisitos na Salesforce

O fluxo Client Credentials exige um app de integração configurado na org:

- Um **External Client App** (ou Connected App) com OAuth habilitado e o fluxo **Client Credentials** ativo.
- Um usuário definido em **Run As**. Toda chamada feita com o token roda com as permissões desse usuário, então use um usuário de integração com o mínimo de acesso necessário.
- Os scopes adequados para o que a collection vai consumir (no mínimo `api`).

O Client Credentials Flow **exige a URL do My Domain** da org. Chamar `login.salesforce.com` ou `test.salesforce.com` retorna erro (veja a seção 8).

## 5. Variáveis de ambiente

Cada environment do Postman define o prefixo do ambiente e as credenciais correspondentes:

| Variável | Exemplo | Observação |
|---|---|---|
| `sf_env` | `uat` | Prefixo usado para buscar as demais |
| `uat_url` | `https://suaorg--uat.sandbox.my.salesforce.com` | My Domain da org |
| `uat_key` | | Consumer Key |
| `uat_sec` | | Consumer Secret |
| `Version` | `v62.0` | Versão da API |

O script cria e mantém sozinho estas três:

| Variável | Conteúdo |
|---|---|
| `sf_access_token` | Token atual |
| `sf_instance_url` | URL da instância retornada pela Salesforce |
| `sf_token_checked_at` | Timestamp da última validação |

Para adicionar PRD, basta um novo environment com `sf_env = prd` e as variáveis `prd_url`, `prd_key` e `prd_sec`. O script não muda.

## 6. O script

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

### Por que a janela de validação

Para uso interativo, 5 minutos é confortável: o token é revalidado poucas vezes por hora. Para o Collection Runner com centenas de requests, uma janela maior reduz ainda mais as chamadas ao `userinfo`. Se o token cair dentro da janela, o request recebe `401` e a próxima revalidação gera um novo.

### Trocando de ambiente

O token fica salvo no próprio environment. Trocar de UAT para PRD no seletor do Postman troca junto o `sf_access_token` e o `sf_instance_url`, sem risco de mandar um token de UAT para PRD.

## 7. Configurando os requests

Com o script na collection, os requests ficam limpos.

**Na collection**, aba *Authorization*:

- Tipo: **Bearer Token**
- Token: `{{sf_access_token}}`
- *Apply Auth to*: o domínio da org, por exemplo `*.my.salesforce.com/*`. Esse campo impede que o token seja enviado se algum request apontar para um host fora da Salesforce.

**Em cada request**, aba *Authorization*: **Inherit auth from parent**. Se a opção não aparecer, o request está solto, fora de uma collection ou pasta.

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

## 8. Troubleshooting

Como o script repassa o `error` e o `error_description` da Salesforce, a mensagem aparece direto no Console do Postman. Os casos mais comuns:

| Erro | Causa provável | Como resolver |
|---|---|---|
| `unsupported_grant_type` / *request not supported on this domain* | Chamada para `login.salesforce.com` ou `test.salesforce.com` | Usar a URL do My Domain em `{env}_url` |
| `invalid_grant` / *no client credentials user enabled* | Usuário *Run As* não definido no app | Configurar o *Run As* nas políticas do app |
| `invalid_client_id` | Consumer Key errada ou de outra org | Conferir a chave e se o app pertence à org de `{env}_url` |
| `invalid_client` / *invalid client credentials* | Consumer Secret errado, ou app recém-criado ainda propagando | Conferir o secret; em apps novos, aguardar alguns minutos |
| `401 INVALID_SESSION_ID` na chamada da API | Token caiu dentro da janela de validação | Reexecutar; a próxima revalidação gera um token novo |
| Erro "Defina uat_url, uat_key..." | Environment errado selecionado ou `sf_env` com outro prefixo | Conferir o environment ativo e o valor de `sf_env` |

## 9. Antes e depois

| | Com `00 - Get Token` | Com Pre-request na collection |
|---|---|---|
| Primeiro uso | Descobrir que precisa rodar o request de token | Só executar o request desejado |
| Token expirou | `401`, rodar o token de novo, repetir o request | Transparente |
| Sessão revogada | `401` sem explicação | Detectado na revalidação, novo token gerado |
| Trocar de ambiente | Rodar o token no novo ambiente | Trocar o environment no seletor |
| Collection Runner | Request de token obrigatoriamente primeiro | Qualquer ordem, qualquer subconjunto |
| Novo request | Configurar header de auth | *Inherit auth from parent* |

## 10. Boas práticas

- **Script na collection, nunca no request.** Script no request reintroduz a duplicação que este padrão quer eliminar.
- **Usuário de integração com menor privilégio.** O token herda todas as permissões do usuário *Run As*. Um token vazado de um admin é um incidente; de um usuário restrito, é um incômodo.
- **Credenciais fora de qualquer arquivo compartilhado.** Ao exportar ou versionar a collection, mantenha `{env}_key`, `{env}_sec` e `sf_access_token` com valores vazios, ou use o Postman Vault.
- **Sempre `sf_instance_url` nas URLs.** Ela vem da própria Salesforce e evita chamar um host diferente daquele que emitiu o token.
- **Teste `errors` em GraphQL.** Status 200 não significa sucesso.

## Conclusão

Centralizar a autenticação num Pre-request Script de collection resolve um problema pequeno que se repete o dia inteiro. O ganho não está só nos segundos economizados por request: está em ter uma collection que **funciona na primeira execução**, em qualquer ordem e em qualquer ambiente, na máquina de qualquer pessoa do time.

No próximo post, essa mesma collection vai para o Git e entra no pipeline do GitHub Actions como smoke test das APIs depois de cada deploy na Salesforce.
