# Asynchronous programming in NodeJS

### Asynchronous programming in JavaScript as of today

Здесь перечислено большинство способов упорядочения асинхронного кода 
(на самом деле, все эти способы, кроме генераторов, сводятся к callbacks)

* callbacks
* async.js
* promises
* async/await
* generators/yield
* observable\events
* functor + chaining + composition
* for await + Symbol.asyncIterator

Все вышеперечисленные способы работы с асинхронностью группируются следующим образом:

* async.js > callbacks
* async/await > promises
* events > observable
* functor + chaining + composition

### Callbacks

Контракты работы с асинхронным кодом через коллбек в ЖС могут быть абсолютно любыми. 
Если мы передаем в качестве аргумента в какую-то функцию коллбек мы его можем передавать как первым аргументом, 
так и последним, так и где-то по середине. Также происходит и с возвращаемыми результатами.

Чтобы весь наш код упорядочить и сделать более-менее однородным по контрактам приняли соглашение: **callback-last, error-first, but you can implement hell easily**

```js
(callback) => callback(data)
    
(...args, callback) => callback(err, data)
```

Если нам требуется вызывать несколько функций последовательно, то асинхронный код на коллбеках будет выглядеть вот так:

```js
readConfig('myConfig', (e, data) => {
    query('select * fron some_table', (e, data) => {
        httpGet('http://some_web_cite', (e, data) => {
            readFile('README.md', (e, data) => {
                ...etc
            });
        });
    });
});
```

Как можно заметить по данному примеру у нас возникает очень большая вложенность и код становится плохо читабельным.
Это можно довольно-таки легко разрешить, если убрать каждый вызов лямбды в отдельную функцию и сделать вызов линейным.

```js
readConfig('myConfig');

function readConfig(fileName) {
    ...; query('select _ from some_table')
}

function query(statement) {
    ...; httpGet('http://some_web_cite');
}
```

Но такой метод все ещё не дает нам ожидаемо хороший результат ввиду того, что каждая функция должна знать о цепочке выполнения
(какая функция за какой должна следовать в последовательности вызовов). Мы не можем управлять последовательностью вызовов и каким-либо образом контролировать её.

Для решения данной проблемы придумали различные утилиты, которые могут упрощать нам жизнь.

### Library async.js or analogues

Идея такова, что мы передаем первым аргументом массив (или объект) функций каждая из которых имеет контракт, который принят для всех асинхронных функций.
У нас есть массив асинхронных функций, следовательно нам осталось написать метод, который пройдется по данному массиву (либо последовательно, либо параллельно)
и сведет всю цепочку вызовов в одно место.

> Функция является асинхронной, только если внутри нее имеется вызов другой асинхронной функции,
> обращение к вводу/выводу, вызов таймеров или любые другие вызовы, которые позволяют разорвать
> синхронное выполнение кода

```js
async.method(
    [... (data, cb) => cb(err, result) ...],
    (err, result) => {}
);
```

> Если итерируемся по объекту, то на выходе получаем объект, если по массиву - массив.

Use **callback-last, error-first, define functions separately, descriptive names, hell remains**

Такой способ работы с асинхронностью нас не полностью спасает. Он упрощает работу с коллбеками,
но если нам необходимо сделать вложенность, то читабельность кода все равно сильно упадет со временем

Ещё один способ работы с асинхронностью это события.

### Events

```js
const ee = new EventEmitter();
const f1 = () => ee.emit('step2');
const f2 = () => ee.emit('step3');
const f3 = () => ee.emit('done');
ee.on('step1', f1.bind(null, par));
ee.on('step2', f2.bind(null, par));
ee.on('step3', f3.bind(null, par));
ee.on('done', () => console.log('done'));
ee.emit('step1');
```

Данный код совмещает в себе все самые неудобные моменты работы с коллбеками и асинхронностью как таковой. 
Но все же имеет право на жизнь.

Следующее что придумали разработчики это обернуть асинхронность в объект и вместо того чтобы возвращать данные, мы возвращаем объект
который имеет подписку на то, что данные придут. Таким образом получился Promise

### Promise

```js
new Promise((resolve, reject) => {
    resolve(data);
    reject(new Error(...));
})
    .then(result => {}, reason => {})
    .catch(err => {});
```

Идея состоит в том, что мы из асинронной функции возвращаем нечто, что символизирует результат, но данных внутри там ещё нет. 
Но потом на данное нечто можно подписаться. Таким образом мы отдельно подписываемся на данные (.then) и на ошибки (.catch).
Таким образом у нас получается 2 различных варианта промиса. Промис с данными и промис с ошибками (reason - также является результатом ошибки)

promise Sequential example: 

```js
Promise.resolve()
    .then(readConfig.bind(null, 'myConfig'))
    .then(query.bind(null, 'select * from some_table'))
    .then(httpGet.bind(null, 'htpp://some_web_cite'))
    .catch(err => console.log(err.message))
    .then(readFile.bind(null, 'README.md'))
    .catch(err => console.log(err.message))
    .then(data => {
        console.dir({ data });
    })
```

Promise Parallel example:

```js
Promise.all([
    readConfig('myConfig'),
    doQuery('select * from some_table'),
    httpGet('http://some_web_cite'),
    readFile('README.md')
]).then(data => {
    console.log('Done');
    console.dir({ data });
}).catch(err => {
    console.log(err.message)
})
```

Promise Mixed: parallel/sequential example:

```js
Promise.resolve()
    .then(readConfig.bind(null, 'myConfig'))
    .then(() => Promise.all([
        query('select * from some_table'),
        httpGet('http://some_web_cite')
    ]))
    .then(readFile.bind(null, 'README.md'))
    .then(data => {
        console.log('Done')
        console.dir({ data })
    })
```

**Separated control flow for success and fail, hell remains for complex parallel/sequential code**

Хоть мы и побороли сложности составления асинхронных запросов, но синтаксис все ещё оставляет желать лучшего. 
Поэтому рассмотрим следующий метод

### Async/await

Функции, которые будут возвращать нам значения асинхронно должны быть объявлены с ключевым словом `async`.
Такая функция может возвращать либо промис, либо `await` промис. Если асинхронная функция возвращает какое-либо значение,
то это значение будет обернуто в `Promise`.

```js
async function f() {
    return await new Promise(...);
}

f.then(console.log()).catch(console.error())
```

**Promises under the hood, control-flow separated, hell remains, perfomance reduced**

Теперь мы попробуем просто отталкиваться от синтаксиса и придумать минималистичный синтаксис, совместимый с JS и максимально читаемый. 
Это приводит нас к functors.

### Functor + Chaining + composition

```js
const c1 = chain()
    .do(readConfig, 'myConfig')
    .do(doQuery, 'select * from some_table')
    .do(httpGet, 'http://some_web_cite')
    .do(readFile, 'README.md');

c1();
```

В данном примере мы вызываем `chain` и этот `chain` может вернуть нам объект с методами `do`. Таким образом мы создаем цепочку вызовов.
И этот объект, который возвращается из чейна, будет накапливать все вызовы внутри какой-то структуры данных (своей коллекции), в которой он получает имя функции и её аргументы.

Реализация данного чейна будет выглядеть вот так:

```js
function chain(prev = null) {
    const cur = () => {
        if (cur.prev) {
            cur.prev.next = cur;
            cur.prev();
        } else {
            cur.forward();
        }
    };
    cur.prev = prev;
    cur.fn = null;
    cur.args = null;
    cur.do = (fn, ...args) => {
        cur.fn = fn;
        cur.args = args;
        return chain(cur);
    };
    cur.forward = () => {
        if (cur.fn) cur.fn(cur.args, () => {
            if (cur.next) cur.next.forward();
        });
    };
    return cur;
}
```

Данный чейн представляет собой функциональный объект с замыканием внутри. 

### Problems

We have several problems with callbacks, async.js, Promise, async/await:

* Nesting and syntax;
* Different contracts;
* Not cancellable, no timeouts;
* Complexity and Perfomance;

### Some tricks to resolve these problems

1. Add timeout to any function:

```js
const fn = par => {
    console.log('Function called, par: ' + par);
}

const fn100 = timeout(100, fn);
const fn200 = timeout(200, fn);

setTimeout(() => {
    fn100('first');
    fn200('second');
}, 150)

function timeout(msec, fn) {
    let timer = setTimeout(() => {
        if (timer) console.log('Function timedout');
        timer = null;
    }, msec);
    return (...args) => {
        if (timer) {
            timer = null;
            fn(...args);
        }
    };
}
```

2. Make function cancelable

```js
const fn = par => {
    console.log('Function called, par: ' + par);
};

const f = cancellable(fn);

f('first');
f.cancel();
f('second');

const cancellable = fn => {
    const wrapper = (...args) => {
        if (fn) return fn(...args);
    };
    wrapper.cancel = () => {
        fn = null;
    };
    return wrapper;
};
```

3. More wrappers!

```js
const f1 = timeout(1000, fn);
const f2 = cancellable(fn);
const f3 = once(fn);
const f4 = limit(10, fn);
const f5 = throttle(10, 1000, fn);
const f6 = debounce(1000, fn);
const f7 = utils(fn)
    .limit(10)
    .throttle(10, 100)
    .timeout(1000);
```

4. Promisify and Callbackify

```js
const promise = promisify(asyncFucnion);
promise.then(...).catch(...);

const callback = callbackify(promise);
callback((err, value) => { ... });
```

