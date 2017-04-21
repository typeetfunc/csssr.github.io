## Что делать?

Вспомним код в начале:

```javascript
var URL = 'https://api.giphy.com/v1/gifs/random?api_key=dc6zaTOxFJmzC&tag=cats';

function app() {
  document.addEventListener(
    'click',
    () => 
      if (event.target.tagName === 'BUTTON') {
        fetch(URL)
          .then(response => response.json().data.image_url)
          .then(gifSrc => document.querySelector('img').setAttribute('src', gifSrc))
      }
  );
}
```

Составим схему происходящего:

![Effects](/images/side-effects/effects.svg)

Давайте попробуем представить эти действия в виде данных - опишем их простыми обьектами:

 - `click` это просто событие - как мы знаем события [уже представлены в виде объектов](https://learn.javascript.ru/obtaining-event-object). Нам сейчас не важны все свойства хватит только `target`
 - `HTTP` запрос мы можем представить в виде объекта содержашего все его параметры - в нашем случае это только `url`
 - `HTTP` ответ можно представить в виде объекта результата запроса
 - вставку ссылки в `img` можно описать обьектом со свойствами `selector`(с каким элементом что либо делаем), `op`(название операции которую делаем с элементом), `args`(необходимые аргументы)

![Data](/images/side-effects/data.svg)

Давайте выделим в нашем коде операции по созданию этих объектов. Создание объекта `HTTP` запроса из объекта события:

```javascript
function clickToReq(event) {
   if (event.target.tagName === 'BUTTON') {
    return {url: URL};
  }
}
```

Создание объекта модификации `DOM` из `HTTP` ответа:

```javascript
function responseToDomOp(res) {
  return {
    selector: 'img', 
    op: 'setAttribute',
    args: ['src', res.json().data.image_url]
  };
}
```

Тогда функция `app` запишется следующим образом:

```javascript
function render() {
  document.addEventListener('click',
    function(event) {
      var req = clickToReq(event);
      req && fetch(req.url)
        .then(responseToDomOp)
        .then(function (action) {
          document.querySelector(action.selector)
          [action.op](...action.args);
        });
      }
    }
  );
}
```

Но откуда взялись само `DOM` событие и `HTTP` ответ? Мы нигде их не вычисляем самостоятельно - они приходят к нам извне, из окружающего мира.

![Outside world](/images/side-effects/outside-world.svg)
Красное - окружающий мир, синее - наша логика

Значит задача абстракции сайд-эффектов сводится к задаче общения с окружающим внешним миром. Как люди общаются с друг другом? Обычно это происходит посредством _сообщений_: разговорных, текстовых и так далее(форма не важна).

Мы сообщаем собеседнику N сообщений и получаем в ответ от него M сообщений, где N и M числа от 0 до бесконечности. При этом собеседник может начать отвечать раньше чем мы закончили передачу своих сообщений, или выждать неопределенный интервал после всех наших сообщений, или вообще ничего нам не сообщить в ответ.

> Ключевым в концепции _обмена сообщениями_ и как раз является полное отсутсвие контроля над тем как собеседник распоряжается нашими сообщениями - и он в свою очередь не может никак повлиять на нашу реакцию на его сообщения(или их отсуствие).
> 

Давайте представим нашу программу в виде обмена сообщениями функции `app` и внешнего мир, для удобства мы разделим весь внешний мир на отдельные части - `DOM` и `HTTP`:

1. `DOM` посылает `app` событие клика

2. `app` проверяет что это событие произошло на `<button/>`, и в этом случае посылает в `HTTP` объект запроса

3. `HTTP` выполняет запрос и присылает в `app` его результат

4. `app` выбирает из запроса новую ссылку на изображение и шлет в `DOM` сообщение с описание мутации `DOM`

![Message passing](/images/side-effects/message-passing.svg)

Как говорится - осталось только запрограммировать!

## Тысяча и один способ отправить сообщение на JS 

Для начала сделаем самое сложное - выберем инструмент для обмена сообщениями на JS. Как и всегда в `npm` предоставляет огромное множество инструментов для этой задачи:

 - EventEmitter
 - Flux/Redux
 - Observable/Event streams
 - Actors
 - Signals/Cells

Но мы поступим как настоящие программисты и напишем свой! От этого инструмента нам необходимо всего три вещи:

 - уметь подписыватся на прослушивание сообщений
 - уметь отписыватся от них
 - уметь посылать сообщения

В качестве источника вдохновения используем `EventEmitter`, который поставляется в [стандартной библиотеке node.js](https://nodejs.org/api/events.html).

#### EventEmitter на коленке

```javascript
function EventEmitter() {
  var listeners = []; // массив подписчиков
  return {
    addListener: function addListener(listener) {
      listeners.push(listener); // добавляем нового подписчика
    },
    removeListener: function removeListener(listener) {
      listeners = listeners.filter(l => l !== listener); // удаляем подписчика
    },
    emit: function emit(msg) {
      // посылаем всем подписчикам новое сообщение
      listeners.forEach(listener => listener(msg));
    }
  };
};
```

Пример использования:

```javascript
var emmiter = EventEmitter();
emmiter.addListener(event => console.log(event));
emmiter.emit('SOME DATA');
// print  'SOME DATA'
```

## Общая картина

Как мы уже поняли наш мир будет пока небольшим и состоять из всего двух частей - `DOM` и `HTTP`. `app` будет получать от него сообщения и отсылать ему свои. 

Источник сообщений от `world`(окружающего мира) назовем `results` - потому что он будет содержать результаты вычислений из окружающего мира - `HTTP` ответ или `DOM` событие.

Источник сообщений из `app` в `world` назовем `effects` - так как он будет содержать эффекты, которые мы захотим применить во внешнем мире - `HTTP` запрос или мутация `DOM`

Схематично представим так:

![Outside world](/images/side-effects/results-and-effects.svg)

Для удобства разделим `results` и `effects` на два `EventEmitter` - `DOM` и `HTTP`

## Логика приложения

Давайте вспомним, что делает функция `app`:

>`app` подписываается на события `DOM`, проверяет что это событие произошло на `<button/>`, и в этом случае посылает в `HTTP` объект запроса
>

![First interactaction](/images/side-effects/programming-app-logic-1.svg)

Так и запишем:

```javascript
results.DOM.addListener(event => { // подписываемся на все происходяшее в DOM
  if (event.target.tagName === 'BUTTON') { // если событие произошло на <button/>
    effects.HTTP.emit({ url: URL });    // посылаем в HTTP объект запроса
  }
});
```

Далее:

>`app` подписывается на все `HTTP` ответы, и когда приходит новый, выбирает из него новую ссылку на изображение и шлет в `DOM` сообщение с описанием мутации `DOM`
>

![Second interactaction](/images/side-effects/programming-app-logic-2.svg)

```javascript
results.HTTP.addListener(response => {
  effects.DOM.emit({
    selector: 'img', 
    op: 'setAttribute',
    args: ['src', response.json().data.image_url]
  });
});
```

Сведем все вместе:

```javascript
function app(results) {
  var effects = {
    DOM: EventEmitter(),
    HTTP: EventEmitter()
  };
  results.DOM.addListener(event => {
    if (event.target.tagName === 'BUTTON') {
      effects.HTTP.emit({ url: URL });
    }
  });
  results.HTTP.addListener(response => {
    effects.DOM.emit({
      selector: 'img',
      op: 'setAttribute',
      args: ['src', response.json().data.image_url]
    });
  });
  return effects;
}
```

#### Время тестирования

Давайте сразу напишем тесты на наше приложение:
TODO  утилиты

```javascript
var results = {
    DOM: new EventEmitter(),
    HTTP: new EventEmitter()
};
var effects = app(results);
// формируем effects и results
var inputs = [
  {type: 'DOM', event: {target: {tagName: 'BUTTON'}}},
  {type: 'HTTP', event: {json: () => ({data: {image_url: 'SOME_URL'}})}}
];
var expected = [
  {type: 'HTTP', event: {url: URL}},
  {type: 'DOM',
    event: {
      selector: 'img', 
      op: 'setAttribute',
      args: ['src', 'SOME_URL']
    }
  }
];
Object.keys(effects) // подписываемся на effects 
  .forEach(type => effects[type] // и в подписке проверяем выходящие сообщения из app
    .addListener(event => assert.deepEqual({ event, type }, expected.shift())
))
// шлем в results входящие сообщения вызывая тем самым проверку effects
inputs.forEach(({type, event}) => results[type].emit(event));
console.log('All Tests passed');
```

Заметьте, что в тесте не использованы какие то специальные окружения вроде `jsdom`. Эти тесты очень легко запустить и выполнить где угодно. И у нас появился стандартный способ:

 - передать данные из внешнего мира в приложение
 - сравнить результирующие воздействия нашего приложения с ожидаемыми

Но тесты это конечно хорошо, но хотелось бы и всетаки научится запускать наше приложения, а для этого нам необходимо "создать мир", то есть написать функцию `world`

## Build the world!

Вспомним что делает функция `world`.

`DOM` часть `world`
 - слушает события реального `DOM` и пересылает их в `results.DOM`
 - исполняет мутации присланные в `effects.DOM`

![World DOM](/images/side-effects/world-dom-part.svg)

```javascript
// пересылаем клики в results.DOM
document.addEventListener('click', event => results.DOM.emit(event));
// слушаем сообщения из effects.DOM и исполняем их
effects.DOM.addListener('DOM', eff => {
  var el = document.querySelector(eff.selector);
  el[eff.op].apply(el, eff.args);
});
```

`HTTP` часть `world`:
 - слушает сообщения `effects.HTTP` и исполняет их при помощи `fetch`
 - при получении ответа посылает его в `results.HTTP`

![World HTTP](/images/side-effects/world-http-part.svg)

```javascript
effects.HTTP.addListener(eff => fetch(eff.url)
  .then(res => results.HTTP.emit(res))
);
```

И все вместе:

```javascript
function world(effects) {
  var results = {
    DOM: EventEmitter(),
    HTTP: EventEmitter()
  };
  document.addEventListener('click', event => results.DOM.emit(event));
  effects.DOM.addListener('DOM', eff => {
    var el = document.querySelector(eff.selector);
    el[eff.op].apply(el, eff.args);
  });
  effects.HTTP.addListener(eff => fetch(eff.url)
    .then(res => results.HTTP.emit(res))
  );
  return results;
}
```

Таким образом мы:

 - выделили в нашем коде отдельную сущность, которая представляет внешний мир - функция `world`
 - позволили нашему приложению - функции `app` не совершать сайд-эффекты, а только передавать их описания в `world` посредством сообщений
 - позволили `app` получать необходимую информацию из внешнего мира при помощи приема сообщений от `world`

## Змея кусающая себя за хвост

Что мы имеем?

 - `app`: принимает `results` возврашает `effects`. Функция полностью чистая, не совершает никаких сайд-эффектов и на одинаковую входную последовательность сообщений всегда вернет одну и туже выходную последовательность. Описывает логику нашего приложения.
 - `world`: принимает `effects` и возврашает `results`. Исполняет переданные ей сайд-эффекты и возврашает их результаты

Кто-то уже наверно заметил подвох. Если мы попробуем попробуем написать код который принимает `app` и `world` и запускает их, то у нас ничего не получится:

```javascript
function run(app, world) {
  var results = world(effects); // Uncaught ReferenceError: effects is not defined
  var effects = app(results);
}
```

Чтобы выполнить `app` нам нужен результат `world`, а чтобы выполнить `world` нам нужен результат `app`. Тупик?

#### Курица или куринное яйцо? Прокси-курица!

![Use proxy for cycling](/images/side-effects/chicken-or-chicken-egg.jpg)

По сути перед нами классическая "проблема курицы и яйца" - одна из неразрешимых проблем философии.

Однако биологи менее рефлексирующие ребята и благодаря им мы знаем первым была ни курица или яйцо, а эволюционный предшественник курицы.

Давайте своруем этот классный прием у живой природы - создадим третий объект который назовем `proxyEffects`:

```javascript
var proxyEffects = {
  DOM: EventEmmiter(),
  HTTP: EventEmmiter()
};
```

Используем его вместо `effects` при вызове `world`:

```javascript
var results = world(proxyEffects);
var effects = app(results);
```

И затем просто перешлем все сообщения из `effects` в `proxyEffects`, тем самым спровоцировав работу всей нашей системы:

```javascript
Object.keys(effects).forEach(type => {
  effects[type].addListener(eff => proxyEffects[type].emit(eff));
});
```

И получим нашу функцию запуска:

```javascript
function run(app, world) {
  var proxyEffects = {
    DOM: EventEmmiter(),
    HTTP: EventEmmiter()
  };
  var results = world(proxyEffects);
  var effects = app(results);
  var effects = app(proxy);
  Object.keys(effects).forEach(type => {
    effects[type].addListener(eff => proxyEffects[type].emit(eff));
  });
}
```

![Use proxy for cycling](/images/side-effects/chiken-or-egg.svg)

Проверяем - работает!

## Плагины для мира

> А что если бы окружающий нас мир был библиотекой с возможностью подключения плагинов?
>

Мы успешно разделили логику приложения и логику выполнения сайд-эффектов. Но к сожалению наше решение получилось не расширяемым - на текущий момент функция `run` работает только для `DOM` и `HTTP` эффектов. Что-ж пришло время это исправить!

Давайте просто представим наш мир - `world` в виде объекта:

```javascript
var world = {
  HTTP: HTTPWorld, // HTTP плагин обрабатывающий effects.HTTP
  DOM: DOMWorld // DOM плагин обрабатывающий effects.DOM
};
```

Теперь мы сможем легко расширять его и добавлять новые обработчики для новых видов сайд-эффектов - даже для тех которые сейчас мы себе и представить не можем.

Перепишем функцию `run`:

```javascript
function run(app, world) {
  var proxyEffects = {};
  var results = {}
  Object.keys(world).forEach(type => {
    // создаем прокси для каждого плагина
    proxyEffects[type] = EventEmitter();
    // получаем results для каждого плагина
    results[type] = world[type](proxyEffects[type]); 
  });
  var effects = app(results);
  Object.keys(effects).forEach(type => {
    effects[type].addListener(eff => proxyEffects[type].emit(eff));
  });
}
```

## Ваш код немного специфичен

В итоге мы получили довольно общее решение, однако есть одна проблема:

```javascript
function app(results) {
  var effects = {
    DOM: EventEmitter(),
    HTTP: EventEmitter()
  };
  results.DOM.addListener(event => {
    if (event.target.tagName === 'BUTTON') {
      effects.HTTP.emit({ url: URL });
    }
  });
  results.HTTP.addListener(response => {
    effects.DOM.emit({
      selector: 'img',
      op: 'setAttribute',
      args: ['src', response.json().data.image_url]
    });
  });
  return effects;
}
```

Наш код просто ужасно выглядит! Он обладает хорошими свойствами, но читать его почему то не хочется. Функция `app` из 9 строчек выросла до 16.

Давайте попробуем "засахарить" наше решение.

#### Используем индукцию

Что делает функция `app`? Преобразовывает один поток сообщений в другой - причем эти потоки могут быть бесконечными. Однако человеку проще анализировать конечные структуры данных поэтому давай те представим наши потоки сообщений в виде массивов:

```javascript
// results.DOM = [{target: {tagName: 'DIV'}}, {target: {tagName: 'BUTTON'}}]
var effects = app(results)
// effects.HTTP = [{url: URL}]
```

То есть из `[{target: {tagName: 'DIV'}}, {target: {tagName: 'BUTTON'}}]` надо получить `[{url: URL}]`. Так и запишем:

```javascript
results.DOM.forEach(event => {
  if (event.target.tagName === 'BUTTON') { // filtration
    effects.HTTP.push({url: URL});        // transformation
  }
});
```

Посути это тот же код из `app` только вместо `addListener` `forEach`, и `push` вместо `emit`.

Но мы знаем что подобные операции на списками очень легко сокращаются при помощи функций высшего порядка: `map`, `filter`, `reduce` и других:

```javascript
effects.HTTP = results.DOM
    .filter(event => event.target.tagName === 'BUTTON')
    .map(event => ({url: URL}));
```

Тот же самый подход мы можем применить и для нашей реализации `EventEmitter` - надо только реализовать соотвествующие методы.

#### .filter для сообщений

Необходимое поведение метода легко представить в виде диаграммы:

![Filter marble input](/images/side-effects/filter-marble-input.svg)

>`.filter(event => event.target.tagName === 'BUTTON')`
>

![Filter marble output](/images/side-effects/filter-marble-output.svg)

Реализуем:

```javascript
function filter(condition) {
  var result = EventEmitter();
  this.addListener(event => { // слушаем все входящие сообщения
    if (condition(event)) { // сообщение удовлетворяет условию?
      result.emit(event); // тогда передаем его в результирующий EventEmitter
    }
  });
  return result;
}
```

#### .map для сообщений

Диаграмма:

![Map marble input](/images/side-effects/map-marble-input.svg)

>`.map(event => ({url: URL}))`
>

![Map marble output](/images/side-effects/map-marble-output.svg)

Реализация:

```javascript
function map(mapper) {
  var result = EventEmitter();
  // слушаем все входящие сообщения и преобразовывая передаем 
  // в результирующий EventEmitter
  this.addListener(event => result.emit(mapper(event)));
  return result;
};
```

TODO Ссылка на обновленный EventEmitter

#### Засахаренная версия

```javascript
function app(results) {
  return {
    HTTP: results.DOM
      .filter(event => event.target.tagName === 'BUTTON')
      .map(() => ({ url: URL })),
    DOM: results.HTTP
      .map(response => ({
          selector: 'img',
          op: 'setAttribute',
          args: ['src', response.json().data.image_url]
      }))
  };
}
```

В результате код читается намного проще и его стало намного меньше - хотя и все еще больше чем в изначальном варианте.

По традиции дам небольшое домашнее задание: попробуйте реализовать метод `reduce` для EventEmitter, тем самым получив аналог популярного нынче `redux`. CodePen с заготовкой TODO

## NPM нас спасет

В процессе поиска решения мы написали несколько утилит, не связанных с логикой конкретного приложения:

1. EventEmitter c несколькими дополнительными методами
2. Функцию `run` которая запускает наше приложение и позволяет емму коммуницировать с `world`
3. Сам `world` - а точнее модули для обработки и исполнения `DOM` и `HTTP` эффектов

Однако наши утилиты не похожи на настоящие библиотеки - скорее на их игрушечные версии. Это сделано намеренно - чтобы показать саму идею.

Но идеи без хорошей реализации обычно мало кому интересны. Мы могли бы доработать наши утилиты до production-ready состояния - но как всегда все уже сделали за нас.

![I am Cycle](/images/side-effects/i-am-cycle.png)

Уже достаточно долгое время существует проект `Cycle.js`. Он очень похож на наше творение как целями - освободить наши приложения от сайд-эффектов, так и реализацией - разделением логики и приложения от окружающего мира через обмен сообщениями.

Сам автор утверждает, что `Cycle.js` это фреймворк. Однако ~это обман чтобы набрать звездочки~ сам `Cycle.js`  это чуть более 100 строчек кода которые реализуют ту самую функцию `run`. Вся остальная мощь этого "фреймворка" заключена в плагинах(которые называются драйверами). Их существует огромное количество, для самых разных целей - от написания фронтенда, до сервер-сайд приложений и cli утилит.

Однако главным ~из искуств для нас является кино~ для нас является фронтенд. Что может предложить `Cycle.js` для написания фронтенд кода:

1. `Cycle/DOM`- драйвер для работы с DOM. Базируется на одном из самых быстрых Virtual DOM движков - `snabbdom`(он же используется к примеру в `Vue.js`). Также присутсвует возможность серверного рендеринга. Имеет совместимость с `WebComponents` и позволяет использовать для описания разметки как `jsx` так и чистый `javascript`. 

2. `Cycle/HTTP` - драйвер для работы с `HTTP` запросами. Основан на `superagent` - имеет широчайщие возможности для работы с запросами и также поддерживает возможность работы как на сервере так и на клиенте.

3. `Cycle/Canvas`, `Cycle/websocket` - тысячи их! Просто придумайте тип сайд-эффекта и приставьте к нему слово `cycle` и скорее всего вы найдете такую библиотеку на `npm` 

## К черту слова - покажи мне твой код!

Функция `app` написаная при помощи `Cycle.js`:

```javascript
import {h2, button, img} from '@cycle/dom'

function app(results) {
    var HTTPEffects$ = results.DOM
        .select('button')
        .events('click') // выбираем все клики на <button>
        .mapTo({url: URL}); // конвертируем их в описания HTTP запросов
    var response$ = results.HTTP
        .select()
        .flatten() // конвертируем HTTP запросы в данные
        .map(response => ({src: JSON.parse(response.text).data.image_url}));
    var DOMEffects$ = response$  
         // в случае если в обработке HTTP запроса произошла ошибка
         // заменяем ошибку на {isError: true}
        .replaceError(error => response$.startWith({isError: true}))
        .startWith(null)  // до первого клика данных нету
        .map(data => div([ // описываем разметку в зависимости от данных
            h2(['Cats']),
            button(['More Please!']),
            data
              ? (data.isError ? h2(['Just error']) : img({src: data.src}))
              : null
        ]));
    
    return {
        DOM: DOMEffects$,
        HTTP: HTTPEffects$
    };
}
```

Как и положено любому продакшен реди приложению мы добавили в функцию `app` обработку возможных ошибок.
Теперь давайте как и положено напишем тесты для нашего приложения - мы ведь всегда пишем тесты для продакшен кода? :)

#### А теперь покажи мне тесты

#### Немного доброй магии

Хотели бы вы когда-нибудь представить все свое приложение в виде графа(диаграммы) взаимодействий различных участков кода? Если да то `Cycle.js` делает ваши мечты реальностью! Вот такую картину мы получим, применив `CycleDevTools` на нашем приложении:

TODO картинка и более интересные примеры

#### Переиспользуем наше приложение

## Чем хорош `Cycle.js`?

1. Все наше приложение(каким бы большим и сложным оно не было) чистая функция - мы можем тестировать, типизировать, композировать, визуализировать его как обычную функцию.

2. Все наше приложение асинхронно и интерактивно - оно явно описывает реакции на все входящие воздействия. Это позволяет избегать многих проблем и асинхронностью и конкурентностью в UI. Эти проблемы крайне болезненны, несмотря на то что многие даже не знают про их существование.

3. Данные подход действительно _универсален_. "Изучи однажды - используй где угодно". Не смотря на то что данный лозунг был создан в react коммьюнити, он в большей степени применим для `Cycle.js` приложений. Действительно, можно ли изучив React написать бекенд приложение? Или `CLI` утилиту? Сам `Cycle.js` никак не связан с фронтендом и может использоватся для написания любых видов интерактивных программ - главное написать соотвествующий драйвер.  

#### Сколько это будет мне стоить?

Любая технология всегда имеет цену. Даже такая простая и мощная как Cycle.js.

1. Взаимодействия "запрос-ответ"(такие как `HTTP` запросы) не лучшим образом ложатся на концепцию  обмена сообщения,  так как формирования запроса полностью  отделено от обработки ответа. Это выглядит довольно не привычно и может отпугивать
TODO пример

2. Так же изза отделения формирования запроса от ответа, возникает проблема понять где же ответ на наш запрос. Расмотрим пример:
TODO пример с HTTP

Однако мы можем легко решить эту проблему добавив нашему запрос некотрую метку, по которой затем мы сможем отфильтровать необходимы нам результат:
TODO HTTP category

Однако как всегда все уже сделано за нас - внутрь большей части драйверов встроена возможность автоматической изоляции. Выглядит это так:
TODO изолят

Просто заворачиваем наш компонент в функцию `isolate` и она автоматически будет передавать туда только сообщения которые относятся к нашему компоненту.

3. Это не фреймворк - `Cycle.js` не имеет единого способа строить архитектуру - за исключением того что функция приложения должна быть без сайд-эффектов. К примеру столь важный вопрос для SPA "Как работать со состоянием приложения?" - имеет множество ответов в мире `Cycle.js`. TODO ссылки. То есть при написании приложения все еще необходимо думать - что очень печалит меня как программиста.

## Конец?

К счастью обмен сообщениями это не единственный способ контролировать сайд-эффекты. Существет еще минимум два:

- Монады:

  + [`IO monad` in Haskell](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/IO)
  + [`Task` in Elm](https://guide.elm-lang.org/architecture/effects/)
  + [`Eff monad` in PureScript](http://www.purescript.org/learn/eff/)
  + [`Task` in `redux-loop`](https://github.com/redux-loop/redux-loop)
  + [`Task` in `fun-task`](https://github.com/rpominov/fun-task/blob/master/examples/io/1.js)
- Продолжения:

  + [Algebraic effects in Ocaml](http://kcsrk.info/ocaml/multicore/effects/2015/05/27/more-effects/)
  + [Algebraic effects in Eff](http://www.eff-lang.org/handlers-tutorial.pdf)
  + [`Effect` in `redux-saga`](http://yelouafi.github.io/redux-saga/docs/basics/DeclarativeEffects.html)

Все эти три способа - обмен сообщениями, монады, продолжения - объединяет то, что все они созданы для абстракции `control-flow` программы. `Control-flow` это скелет нашей программы, ее базис - поэтому эти способы возникают при решении большей части проблем в Computer Science - не только при решении проблемы сайд-эффектов. Однако это тема для отдельной большой статьи. 



