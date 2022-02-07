title: События
---
У брокера есть встроенная шина событий для поддержки [управляемой событиями архитектуры](http://microservices.io/patterns/data/event-driven-architecture.html) и для отправки событий локальным и удаленным сервисам.

{% note info %}
Please note that built-in events are fire-and-forget meaning that if the service is offline, the event will be lost. For persistent, durable and reliable events please check [moleculer-channels](https://github.com/moleculerjs/moleculer-channels).
{% endnote %}

# Сбалансированные события
Прослушиватели событий расположены в логических группах. Это означает, что в каждой группе вызывается только один слушатель.

> **Пример:** у вас есть 2 основных сервиса: `users` & `payments`. Оба подписываются на cобытие `user.created`. Вы запускаете 3 экземпляра сервиса `users` и 2 экземпляра сервиса `payments`. Когда вы выдаете событие `user.created`, только один экземпляр сервиса `users` и один экземпляр сервиса `payments` получит событие.

<div align="center">
    <img src="assets/balanced-events.gif" alt="Диаграмма сбалансированных событий" />
</div>

Название группы происходит от имени сервиса, но оно может быть перезаписано в определении события в сервисах.

**Пример**
```js
module.exports = {
    name: "payment",
    events: {
        "order.created": {
            // Регистрация обработчика в группу "other" вместо группы "payment".
            group: "other",
            handler(ctx) {
                console.log("Payload:", ctx.params);
                console.log("Sender:", ctx.nodeID);
                console.log("Metadata:", ctx.meta);
                console.log("The called event name:", ctx.eventName);
            }
        }
    }
}
```

## Выдача сбалансированного события
Отправлять сбалансированные события с помощью функции `broker.emit`. Первый параметр — это название события, второй параметр — это полезная нагрузка. _Чтобы отправить несколько значений, оберните их в `Object`._

```js
// `user` будет сериализован к транспорту.
broker.emit("user.created", user);
```

Укажите, какие группы/сервисы должны получить событие:
```js
// Только `mail` & `payments` сервисы принимают его
broker.emit("user.created", user, ["mail", "payments"]);
```

# Широковещательное событие
Штроковещательное событие отправляется всем доступным локальным & удаленным сервисам. Оно не сбалансировано, все экземпляры сервиса получат его.

<div align="center">
    <img src="assets/broadcast-events.gif" alt="Диаграмма широковещательных событий" />
</div>

Отправка широковещательного события выполняется с помощью метода `broker.broadcast`.
```js
broker.broadcast("config.changed", config);
```

Укажите, какие группы/сервисы должны получить событие:
```js
// Отправляем во все экземпляры сервиса "mail"
broker.broadcast("user.created", { user }, "mail");

// Отправляем всем "user" & "purchase" экземплярам сервиса.
broker.broadcast("user.created", { user }, ["user", "purchase"]);
```

## Локальное широковещательное событие
Для отправки события только всем локальным службам используется метод `broker.broadcastLocal`.
```js
broker.broadcastLocal("config.changed", config);
```

# Подписка на события

Версия `v0.14` поддерживает обработчики событий на основе контекста. Контекст событий полезен, если вы используете архитектуру под управлением событий и хотите отслеживать ваши события. Если вы знакомы с [Action Context](context.html), вы будете чувствовать себя как дома. Контекст события очень похож на контекст действий, за исключением нескольких новых свойств относящихся к событиям. [Посмотреть полный список всех свойств](context.html)

{% note info Legacy event handlers %}

Вам не нужно переписывать все существующие обработчики событий, так как Moleculer все еще поддерживает старую сигнатуру `"user.created"(payload) { ... }`. Он способен обнаружить различные сигнатуры обработчиков событий:
- Если найдена сигнатура `"user.created"(ctx) { ... }`, то вызов выполнится с контекстом событий.
- Если нет, вызов выполнится со старыми аргументами & 4-й аргумент будет контекст события, например `"user.created"(payload, отправитель, eventName, ctx) {...}`
- You can also force the usage of the new signature by setting `context: true` in the event declaration

{% endnote %}

**Обработчик событий на основе контекста & выпуск вложенного события**
```js
module.exports = {
    name: "accounts",
    events: {
        "user.created"(ctx) {
            console.log("Payload:", ctx.params);
            console.log("Sender:", ctx.nodeID);
            console.log("Metadata:", ctx.meta);
            console.log("The called event name:", ctx.eventName);

            ctx.emit("accounts.created", { user: ctx.params.user });
        },

        "user.removed": {
            // Force to use context based signature
            context: true,
            handler(other) {
                console.log(`${this.broker.nodeID}:${this.fullName}: Event '${other.eventName}' received. Payload:`, other.params, other.meta);
            }
        }
    }
};
```


Подписка на события осуществляется в ['events' свойстве сервиса](services.html#events). Допускается использование масок (`?`, `*`, `**`) в именах событий.

```js
module.exports = {
    events: {
        // Подписка на событие `user.created`
        "user.created"(ctx) {
            console.log("User created:", ctx.params);
        },

        // Подписака на все собтия `user`, например "user.created" или "user.removed"
        "user.*"(ctx) {
            console.log("User event:", ctx.params);
        }
        // Подписка на каждое событие
        // Используется сигнатура устаревшего обработчика событий с контекстом
        "**"(payload, sender, event, ctx) {
            console.log(`Event '${event}' received from ${sender} node:`, payload);
        }
    }
}
```

## Валидация параметров события
Аналогично проверке параметра действия, поддерживается проверка параметров события. Как и в определении действий, следует определить `параметры` для событий, а встроенный `Валидатор` валидирует параметры в событиях.

```js
// mailer.service.js
module.exports = {
    name: "mailer",
    events: {
        "send.mail": {
            //  Схема валидации
            params: {
                from: "string|optional",
                to: "email",
                subject: "string"
            },
            handler(ctx) {
                this.logger.info("Event received, parameters OK!", ctx.params);
            }
        }
    }
};
```
> Ошибки валидации не отсылаются обратно вызывающему, они логируются и могут быть пойманы с помощью нового [глобального обработчика ошибок](broker.html#Global-error-handler).

# Внутренние события
Брокер транслирует некоторые внутренние события. Эти события всегда начинаются с префикса `$`.

## `$services.changed`
Брокер отправляет это событие, если локальный узел или удаленный узел загружает или уничтожает сервисы.

**Payload**

| Название       | Тип       | Описание                               |
| -------------- | --------- | -------------------------------------- |
| `localService` | `Boolean` | True, если Локальный сервис изменился. |

## `$circuit-breaker.opened`
Брокер отправляет это событие, когда модуль прерывания зацикливаний изменит свое состояние на `открыто`.

**Payload**

| Название   | Тип      | Описание           |
| ---------- | -------- | ------------------ |
| `nodeID`   | `String` | Идентификатор узла |
| `action`   | `String` | Название действия  |
| `failures` | `Number` | Количество сбоев   |


## `$circuit-breaker.half-opened`
Брокер отправляет это событие, когда модуль прерывания зацикливаний изменит свое состояние на `полуоткрыто`.

**Payload**

| Название | Тип      | Описание           |
| -------- | -------- | ------------------ |
| `nodeID` | `String` | Идентификатор узла |
| `action` | `String` | Название действия  |

## `$circuit-breaker.closed`
Брокер отправляет это событие, когда модуль прерывания зацикливаний изменит свое состояние на `закрыто`.

**Payload**

| Название | Тип      | Описание           |
| -------- | -------- | ------------------ |
| `nodeID` | `String` | Идентификатор узла |
| `action` | `String` | Название действия  |

## `$node.connected`
Брокер посылает это событие когда узел подключен или переподключен.

**Payload**

| Название      | Тип       | Описание                 |
| ------------- | --------- | ------------------------ |
| `node`        | `Node`    | Объект информации о узле |
| `reconnected` | `Boolean` | Переподключено?          |

## `$node.updated`
Брокер отправляет это событие, когда он получил сообщение INFO от узла (например, серивс загружен или уничтожен).

**Payload**

| Название | Тип    | Описание                 |
| -------- | ------ | ------------------------ |
| `node`   | `Node` | Объект информации о узле |

## `$node.disconnected`
Брокер посылает это событие когда узел откючен (плавно или неожиданно).

**Payload**

| Название     | Тип       | Описание                                                                       |
| ------------ | --------- | ------------------------------------------------------------------------------ |
| `node`       | `Node`    | Объект информации о узле                                                       |
| `unexpected` | `Boolean` | `true` - Не получен хартбит, `false` - Получено сообщение `ОТКЛЮЧИТЬ` от узла. |

## `$broker.started`
Брокер отправляет это событие после того, как вызван `broker.start()` и все локальные сервисы запущены.

## `$broker.stopped`
Брокер отправляет это событие после того, как вызван `broker.stop()` и все локальные сервисы остановлены.

## `$transporter.connected`
Транспорт отправляет это событие после подключения транспорта.

## `$transporter.disconnected`
Транспорт отправляет это событие после отключения транспорта.

## `$broker.error`
The broker emits this event when an error occurs in the [broker](broker.html). **Event payload**
```js
{
  "error": "<the error object with all properties>"
  "module": "broker" // Name of the module where the error happened
  "type": "error-type" // Type of error. Full of error types: https://github.com/moleculerjs/moleculer/blob/master/src/constants.js
}
```

## `$transit.error`
The broker emits this event when an error occurs in the transit module. **Event payload**
```js
{
  "error": "<the error object with all properties>"
  "module": "transit" // Name of the module where the error happened
  "type": "error-type" // Type of error. Full of error types: https://github.com/moleculerjs/moleculer/blob/master/src/constants.js
}
```

## `$transporter.error`
The broker emits this event when an error occurs in the [transporter](networking.html#Transporters) module. **Event payload**
```js
{
  "error": "<the error object with all properties>"
  "module": "transit" // Name of the module where the error happened
  "type": "error-type" // Type of error. Full of error types: https://github.com/moleculerjs/moleculer/blob/master/src/constants.js
}
```

## `$cacher.error`
The broker emits this event when an error occurs in the [cacher](caching.html) module. **Event payload**
```js
{
  "error": "<the error object with all properties>"
  "module": "transit" // Name of the module where the error happened
  "type": "error-type" // Type of error. Full of error types: https://github.com/moleculerjs/moleculer/blob/master/src/constants.js
}
```

## `$discoverer.error`
The broker emits this event when an error occurs in the [discoverer](registry.html) module. **Event payload**
```js
{
  "error": "<the error object with all properties>"
  "module": "transit" // Name of the module where the error happened
  "type": "error-type" // Type of error. Full of error types: https://github.com/moleculerjs/moleculer/blob/master/src/constants.js
}
```


