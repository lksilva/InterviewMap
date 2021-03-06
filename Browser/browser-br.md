<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Tabela de donteúdos** _generated with [DocToc](https://github.com/thlorenz/doctoc)_

- [Mecanismo de evento](#mecanismo-de-evento)
  - [As três fases do evento de propagação](#as-três-fases-do-evento-de-propagação)
  - [Inscrição no Evento](#inscrição-no-evento)
  - [Delegação de Eventos](#delegação-de-eventos)
- [Cross Domain](#cross-domain)
  - [JSONP](#jsonp)
  - [CORS](#cors)
  - [document.domain](#documentdomain)
  - [postMessage](#postmessage)
- [Event Loop](#event-loop)
  - [Event Loop no Node](#event-loop-no-node)
    - [timer](#timer)
    - [pending callbacks](#pending-callbacks)
    - [idle, prepare](#idle-prepare)
    - [poll](#poll)
    - [check](#check)
    - [close callbacks](#close-callbacks)
- [Storage](#storage)
  - [cookie，localStorage，sessionStorage，indexDB](#cookielocalstoragesessionstorageindexdb)
  - [Service Worker](#service-worker)
- [Rendering mechanism](#rendering-mechanism)
  - [Difference between Load & DOMContentLoaded](#difference-between-load--domcontentloaded)
  - [Layers](#layers)
  - [Repaint & Reflow](#repaint--reflow)
  - [Minimize Repaint & Reflow](#minimize-repaint--reflow)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Mecanismo de evento

## As três fases do evento de propagação

O evento de propagação tem três fases:

- O objeto de evento se propaga do Window para o pai do alvo.
  Captura de eventos irá disparar.
- O objeto de evento chega ao destino de evento do objeto alvo. Eventos registrados para o alvo serão disparados.
- O objeto de evento propaga do alvo pai para o Window. Eventos Bubbling irão disparar.

O evento de propagação geralmente segue a sequência abaixo, mas existe excessões. Se o nó do alvo é registrado para ambos bubbling e capturando evento, eventos serão invocados na ordem que eles foram registrados.

```js
// O seguinte code ira mostrar bubbling primeiro e então disparar a captura de eventos
node.addEventListener(
  "click",
  event => {
    console.log("bubble");
  },
  false
);
node.addEventListener(
  "click",
  event => {
    console.log("capture");
  },
  true
);
```

## Inscrição no Evento

Geralmente, usamos o `addEventListener` para registrar um evento, e o `useCapture` é uma função parâmentro, do qual recebe um valos boleano, o valor padrão é `false`. `useCapture` determina se o evento registrado é está capturando eventos ou um evento bubbling. Para o objeto parâmentro, você pode usar as seguintes propriedades:

- `capture`, valor boleano, igua a `useCapture`
- `once`, valor boleano, `true` indica que o callback deveria ser chamado mais de uma vez, depois invocado o listener será removido
- `passive`, boleano, significa ele nunca irá chamar `preventDefault`

Generalizando, nós queremos apenas que o evento dispare no alvo. Para alcançar isso nós usamos `stopPropagation` para previnir a propagação do evento. Usualmente, nós deveriamos pensar que `stopPropagation` é usado para parar o evento bubbling, mas essa função também pode prevenir o evento capturado.
`stopImmediatePropagation` pode alcançar o mesmo efeito, e isso pode também previnir outros listeners do mesmo evento a partir da chamada.

```js
node.addEventListener(
  "click",
  event => {
    event.stopImmediatePropagation();
    console.log("bubbling");
  },
  false
);
// Clicando no nó irá apenas executar a função abaixo, essa função não irá executar
node.addEventListener(
  "click",
  event => {
    console.log("capture ");
  },
  true
);
```

## Delegação de Eventos

Se um nó filho dentro de um nó pai é dinâmicamente gerado, eventos no nó deveram ser adicionados ao nó pai:

```html
<ul id="ul">
  <li>1</li>
  <li>2</li>
  <li>3</li>
  <li>4</li>
  <li>5</li>
</ul>
<script>
  let ul = document.querySelector("#ul");
  ul.addEventListener("click", event => {
    console.log(event.target);
  });
</script>
```

Delegação de eventos tem as seguintes vatagens sobre adicionar eventos em linha reta para os nós filhos:

- Economiza memória
- Não precisa remover event listeners nos nós filhos

# Cross Domain

Navegadores tem a política de mesma origem por razões de segurança. Em outras palavras, se o protocolo, nome do domínio ou porta tem uma diferença, isso seria cross-domain, e a requisição irá falhar.

Nós podemos resolver o problema de cross-domain através dos seguintes métodos:

## JSONP

O princípio do JSONP é muito simples, isso é fazer uso da `<script>` tag não sujeito a política de mesma origem. Use o atributo `src` da `<script>` tag e prover uma função callback to receber os dados:

```js
<script src="http://domain/api?param1=a&param2=b&callback=jsonp"></script>
<script>
    function jsonp(data) {
    	console.log(data)
	}
</script>
```

JSONP é simples para usar e tem ótima compatibilidade, mas isso é limitado a requisições `get`.

Você pode encontrar a situações onde você tem algum nome de callback em múltiplas requisições JSONP. Nessa situação você precisa encapsular JSONP. O seguinte é uma simples implementação:

```js
function jsonp(url, jsonpCallback, success) {
  let script = document.createElement("script");
  script.src = url;
  script.async = true;
  script.type = "text/javascript";
  window[jsonpCallback] = function(data) {
    success && success(data);
  };
  document.body.appendChild(script);
}
jsonp("http://xxx", "callback", function(value) {
  console.log(value);
});
```

## CORS

CORS requer suporte dos navegadores e do backend ao mesmo tempo. Internet Explorer 8 e 9 expõe CORS via XDomainRequest object.

O navegador vai automaticamente realizar CORS. A chave para implementar CORS é o backend. Enquanto o backend implementa CORS, ele habilita cross-domain.

O servidor seta `Access-Control-Allow-Origin` para habilitar CORS. Essa propriedade especifica quais dominios podem acesar os recursos. Se definido como curinga, todos os websites podem acessar os recursos.

## document.domain

Esse só pode ser usando para o mesmo second-level de domínio, por exemplo, `a.test.com` e `b.test.com` são adequados para esse caso.

Set `document.domain = 'test.com'` habilitaria CORS com o mesmo domínio de segundo nível.

## postMessage

Esse método é geralmente usado para pegar dados a partir de página de terceiros. Uma página envia uma mensagem, a outra página verifica os arquivos e recebe a mensagem:

```js
// envia a página
window.parent.postMessage("message", "http://test.com");
// recebe da página
var mc = new MessageChannel();
mc.addEventListener("message", event => {
  var origin = event.origin || event.originalEvent.origin;
  if (origin === "http://test.com") {
    console.log("success");
  }
});
```

# Event Loop

Como bem sabemos, JS é não bloqueante e linguagem single-threaded, porque JS nasceu para interagir com o navegador no inicio. Se JS fosse uma linguagem multi-threaded, nós teriamos problemas para manipular o DOM em multiplas threads (imagine adicionar nós em um thread e deletar nós em outra thread ao mesmo tempo), contudo nós deveriamos introduzir uma tranca de leitura-escrita para resolver esse problema.

Executando contexto, gerado durante a execução do JS, será empurrado dentro pilha de chamadas sequencialmente. Código assíncrono irá desligar ser empurrado dentro da filha de tarefas, existe múltiplos tipos de tarefas. Uma vez a pilha de chamadas estive vázia, o Event Loop vai processar a próxima mensagem na fila de tarefas e colocar dentro pilha de chamadas para executar, portanto essencialmente a operação assíncrona em JS é atualmente síncrona.

```js
console.log("script start");

setTimeout(function() {
  console.log("setTimeout");
}, 0);

console.log("script end");
```

O código acima é assíncrono, apesar do `setTimeout` delay ser 0. Issso aconteceu porque o padrão HTML5 estipula que o segundo parâmetro da função `setTimeout` não deve ser menos que 4 milissegundos, de outra forma será automático. Então `setTimeout` é logado depois do `script end`.

Tarefas diferentes são assinadas para fila de tarefas diferentes. Tarefas podem ser divididas em `microtasks` e `macrotasks`. Na especificação ES6, uma `microtask` é chamada em um `job` e uma `macrotask` é chamada em uma `task`.

```js
console.log("script start");

setTimeout(function() {
  console.log("setTimeout");
}, 0);

new Promise(resolve => {
  console.log("Promise");
  resolve();
})
  .then(function() {
    console.log("promise1");
  })
  .then(function() {
    console.log("promise2");
  });

console.log("script end");
// script start => Promise => script end => promise1 => promise2 => setTimeout
```

Apesar `setTimeout` estar definido antes `Promise`, o print acima ainda ocorro porque `Promise` pertence a microtask e `setTimeout` pertence a macrotask.

Microtask incluem `process.nextTick`, `promise`, `Object.observe` e `MutationObserver`.

Macrotasks incluem `script`, `setTimeout`, `setInterval`, `setImmediate`, `I/O` and `UI rendering`.

Muitas pessoas cometem o mal entendido que microtasks sempre são executadas antes das macrotasks e isso não é verdade. Porque a macrotask incluem `script`, o browser ira realizar essas macrotask primeiro, seguido por qualquer microtask em um código assíncrono.

Então a sequência correta de um event loop parece algo como esse:

1. Executa código assíncrono, que pertence a uma macrotask
2. Uma vez a pilha de chamadas vázia, busca se qualquer microtask precisa ser executada
3. Execute todas as microtasks
4. Se necessário, renderize a UI
5. Então começe o próximo ciclo do event loop, e execute a operação assíncrona em uma macrotask

De acordo com a sequência acima do event loop, se o código assíncrono é uma macrotask tendo uma grande número de calculos e precisa operar no DOM, colocamos a operação do DOM em uma microtask para uma rápida resposta na interface.

## Event Loop no Node

O event loop no Node não se comporta da mesma maneira como no navegador.

O event loop no Node é dividido em 6 fases, e eles são executados em ordem repetidamente:

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     pending callbacks │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<──---|   connections │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

### timer

A fase do `timer` executa as callback de `setTimeout` e `setInterval`.

O `Time` especifica o tempo que a callback vai executar tanto quanto eles podem ser agendados, depois a quantidade especifica de tempo passado ao invés do tempo exato que uma pessoa quer que ele seja executado.

O limite minimo de tempo está entre `[1, 2147483647]`. Se o conjunto de tempo não está nesse intervalo, ele será setado para 1.

### pending callbacks

Essa fase executa I/O callbacks diferida para a próxima iteração do loop.

### idle, prepare

A fase do `idle, prepare` serve para implementação interna.

### poll

Essa fase recebe novos eventos de I/O; executa o I/O relacionado as callbacks (quase sempre com a excessão de fechar as callbacks, os agendados pelos timers, e setImmediate()); node irá bloquear aqui quando apropriado.

O `poll` é a fase de duas funções principais:

1. Calcular quanto tempo ele deverá bloquear e executar a votação do I/O, então
2. Processar os eventos na fila de votação.

Quando o evet loop entra na fase de `poll` não existe timers agendados, uma das duas coisas vai acontecer:

- Se o a fila do `poll` não estive vázia, o evet loop vai iterar através dessa fila de callbacks executando eles sincronamente até ou a fila ser percorrida, ou o sistema chegar no seu limite.

- If the `poll` queue is empty, one of two more things will happen:
  1. If scripts have been scheduled by `setImmediate`. the event loop will end the `poll` phase and continue to the check phase to execute those scheduled scripts.
  2. If scripts have not been scheduled by `setImmediate`, the event loop will wait for callbacks to be added to the queue, then execute them immediately.

Once the `poll` queue is empty the event loop will check for timers whose time thresholds have been reached. If one or more timers are ready, the event loop will wrap back to the timers phase to execute those timers' callbacks.

### check

The `check` phase executes the callbacks of `setImmediate`.

### close callbacks

The `close` event will be emitted in this phase.

And in Node, the order of execution of timers is random in some cases:

```js
setTimeout(() => {
  console.log("setTimeout");
}, 0);
setImmediate(() => {
  console.log("setImmediate");
});
// Here, it may log setTimeout => setImmediate
// It is also possible to log the opposite result, which depends on performance
// Because it may take less than 1 millisecond to enter the event loop, `setImmediate` would be executed at this time.
// Otherwise it will execute `setTimeout`
```

Certainly, in this case, the execution order is the same:

```js
var fs = require("fs");

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log("timeout");
  }, 0);
  setImmediate(() => {
    console.log("immediate");
  });
});
// Because the callback of `readFile` was executed in `poll` phase
// Founding `setImmediate`,it immediately jumps to the `check` phase to execute the callback
// and then goes to the `timer` phase to execute `setTimeout`
// so the above output must be `setImmediate` => `setTimeout`
```

The above is the implementation of the macrotask. The microtask will be executed immediately after each phase is completed.

```js
setTimeout(() => {
  console.log("timer1");

  Promise.resolve().then(function() {
    console.log("promise1");
  });
}, 0);

setTimeout(() => {
  console.log("timer2");

  Promise.resolve().then(function() {
    console.log("promise2");
  });
}, 0);
// The log result is different, when the above code is executed in browser and node
// In browser, it will log: timer1 => promise1 => timer2 => promise2
// In node, it may log: timer1 => timer2 => promise1 => promise2
// or timer1, promise1, timer2, promise2
```

`process.nextTick` in Node will be executed before other microtasks.

```js
setTimeout(() => {
  console.log("timer1");

  Promise.resolve().then(function() {
    console.log("promise1");
  });
}, 0);

process.nextTick(() => {
  console.log("nextTick");
});
// nextTick => timer1 => promise1
```

# Storage

## cookie，localStorage，sessionStorage，indexDB

|         features          |                                       cookie                                       |               localStorage                |                          sessionStorage                           |                  indexDB                  |
| :-----------------------: | :--------------------------------------------------------------------------------: | :---------------------------------------: | :---------------------------------------------------------------: | :---------------------------------------: |
|    Life cycle of data     |       generally generated by the server, but you can set the expiration time       | unless cleared manually, it always exists | once the browser tab is closed, it will be cleaned up immediately | unless cleared manually, it always exists |
|   Storage size of data    |                                         4K                                         |                    5M                     |                                5M                                 |                 unlimited                 |
| Communication with server | it is carried in the header everytime, and has a performance impact on the request |            doesn't participate            |                        doesn't participate                        |            doesn't participate            |

As we can see from the above table, `cookies` are no longer recommended for storage. We can use `localStorage` and `sessionStorage` if we don't have much data to store. Use `localStorage` to store data that doesn't change much, otherwise `sessionStorage` can be used.

For `cookies`, we also need pay attention to security issue.

| attribute |                                                   effect                                                   |
| :-------: | :--------------------------------------------------------------------------------------------------------: |
|   value   | the value should be encrypted if used to save the login state, and the cleartext user ID shouldn't be used |
| http-only |                       cookies cannot be accessed through JS, for reducing XSS attack                       |
|  secure   |                        cookies can only be carried in requests with HTTPS protocol                         |
| same-site |              browsers cannot pass cookies in cross-origin requests, for reducing CSRF attacks              |

## Service Worker

> Service workers essentially act as proxy servers that sit between web applications, the browser and the network (when available). They are intended, among other things, to enable the creation of effective offline experiences, intercept network requests and take appropriate action based on whether the network is available, and update assets residing on the server. They will also allow access to push notifications and background sync APIs.

At present, this technology is usually used to cache files and increase the render speed of the first screen. We can try to implement this function:

```js
// index.js
if (navigator.serviceWorker) {
  navigator.serviceWorker
    .register("sw.js")
    .then(function(registration) {
      console.log("service worker register success");
    })
    .catch(function(err) {
      console.log("servcie worker register error");
    });
}
// sw.js
// Listen for the `install` event, and cache the required files in the callback
self.addEventListener("install", e => {
  e.waitUntil(
    caches.open("my-cache").then(function(cache) {
      return cache.addAll(["./index.html", "./index.js"]);
    })
  );
});

// intercept all the request events
// use the cache directly if the requested data already existed in the cache; otherwise, send requests for data
self.addEventListener("fetch", e => {
  e.respondWith(
    caches.match(e.request).then(function(response) {
      if (response) {
        return response;
      }
      console.log("fetch source");
    })
  );
});
```

Open the page, we can see that the Service Worker has started in the `Application` pane of devTools:

![](https://user-gold-cdn.xitu.io/2018/3/28/1626b1e8eba68e1c?w=1770&h=722&f=png&s=192277)

In the Cache pane, we can also find that the files we need have been cached:

![](https://user-gold-cdn.xitu.io/2018/3/28/1626b20dfc4fcd26?w=1118&h=728&f=png&s=85610)

Refreshing the page, we can see that our cached data is read from the Service Worker:

![](https://user-gold-cdn.xitu.io/2018/3/28/1626b20e4f8f3257?w=2818&h=298&f=png&s=74833)

# Rendering mechanism

The mechanism of the browser engine usually has the following steps:

1. Parse HTML to construct the DOM tree.

2. Parse CSS to construct the CSSOM tree.

3. Create the render tree by combining the DOM & CSSOM.

4. Run layout based on the render tree, then calculate each node's exact coordinates on the screen.

5. Paint elements by GPU, composite layers and display on the screen.

![](https://user-gold-cdn.xitu.io/2018/4/11/162b2ab2ec70ac5b?w=900&h=352&f=png&s=49983)

When building the CSSOM tree, the rendering is blocked until the CSSOM tree is built. And building the CSSOM tree is a very cost-intensive process, so you should try to ensure that the level is flat and reduce excessive cascading. The more specific the CSS selector is, the slower the execution.

When the HTML is parsing the script tag, the DOM is paused and will restart from the paused position. In other words, the faster you want to render the first screen, the less you should load the JS file on the first screen. And CSS will also affect the execution of JS. JS will only be executed when the stylesheet is parsed. Therefore, it can be considered that CSS will also suspend the DOM in this case.

![](https://user-gold-cdn.xitu.io/2018/7/8/1647838a3b408372?w=1676&h=688&f=png&s=154480)

![](https://user-gold-cdn.xitu.io/2018/7/8/16478388e773b16a?w=1504&h=760&f=png&s=123231)

## Difference between Load & DOMContentLoaded

**Load** event occurs when all the resources (e.g. DOM、CSS、JS、pictures) have been loaded.

**DOMContentLoaded** event occurs as soon as the HTML of the pages has been loaded, no matter whether the other resources have been loaded.

## Layers

Generally，we can treat the document flow as a single layer. Some special attributes also could create a new layer. **Different Layers are independent**. So, it is recommended to create a new layer to render some elements which changes frequently. **But it is also a bad idea to create too many layers.**

The following attributes usually can create a new layer:

- 3Dtranslate: `translate3d`, `translateZ`

- `will-change`

- tags like: `video`, `iframe`

- animation achieved by `opacity`

- `position: fixed`

## Repaint & Reflow

Repaint and Reflow is a small step in the main rendering flow, but they have a great impact on the performance.

- Repaint occurs when the node changes but doesn't affect the layout, e.g. `color`.

- Reflow occurs when the node changes caused by layout or the geometry attributes.

Reflow will trigger a Repaint, but the opposite is not necessarily true. Reflow is much more expensive than Repaint. The changes in deep level node's attributes may cause a series of changes of its ancestral nodes.

Actions like the following may cause performance problems:

- change the window's size

- change font-family

- add or delete styles

- change texts

- change position & float

- change box model

You may not know that Repaint and Reflow has something to do with the **Event Loop**.

- In a event loop, when a microtask finishes, the engine will check whether the document needs update. As the refresh rate of the browse is 60Hz, this means it will update every 16ms.

- Then browser would check whether there are events like `resize` or `scroll` and if true, trigger the handlers. So the handlers of resize and scroll will be invoked every 16ms, which means automatic throttling.

- Evaluate media queries and report changes.

- Update animations and send events.

- Check whether this is a full-screen event.

- Execute `requestAnimationFrame` callback.

- Execute `IntersectionObserver` callback, which is used to determine whether an element should be displaying, usually in lazy-load, but has poor compatibility.

- Update the screen.

- The above events may occur in every frame. If there is idle time, the `requestIdleCallback` callback will be called.

All of the above are from [HTML Documents](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model).

## Minimize Repaint & Reflow

- Use `translate` instead of `top`:

```html
<div class="test"></div>
<style>
  .test {
    position: absolute;
    top: 10px;
    width: 100px;
    height: 100px;
    background: red;
  }
</style>
<script>
  setTimeout(() => {
    // occurs reflow
    document.querySelector(".test").style.top = "100px";
  }, 1000);
</script>
```

- Use `visibility` instead of `display: none`, because the former will only cause Repaint while the latter will cause Reflow, which changes the layout.

- Change the DOM when it is offline, e.g. change the DOM 100 times after set it `display: none` and then show it on screen. During this process there is only one Reflow.

- Do not put an attribute of a node inside a loop:

```js
for (let i = 0; i < 1000; i++) {
  // it will cause the reflow to get offsetTop, because it need to calculate the right value
  console.log(document.querySelector(".test").style.offsetTop);
}
```

- Do not use table to construct the layout, because even a little change will cause the re-construct.

- Animation speed matters: the faster it goes, the more Reflow. You can also utilize `requestAnimationFrame`.

- The css selector will search to match from right to left, so you'd better avoid deep level DOM node.

- As we know that the layer will prevent the changed node from affecting others, so it is good practice to create a new layer for animations with high frequency.

![](https://user-gold-cdn.xitu.io/2018/3/29/1626fb6f33a6f9d7?w=1588&h=768&f=png&s=263260)
