# Promises

## Motivação

Considere a seguinte função JavaScript síncrona para ler um arquivo e analisá-lo como JSON. É simples e fácil de ler, mas você não gostaria de usá-lo na maioria das aplicações, uma vez que irá bloquear. Isto significa que, enquanto você está lendo o arquivo do disco (uma operação lenta) nada mais pode acontecer.

```js
function readJSONSync(filename) {
  return JSON.parse(fs.readFileSync(filename, 'utf8'));
}
```

Para tornar o nosso aplicativo *performancático* e responsivo, precisamos fazer todas as operações que envolvem IO ser assíncronas. A maneira mais simples de fazer isso seria usar uma chamada de retorno(callback). No entanto, uma implementação ingênua provavelmente vai dar errado:

```js
function readJSON(filename, callback){
  fs.readFile(filename, 'utf8', function (err, res){
    if (err) return callback(err);
    callback(null, JSON.parse(res));
  });
}
```

O parâmetro de retorno de chamada adicional confunde nossa idéia do que é a entrada e qual é o valor de retorno.

- O parâmetro adicional do callback confunde nossa idéia do que é a entrada e qual é o valor de retorno.
- Ele não funciona com as primitivas de controle de fluxo.
- Ele não lida com erros lançados pelo `JSON.parse`

Precisamos lidar com erros lançados pelo `JSON.parse` mas também precisamos ter cuidado para não lidar com erros lançados pela função callback. No momento em que fizemos tudo isso nosso código é uma bagunça de tratamento de erros:

```js
function readJSON(filename, callback){
  fs.readFile(filename, 'utf8', function (err, res){
    if (err) return callback(err);
    try {
      res = JSON.parse(res);
    } catch (ex) {
      return callback(ex);
    }
    callback(null, res);
  });
}
```

Apesar de toda essa bagunça de código de tratamento de erros, ainda ficamos com o problema do parâmetro do callback. Promises irão ajudá-lo a lidar com os erros, naturalmente, e escrever um código mais limpo por não ter parâmetros no callback, e sem modificar a arquitetura subjacente (isto é, você pode implementá-los em JavaScript puro e usá-los para encapsular operações assíncronas existentes).

## O que é uma promise?

A idéia central por trás da *promise* é que uma *promise* representa o resultado de uma operação assíncrona. A *promise* está em um dos três estados diferentes:

- pendente(pending) - O estado inicial de uma *promise*.
- cumprida(fulfilled) - O estado de uma *promise* que representa uma operação bem sucedida.
- rejeitados(rejected) - O estado de uma *promise* que representa uma operação falhou.

Uma vez que uma promessa é cumprida ou rejeitada, é imutável (ou seja, ele nunca pode mudar novamente).

## Construindo uma promessa

Depois de todas os APIs retornarem *promises*, deve ser relativamente raro você precisar construir um à mão. Nesse meio tempo, precisamos de uma maneira para criar um polyfill para APIs existentes. Por exemplo:

```js
function readFile(filename, enc){
  return new Promise(function (fulfill, reject){
    fs.readFile(filename, enc, function (err, res){
      if (err) reject(err);
      else fulfill(res);
    });
  });
}
```

Nós usamos `new Promise` para construir a *promise*. Nós damos ao `construtor` uma **factory** que faz o trabalho real. Essa função é chamada imediatamente com dois argumentos. O primeiro argumento cumpre a *promise* e o segundo argumento rejeita a *promise*. Uma vez que a operação foi concluída, chamamos a função apropriada.

## Aguardando uma *promises*

Para utilizar uma *promise*, temos que de alguma forma ser capaz de esperar por ela para ser cumprida ou rejeitada. A maneira de fazer isso é usando `promise.done` (ver aviso no final desta seção se tentar executar estas amostras).

Com isto em mente, é fácil para re-escrever a nossa função readJSON antes de usar *promise*:

```js
function readJSON(filename){
  return new Promise(function (fulfill, reject){
    readFile(filename, 'utf8').done(function (res){
      try {
        fulfill(JSON.parse(res));
      } catch (ex) {
        reject(ex);
      }
    }, reject);
  });
}
```

Este ainda tem um monte de código de tratamento de erros (vamos ver como nós podemos melhorar isso na próxima seção), mas é muito menos propenso a erros para escrever, e nós já não temos um parâmetro extra estranho.

### Não padrão

Note-se que `promise.done` (utilizado nos exemplos nesta seção) não foi padronizado. É apoiado pela maioria das grandes bibliotecas de *promise*, porém, e é útil tanto como instrumento de ensino e no código de produção. Eu recomendo usá-lo junto com o seguinte polyfill (minified / unminified)

```html
<script src="https://www.promisejs.org/polyfills/promise-done-7.0.4.min.js"></script>
```

## Transformação / Encadeamento

Seguindo o nosso exemplo através, o que realmente queremos fazer é transformar a *promise* através de outra operação. No nosso caso, esta segunda operação é síncrona, mas pode apenas facilmente ter sido uma operação assíncrona. Felizmente, as *promises* têm um método (totalmente padronizado, exceto jQuery) para transformar as *promises* e as operações de encadeamento.

Simplificando, `.then` é para `.done` como `.Map` é para `.forEach`. Para colocar isso de outra forma, usar `.then` sempre que você está indo fazer algo com o resultado (mesmo se isso é só esperar que ela termine) e usar `.done` sempre que você não está pensando em fazer qualquer coisa com o resultado.

Agora nós podemos re-escrever o nosso exemplo original como simplesmente:

```js
function readJSON(filename){
  return readFile(filename, 'utf8').then(function (res){
    return JSON.parse(res)
  })
}
```

Sendo `JSON.parse` apenas uma função, podemos re-escrever isto como:

```js
function readJSON(filename){
  return readFile(filename, 'utf8').then(JSON.parse);
}
```

Isto é muito próximo ao exemplo síncrono simples que começamos com.

## Implementações / Polyfills

*Promises* são úteis tanto em node.js quanto no navegador.

## jQuery

Isto parece um bom momento para avisá-lo de que o que `jQuery` chama a *promise* é de fato totalmente diferente do que todo mundo chama de uma *promise*. As *promises* de jQuery tem uma API mal pensada que provavelmente só irá confundi-lo. Felizmente, em vez de usar estranha versão do `jQuery` de uma *promise*, você pode simplesmente convertê-lo para uma *promise* padronizada realmente simples:

```js
var jQueryPromise = $.ajax('/data.json');
var realPromise = Promise.resolve(jQueryPromise);
//agora apenas use `realPromise` como quiser.
```

## Navegador

*Promises* são atualmente suportadas apenas por uma muito pequena seleção de navegadores (ver tabelas de compatibilidade kangax). A boa notícia é que eles são extremamente fáceis de criar um *polyfill* (minified / unminified):

```html
<script src="https://www.promisejs.org/polyfills/promise-7.0.4.min.js"></script>
```

Nenhum dos navegadores atualmente suporta `Promise.prototype.done` por isso, se você quiser usar esse recurso, e você não está incluindo o polyfill acima, você deve, pelo menos, incluir este polyfill (minified / unminified):

```html
<script src="https://www.promisejs.org/polyfills/promise-done-7.0.4.min.js"></script>
```

## Node.js

Não é geralmente visto como uma boa prática criar polyfill para coisas em Node.js. Em vez disso, é melhor apenas requisitar a biblioteca sempre que você precisar.

Para instalar a *promise* de execução:

```
npm install promise --save
```

Depois, você pode carregá-la em uma variável local usando "require":

```js
var Promise = require('promise');
```

A biblioteca "promise" também oferece algumas extensões realmente úteis para interagir com node.js

```js
var readFile = Promise.denodeify(require('fs').readFile);
// now `readFile` will return a promise rather than expecting a callback

function readJSON(filename, callback){
  // If a callback is provided, call it with error as the first argument
  // and result as the second argument, then return `undefined`.
  // If no callback is provided, just return the promise.
  return readFile(filename, 'utf8').then(JSON.parse).nodeify(callback);
}
```

## Leitura adicional

[Patterns](https://www.promisejs.org/patterns/) - padrões de uso da *promise*, introduzindo vários métodos auxiliares que você vai economizar tempo.
[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - A rede de desenvolvedores Mozilla tem excelente documentação de *promises*.
[YouTube](https://www.youtube.com/watch?v=qbKWsbJ76-s) - Um vídeo meu JSConf.eu que discute muitas das mesmas coisas que aparecem neste artigo.
