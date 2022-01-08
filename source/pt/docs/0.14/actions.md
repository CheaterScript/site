title: Ações
---

Ações são os métodos de um serviço que podem ser chamados externamente. As ações são chamadas via RPC (Remote Procedure Call). Elas são parecidas com requisições HTTP, com parâmetros de requisição e respostas de retorno.

Se você possuir várias instâncias de um serviço, o broker irá balancear as requisições entre as instâncias. [Leia mais sobre balanceamento](balancing.html).

<div align="center">
    <img src="assets/action-balancing.gif" alt="Diagrama de balanceamento de ações" />
</div>

## Chamada de serviços
Para chamar um serviço utilize o método `broker.call`. O broker procura o serviço (e um nó) que possui a ação especificada e chama-a. A função retorna uma `Promise`.

### Sintaxe
```js
const res = await broker.call(actionName, params, opts);
```
O parâmetro `actionName` é uma string separada por pontos. Sua primeira parte é o nome do serviço, enquanto a segunda parte representa o nome da ação. Então, se você tiver um serviço de `posts` com uma ação `create`, você pode chamá-la utilizando `posts.create`.

O parâmetro `params` é um objeto que é passado para a ação como parte do [Contexto](context.html). O serviço pode acessá-lo através de `ctx.params`. *É opcional. Caso não seja definido, seu valor padrão é `{}`*.

O parâmetro `opts` é um objeto utilizado para definir/substituir alguns parâmetros da solicitação, por exemplo: `timeout`, `retryCount`. *É opcional.*

**Opções de chamada disponíveis:**

| Nome               | Tipo      | Valor padrão | Descrição                                                                                                                                                                                                                                                                                                                                  |
| ------------------ | --------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `timeout`          | `Number`  | `null`       | Tempo limite da requisição em milissegundos. Se a requisição atingir o tempo limite e você não definir `fallbackResponse`, o broker lançará um erro de `RequestTimeout`. Defina como `0` para desativar. Quando não definido, assumirá o `requestTimeout` definido nas configurações do broker. [Leia mais](fault-tolerance.html#Timeout). |
| `retries`          | `Number`  | `null`       | Número de tentativas. Se a requisição atingir seu tempo limite, o broker irá tentar chamá-la novamente. Defina como `0` para desativar. Quando não definido, assumirá o `retryPolicy.retries` definido nas configurações do broker. [Leia mais](fault-tolerance.html#Retry).                                                               |
| `fallbackResponse` | `Any`     | `null`       | Se a solicitação falhar, essa resposta alternativa será retornada. [Leia mais](fault-tolerance.html#Fallback).                                                                                                                                                                                                                             |
| `nodeID`           | `String`  | `null`       | ID do nó de destino. Se definido, fará uma chamada direta para o nó especificado.                                                                                                                                                                                                                                                          |
| `meta`             | `Objeto`  | `{}`         | Metadados da requisição. Acesse-o utilizando ` ctx.meta ` nas ações. Será transferido & mesclado em chamadas encadeadas.                                                                                                                                                                                                                   |
| `parentCtx`        | `Context` | `null`       | Instância de contexto pai ` Context `. Use-o para encadear as chamadas.                                                                                                                                                                                                                                                                    |
| `requestID`        | `String`  | `null`       | ID de Requisição ou Correlação. Use-o para rastreamento.                                                                                                                                                                                                                                                                                   |


### Utilização
**Chamada sem parâmetros**
```js
const res = await broker.call("user.list");
```

**Chamada com parâmetros**
```js
const res = await broker.call("user.get", { id: 3 });
```

**Chamada com opções de chamada**
```js
const res = await broker.call("user.recommendation", { limit: 5 }, {
    timeout: 500,
    retries: 3,
    fallbackResponse: defaultRecommendation
});
```

**Chamada com tratamento de erros**
```js
broker.call("posts.update", { id: 2, title: "Modified post title" })
    .then(res => console.log("Post updated!"))
    .catch(err => console.error("Unable to update Post!", err));    
```

**Chamada direta: obtenha informações de saúde do nó "node-21"**
```js
const res = await broker.call("$node.health", null, { nodeID: "node-21" })
```

### Metadados
Envie metadados para serviços com a propriedade `meta`. Acesse-as utilizando ` ctx.meta ` nas ações. Observe que em chamadas aninhadas o `meta` é mesclado.
```js
broker.createService({
    name: "test",
    actions: {
        first(ctx) {
            return ctx.call("test.second", null, { meta: {
                b: 5
            }});
        },
        second(ctx) {
            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        }
    }
});

broker.call("test.first", null, { meta: {
    a: "John"
}});
```

` meta ` é enviada de volta ao serviço que fez a chamada do método. Você pode utilizar para enviar metadados extras de volta ao remetente da ação. Ex.: enviar cabeçalhos de resposta de volta para o API gateway ou gravar dados do usuário conectado nos metadados.

```js
broker.createService({
    name: "test",
    actions: {
        async first(ctx) {
            await ctx.call("test.second", null, { meta: {
                a: "John"
            }});

            console.log(ctx.meta);
            // Prints: { a: "John", b: 5 }
        },
        second(ctx) {
            // Modify meta
            ctx.meta.b = 5;
        }
    }
});
```

Ao chamar ações dentro do serviço (`this.actions.xy()`), você deve definir o campo ` parentCtx` de `meta` para transmitir dados.

**Chamadas internas**
```js
broker.createService({
  name: "mod",
  actions: {
    hello(ctx) {
      console.log(ctx.meta);
      // Prints: { user: 'John' }
      ctx.meta.age = 123
      return this.actions.subHello(ctx.params, { parentCtx: ctx });
    },

    subHello(ctx) {
      console.log("meta from subHello:", ctx.meta);
      // Prints: { user: 'John', age: 123 }
      return "hi!";
    }
  }
});

broker.call("mod.hello", { param: 1 }, { meta: { user: "John" } });
```

### Timeout

O tempo limite (timeout) também pode ser definido na definição de uma ação. Ele substitui a [opção `requestTimeout`](fault-tolerance.html#Timeout) do gerenciador de serviços (broker) global, mas não substitui o `timeout` nas opções de chamada.

**Exemplo**
 ```js
// moleculer.config.js
module.exports = {
    nodeID: "node-1",
    requestTimeout: 3000
};

 // greeter.service.js
module.exports = {
    name: "greeter",
    actions: {
        normal: {
            handler(ctx) {
                return "Normal";
            }
        },
         slow: {
            timeout: 5000, // 5 secs
            handler(ctx) {
                return "Slow";
            }
        }
    },
```
**Exemplos de chamada**
```js
// Utiliza o timeout global como 5000
await broker.call("greeter.normal");
 // Utiliza o timeout de 3000 definido na ação 
await broker.call("greeter.slow");
 // Utiliza o timeout de 1000 passado nas opções da chamada
await broker.call("greeter.slow", null, { timeout: 1000 });
```
### Chamadas múltiplas

Chamar várias ações ao mesmo tempo também é possível. Para fazer isso, utilize `broker.mcall` ou `ctx.mcall`.
**`mcall` com Objeto Array<Object\\>**

```js
await broker.mcall(
    [
        { action: 'posts.find', params: { author: 1 }, opções: { /* Opções da chamada. */} },
        { action: 'users.find', params: { name: 'John' } }
    ],
    {
        // Opções comuns para todas as chamadas.
        meta: { token: '63f20c2d-8902-4d86-ad87-b58c9e2333c2' }
    }
);
```

**`mcall` com Objeto e options.meta**
```js
await broker.mcall(
    {
        posts: { action: 'posts.find', params: { author: 1 }, options: { /* Opções da chamada. */} },
        users: { action: 'users.find', params: { name: 'John' } }
    }, 
    {
        // Opções comuns para todas as chamadas.
        meta: { token: '63f20c2d-8902-4d86-ad87-b58c9e2333c2' }
    }
);
```

**opção `settled` em `broker.mcall`**

O método `mcall` possui uma nova opção `settled` para receber todas os resultados de Promises. Se `settled: true`, `mcall` retorna uma Promise resolvida para todos os casos e o response contém os status e as respostas de todas as chamadas. Note que, sem esta opção você não saberá quantas (nem quais) chamadas foram rejeitadas.

Exemplo
```js
const res = await broker.mcall([
    { action: "posts.find", params: { limit: 2, offset: 0 },
    { action: "users.find", params: { limit: 2, sort: "username" } },
    { action: "service.notfound", params: { notfound: 1 } }
], { settled: true });
console.log(res);
```

`res` será algo parecido a

```js
[
    { status: "fulfilled", value: [/*... resposta de `posts.find`...*/] },
    { status: "fulfilled", value: [/*... resposta of `users.find`...*/] },
    { status: "rejected", reason: {/*... Resposta rejeitada/Erro`...*/} }
]
```

## Streaming
O Moleculer suporta os streams do Node.js nos parâmetros da requisição `params` e nas respostas. Utilize streams para transferir um arquivo recebido de um gateway, codificar/decodificar ou compactar/descompactar streams.

### Exemplos

**Enviar um arquivo como stream para um serviço**
```js
const stream = fs.createReadStream(fileName);

broker.call("storage.save", stream, { meta: { filename: "avatar-123.jpg" }});
```

{% note info Object Mode Streaming%}
Também há suporte para [Object Mode Streaming](https://nodejs.org/api/stream.html#stream_object_mode). Para ativar, defina `$streamObjectMode: true` no [`meta`](actions.html#Metadata).
{% endnote %}

Note que `params` deve ser um stream, você não pode adicionar nenhuma variável adicional a `params`. Use a propriedade `meta` para transferir dados adicionais.

**Recebendo um stream em um serviço**
```js
module.exports = {
    name: "storage",
    actions: {
        save(ctx) {
            // Save the received stream to a file
            const s = fs.createWriteStream(`/tmp/${ctx.meta.filename}`);
            ctx.params.pipe(s);
        }
    }
};
```

**Retornar um stream como resposta em um serviço**
```js
module.exports = {
    name: "storage",
    actions: {
        get: {
            params: {
                filename: "string"
            },
            handler(ctx) {
                return fs.createReadStream(`/tmp/${ctx.params.filename}`);
            }
        }
    }
};
```

**Processando o fluxo recebido no lado do remetente**
```js
const filename = "avatar-123.jpg";
broker.call("storage.get", { filename })
    .then(stream => {
        const s = fs.createWriteStream(`./${filename}`);
        stream.pipe(s);
        s.on("close", () => broker.logger.info("File has been received"));
    })
```

**Exemplo de serviço de codificação/decodificação AES**
```js
const crypto = require("crypto");
const password = "moleculer";

module.exports = {
    name: "aes",
    actions: {
        encrypt(ctx) {
            const encrypt = crypto.createCipher("aes-256-ctr", password);
            return ctx.params.pipe(encrypt);
        },

        decrypt(ctx) {
            const decrypt = crypto.createDecipher("aes-256-ctr", password);
            return ctx.params.pipe(decrypt);
        }
    }
};
```

## Visibilidade das ações
A ação tem uma propriedade `visibility` para controlar sua visibilidade e a possibilidade de chamá-la por outros serviços.

**Valores disponíveis:**
- `published` ou `null`: ação pública. Pode ser chamada localmente, remotamente, e pode ser publicada através da API Gateway
- `public`: ação pública, pode ser chamada localmente & remotamente, mas não publicada via API Gateway
- `protected`: só pode ser chamado localmente (de serviços locais)
- `private`: só pode ser chamado internamente (através de `this.actions.xy()` dentro do serviço)

**Alterar visibilidade**
```js
module.exports = {
    name: "posts",
    actions: {
        // É públicado por padrão
        find(ctx) {},
        clean: {
            // Chamado apenas via `this.actions.clean`
            visibility: "private",
            handler(ctx) {}
        }
    },
    methods: {
        cleanEntities() {
            // Chama a ação diretamente
            return this.actions.clean();
        }
    }
}
```

> O valor padrão é `null` (que será considerado como `published`) devido à compatibilidade com versões anteriores.

## Hooks de ação
Os hooks de ação são funções de middleware conectáveis e reutilizáveis que podem ser registradas em `before`, `after` ou em `errors` nas ações de serviço. Um hook é uma `função` ou uma `String`. Em caso de `String`, seu nome deve corresponder ao nome do [método](services.html#Methods) do serviço.

### Before Hooks
Os before hooks são executados antes de uma ação ocorrer. Recebem `ctx` e podem manipular `ctx.params`, `ctx.meta`, ou adicionar variáveis personalizadas em `ctx.locals` para ser utilizadas nos handlers das ações. Se houver algum problema, o hook pode disparar um `Error`. _Por favor, observe que você não pode quebrar/ignorar as futuras execuções de hooks ou handlers da ação._

**Principais usos:**
- tratamentos de parâmetros
- validações de parâmetros
- pesquisar entidades
- autorização

### After hooks
Os after hooks são executados após uma ação ocorrer. Recebem `ctx` e `response`. Eles podem manipular ou alterar completamente a resposta da ação. A resposta da ação deve sempre ser retornada no hook.

**Principais usos:**
- preencher propriedades
- remover dados confidenciais.
- envolver a resposta em um `Object`
- converter a estrutura da resposta

### Error hooks
Os hooks de erro são chamados quando um `Erro` é lançado durante a chamada da ação. Recebe `ctx` e `err`. Pode lidar com o erro e retornar uma resposta (fallback) ou lançar o erro.

**Principais usos:**
- manipulação de erros
- encapsular o erro em outro
- resposta de fallback

### Declaração de nível de serviço
Hooks can be assigned to a specific action (by indicating action `name`), all actions (`*`) in service or by indicating a wildcard (e.g., `create-*`). The latter will be applied to all actions whose name starts with `create-`.

{% note warn%}
Note que a ordem de registro do hook importa, pois define a sequência pela qual os hooks são executados. Para obter mais informações, dê uma olhada em [ordem de execução dos hooks](#Execution-order).
{% endnote %}

**Before Hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "posts",
    mixins: [DbService]
    hooks: {
        before: {
            // Define um hook global para todas as ações
            // O hook chamará o método `resolveLoggedUser`.
            "*": "resolveLoggedUser",

            // Define multiple hooks for action `remove`
            remove: [
                function isAuthenticated(ctx) {
                    if (!ctx.user)
                        throw new Error("Forbidden");
                },
                function isOwner(ctx) {
                    if (!this.checkOwner(ctx.params.id, ctx.user.id))
                        throw new Error("Only owner can remove it.");
                }
            ],
            // Applies to all actions that start with "create-"
            "create-*": [
                async function (ctx){}
            ],
            // Applies to all actions that end with "-user"
            "*-user": [
                async function (ctx){}
            ],
        }
    },

    methods: {
        async resolveLoggedUser(ctx) {
            if (ctx.meta.user)
                ctx.user = await ctx.call("users.get", { id: ctx.meta.user.id });
        }
    }
}
```

**After & Error Hooks**

```js
const DbService = require("moleculer-db");

module.exports = {
    name: "users",
    mixins: [DbService]
    hooks: {
        after: {
            // Define a global hook for all actions to remove sensitive data
            "*": function(ctx, res) {
                // Remove password
                delete res.password;

                // Please note, must return result (either the original or a new)
                return res;
            },
            get: [
                // Add a new virtual field to the entity
                async function (ctx, res) {
                    res.friends = await ctx.call("friends.count", { query: { follower: res._id }});

                    return res;
                },
                // Populate the `referrer` field
                async function (ctx, res) {
                    if (res.referrer)
                        res.referrer = await ctx.call("users.get", { id: res._id });

                    return res;
                }
            ],
            // Applies to all actions that start with "create-"
            "create-*": [
                async function (ctx, res){}
            ],
            // Applies to all actions that end with "-user"
            "*-user": [
                async function (ctx, res){}
            ],
        },
        error: {
            // Global error handler
            "*": function(ctx, err) {
                this.logger.error(`Error occurred when '${ctx.action.name}' action was called`, err);

                // Throw further the error
                throw err;
            },
            // Applies to all actions that start with "create-"
            "create-*": [
                async function (ctx, err){}
            ],
            // Applies to all actions that end with "-user"
            "*-user": [
                async function (ctx, err){}
            ],
        }
    }
};
```

### Declaração a nível de ação
Hooks também podem ser registrados dentro da declaração da ação.

{% note warn%}
Observe que a ordem de registro do hook importa, pois define a sequência pela qual os hooks são executados. Para obter mais informações, dê uma olhada em [ordem de execução dos hooks](#Execution-order).
{% endnote %}

**Before & After hooks**

```js
broker.createService({
    name: "greeter",
    actions: {
        hello: {
            hooks: {
                before(ctx) {
                    broker.logger.info("Before action hook");
                },
                after(ctx, res) {
                    broker.logger.info("After action hook"));
                    return res;
                }
            },

            handler(ctx) {
                broker.logger.info("Action handler");
                return `Hello ${ctx.params.name}`;
            }
        }
    }
});
```
### Ordem de execução
É importante ter em mente que os hooks têm uma ordem de execução específica. Isto é especialmente importante quando vários hooks estão registrados em diferentes níveis ([serviço](#Service-level-declaration) e/ou [ação](#Action-level-declaration)).  Em geral, os hooks têm a seguinte lógica de execução:

- `before` hooks: global (`*`) `->` nível de serviço `->` nível de ação.

- `after` hooks: nível de ação `->` nível de serviço `->` global (`*`).

{% note info%}
When using several hooks it might be difficult visualize their execution order. However, you can set the [`logLevel` to `debug`](logging.html#Log-Level-Setting) to quickly check the execution order of global and service level hooks.
{% endnote %}

**Exemplo de uma cadeia de execução de hook global, a nível de serviço & de ação**
```js
broker.createService({
    name: "greeter",
    hooks: {
        before: {
            "*"(ctx) {
                broker.logger.info(chalk.cyan("Before all hook"));
            },
            hello(ctx) {
                broker.logger.info(chalk.magenta("  Before hook"));
            }
        },
        after: {
            "*"(ctx, res) {
                broker.logger.info(chalk.cyan("After all hook"));
                return res;
            },
            hello(ctx, res) {
                broker.logger.info(chalk.magenta("  After hook"));
                return res;
            }
        },
    },

    actions: {
        hello: {
            hooks: {
                before(ctx) {
                    broker.logger.info(chalk.yellow.bold("    Before action hook"));
                },
                after(ctx, res) {
                    broker.logger.info(chalk.yellow.bold("    After action hook"));
                    return res;
                }
            },

            handler(ctx) {
                broker.logger.info(chalk.green.bold("      Action handler"));
                return `Hello ${ctx.params.name}`;
            }
        }
    }
});
```
**Saída produzida por hooks globais, nível de serviço & de ação**
```bash
INFO  - Before all hook
INFO  -   Before hook
INFO  -     Before action hook
INFO  -       Action handler
INFO  -     After action hook
INFO  -   After hook
INFO  - After all hook
```

### Reusabilidade
A maneira mais eficiente de reutilizar hooks é declarando-os como métodos de serviço em um arquivo separado e importando-os com o mecanismo [mixin](services.html#Mixins). Dessa forma, um único gancho pode ser facilmente compartilhado entre várias ações.

```js
// authorize.mixin.js
module.exports = {
    methods: {
        checkIsAuthenticated(ctx) {
            if (!ctx.meta.user)
                throw new Error("Unauthenticated");
        },
        checkUserRole(ctx) {
            if (ctx.action.role && ctx.meta.user.role != ctx.action.role)
                throw new Error("Forbidden");
        },
        checkOwner(ctx) {
            // Check the owner of entity
        }
    }
}
```

```js
// posts.service.js
const MyAuthMixin = require("./authorize.mixin");

module.exports = {
    name: "posts",
    mixins: [MyAuthMixin]
    hooks: {
        before: {
            "*": ["checkIsAuthenticated"],
            create: ["checkUserRole"],
            update: ["checkUserRole", "checkOwner"],
            remove: ["checkUserRole", "checkOwner"]
        }
    },

    actions: {
        find: {
            // No required role
            handler(ctx) {}
        },
        create: {
            role: "admin",
            handler(ctx) {}
        },
        update: {
            role: "user",
            handler(ctx) {}
        }
    }
};
```
### Armazenamento local
A propriedade `locals` do `Contexto` é um armazenamento simples que pode ser usado para armazenar alguns dados adicionais e passá-los para o manipulador de ações. A propriedade `locals` utilizada em conjunto com hooks formam um poderoso combo:

**Configurando `ctx.locals` em um before hook**
```js
module.exports = {
    name: "user",

    hooks: {
        before: {
            async get(ctx) {
                const entity = await this.findEntity(ctx.params.id);
                ctx.locals.entity = entity;
            }
        }
    },

    actions: {
        get: {
            params: {
                id: "number"
            },
            handler(ctx) {
                this.logger.info("Entity", ctx.locals.entity);
            }
        }
    }
}
```
