---
title: Цикл событий Node.js, Таймеры и `process.nextTick()
layout: docs.hbs
---

# Цикл событий Node.js, Таймеры и `process.nextTick()`

## Что такое Цикл Событий?

Несмотря на однопоточность, Node.js может выполнять неблокирующие I/O операции
при помощи цикла событий, который делегирует исполнение подобных операций
ядру операционной системы при любой возможности.

Since most modern kernels are multi-threaded, they can handle multiple
operations executing in the background. When one of these operations
completes, the kernel tells Node.js so that the appropriate callback
may be added to the **poll** queue to eventually be executed. We'll explain
this in further detail later in this topic.

Поскольку большинство современных ядер являются многопоточными, они
способны исполнять сразу несколько операций одновременно. Когда одна
из таких операций заканчивается, ядро сообщает Node.js, что соответствующая
callback-функция ([функция обратного вызова][https://ru.wikipedia.org/wiki/Callback_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)])  может быть добавлена в очередь **poll** этапа,
чтобы в конце концов она была исполнена. Далее мы объясним этот процесс более подробно.

## Event Loop Explained
## Цикл событий в деталях

When Node.js starts, it initializes the event loop, processes the
provided input script (or drops into the [REPL][], which is not covered in
this document) which may make async API calls, schedule timers, or call
`process.nextTick()`, then begins processing the event loop.

Когда запускается Node.js, он инициализирует цикл событий, обрабатывает
предоставленный JS скрипт, который может сделать асинхронные API вызовы,
установить таймеры или вызвать `process.nextTick()`, затем Node.js начинает
обработку цикла событий.

Следующая диаграмма показывает упрощенное представление последовательньсти
операций цикла событий.

```
   ┌────────────────────────────┐
┌─>│           timers           |
|  |           таймеры          │
│  └─────────────┬──────────────┘
│  ┌─────────────┴──────────────┐
│  │     pending callbacks      |   
│  | ожидающие callback-функции |
│  └─────────────┬──────────────┘
│  ┌─────────────┴──────────────┐
│  │       idle, prepare        | 
│  |    ожидание, подготовка    |
│  └─────────────┬──────────────┘
│  ┌─────────────┴──────────────┐      ┌───────────────┐      
│  │           poll             │      │   incoming:   │
│  │           опрос            │<─────┤  connections, │      
│  └─────────────┬──────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐       └───────────────┘
│  │           check           |
│  |          проверка         |
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      |
   │      callback-функции     |
   |      "close" событий      |
   └───────────────────────────┘
```

*каждая рамка — этап в цикле событий* 

Each phase has a FIFO queue of callbacks to execute. While each phase is
special in its own way, generally, when the event loop enters a given
phase, it will perform any operations specific to that phase, then
execute callbacks in that phase's queue until the queue has been
exhausted or the maximum number of callbacks has executed. When the
queue has been exhausted or the callback limit is reached, the event
loop will move to the next phase, and so on.

На каждом этапе есть своя FIFO очередь callback-функций, которые
нужно обработать. Хотя каждый этап имеет свои отличия, но в общем, когда
цикл событий переходит на очередной этап, он исполняет все специфические
операции связанные с этим этапом, а затем обрабатывает callback-функции в его очереди.
Обработка функций происходит до тех пор, пока очередь не окажется
пустой или не будет обработано максимально разрешенное количество таких функций.
После этого цикл событий перейдет на следующий этап и так далее.

Since any of these operations may schedule _more_ operations and new
events processed in the **poll** phase are queued by the kernel, poll
events can be queued while polling events are being processed. As a
result, long running callbacks can allow the poll phase to run much
longer than a timer's threshold. See the [**timers**](#timers) and
[**poll**](#poll) sections for more details.

Любая из этих операций может запланировать _дополнительные_ операции. 
Новые события, обработанные на этапе **опроса**, добавляются ядром в очередь.
а новые события этого этапа могут быть отправлены в очередь, пока поллирующие события проходя обработку.  
В результате, продолжительно работающие функции обратного вызова, могут затянуть  работу полл этапа 
на время, которое заметно превышает минимально допустимое время таймера.

_**Замечание:** Существует небольшое различие между релизациями в Windows и
Unix/Linux, но для данной демонстрации это не важно. Наиболее важные части здесь. 
На самом деле существует семь или восемь шагов, но значимые, которые использует Node.js,
те что на схеме выше._


## Phases Overview
## Этапы

* **timers**: this phase executes callbacks scheduled by `setTimeout()`
 and `setInterval()`.
* **pending callbacks**: executes I/O callbacks deferred to the next loop
 iteration.
* **idle, prepare**: only used internally.
* **poll**: retrieve new I/O events; execute I/O related callbacks (almost
 all with the exception of close callbacks, the ones scheduled by timers,
 and `setImmediate()`); node will block here when appropriate.
* **check**: `setImmediate()` callbacks are invoked here.
* **close callbacks**: some close callbacks, e.g. `socket.on('close', ...)`.

* **timers** ( _таймеры_ ): на этом этапе исполняются callback-функции запланированные `setTimeout()`
 и `setInterval()`.
* **pending callbacks** ( _ожидающие callback-функции_ ): тут исполняются callback-функции I/O, исполнение которых было отложено на следующую итерацию цикла. 
* **idle, prepare** ( _ожидание, подготовка_ ): выполняет внутренние работы.
* **poll** ( _опрос_ ): получает новые I/O события; исполняет callback-функции, которые
имеют отношение к I/O операциям (практически все, за исключением тех функций, что должны
сработать при событии "close", тех, что были запланированны таймерами и `setImmediate()`); node will block here when appropriate.
* **check** ( _проверка_ ): здесь будут вызваны callback-функции установленные посредством `setImmediate()`.
* **close callbacks** ( _callback-функции "close"_ ): callback-функции связанные с "close" событиями. Например: `socket.on('close', ...)`.


Between each run of the event loop, Node.js checks if it is waiting for
any asynchronous I/O or timers and shuts down cleanly if there are not
any.

После каждого повтора цикла событий, Node.js проверяет, не ждет ли цикл
ответ от какогой-либо асинхронной I/O операции или от установленного таймера.
Если таких нет, Node.js завершит свою работу.


## Phases in Detail
## Этапы в подробностях

### timers
### таймеры

A timer specifies the **threshold** _after which_ a provided callback
_may be executed_ rather than the **exact** time a person _wants it to
be executed_. Timers callbacks will run as early as they can be
scheduled after the specified amount of time has passed; however,
Operating System scheduling or the running of other callbacks may delay
them.

Таймер определяет **минимально допустимое значение** _после которого_ указанная
callback-функция _может быть исполнена_, а не **конкретное** время, когда
программист _хочет, чтобы эта функция была исполнена_. Callback-функции,
установленные в таймерах, будут исполнены как можно быстрее, после того
как прошло минимально допустимое значение таймера; однако, планировщик
ОС или обработка друих callback-функций может задержать их.

_**Замечание**: Технически, [**poll** этап](#poll) контролирует, когда
таймеры, будут обработанны._

For example, say you schedule a timeout to execute after a 100 ms
threshold, then your script starts asynchronously reading a file which
takes 95 ms:

Допустим, вы установили таймер, который должен сработать не раньше
чем через 100 мс, затем ваш скрипт начинает асинхронно считывать файл,
что займет 95 мс:

```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // Представьте, что эта операция займет 95 мс
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// запустить функция someAsyncOperation, которая займет 95 мс
someAsyncOperation(() => {
  const startCallback = Date.now();

  // делай какую-нибудь работу на протяжении 10 мс
  while (Date.now() - startCallback < 10) {
    // ничего не делай
  }
});
```

When the event loop enters the **poll** phase, it has an empty queue
(`fs.readFile()` has not completed), so it will wait for the number of ms
remaining until the soonest timer's threshold is reached. While it is
waiting 95 ms pass, `fs.readFile()` finishes reading the file and its
callback which takes 10 ms to complete is added to the **poll** queue and
executed. When the callback finishes, there are no more callbacks in the
queue, so the event loop will see that the threshold of the soonest
timer has been reached then wrap back to the **timers** phase to execute
the timer's callback. In this example, you will see that the total delay
between the timer being scheduled and its callback being executed will
be 105ms.

Когда цикл событий переходит на этап **опроса**, его очередь пуста
(`fs.readFile()` еще не завершился), так что он будет ждать, пока не 
подойдет минимальное допустимое значение ближайшего таймера. Пока он
ждет, прошло 95 мс. `fs.readFile()` завершает чтение файла, его
callback-функция помещена в очередь **опроса** и исполнена. Исполнение
этой функции занимает 10 мс. Теперь очередь пуста, а цикл событий
видит, что минимально допустимое значение ближайшего таймера уже прошло.
Он возвращается обратно на этап **таймеров**, чтобы исполнить соответсвующую
callback-функцию. В данном примере, вы увидите, что общее время задержки между
установкой таймера и срабатыванием его callback-функции будет равно 105 мс.

Note: To prevent the **poll** phase from starving the event loop, [libuv][]
(the C library that implements the Node.js
event loop and all of the asynchronous behaviors of the platform)
also has a hard maximum (system dependent) before it stops polling for
more events.

Замечание: Чтобы предотвратить "голодание" цикла событий из-за работы этапа **опроса**, 
в [libuv][] (Библиотека Си, посредством которой в Node.js реализован
цикл событий и все асинхронные операции платформы) "вшито" ограничение 
(зависимое от системы), достигнув которое он прекращает проверку на наличие новых событий.


### pending callbacks
### ожидающие callback-функции

This phase executes callbacks for some system operations such as types
of TCP errors. For example if a TCP socket receives `ECONNREFUSED` when
attempting to connect, some \*nix systems want to wait to report the
error. This will be queued to execute in the **pending callbacks** phase.

Этот этап исполняет callback-функции для некоторых системных операций,
таких как виды TCP ошибок. Например, если TCP сокет получает `ECONNREFUSED`
при попытке подключения, некоторые \*nix системы хотят подождать, прежде
чем сообщить об ошибке. Их callback-функции будут помещены в очередь этапа **ожидающие callback-функции**.

### poll
### опрос

The **poll** phase has two main functions:
Этап **poll** имеет две основные функции:

1. Calculating how long it should block and poll for I/O, then
2. Processing events in the **poll** queue.

1. Опеределить как долго следует блокировать цикл событий и "опрашивать" I/O, затем
2. Обработать события в очереди **опроса**.

When the event loop enters the **poll** phase _and there are no timers
scheduled_, one of two things will happen:

Когда цикл событий переходит на этап **опроса**, _а установленные таймеры
отсутсвуют_, случится одна из двух вещей:

* _If the **poll** queue **is not empty**_, the event loop will iterate
through its queue of callbacks executing them synchronously until
either the queue has been exhausted, or the system-dependent hard limit
is reached.

* _Если очередь опроса **непустая**_, цикл событий начнет синхронно
обрабатывать очередь callback-функций до тех пор, пока они
не закончатся, или пока цикл событий не обработает максимально допустимое
количество callback-функций.

* _If the **poll** queue **is empty**_, one of two more things will
happen:
  * If scripts have been scheduled by `setImmediate()`, the event loop
  will end the **poll** phase and continue to the **check** phase to
  execute those scheduled scripts.

  * If scripts **have not** been scheduled by `setImmediate()`, the
  event loop will wait for callbacks to be added to the queue, then
  execute them immediately.

* _Если очередь опроса **пустая**_, одна из еще двух вещей 
случится:
  * Если есть скрипты установленные `setImmediate()`, цикл событий
  завершит этап **опроса** и перейдет к этапу **проверки**, чтобы исполнить эти 
  скрипты.

  * Если скриптов установленных `setImmediate()` **нет**, цикл событий
  будет ожидать размещения новых callback-функций в свою очередь, затем
  исполнит их.
  
Once the **poll** queue is empty the event loop will check for timers
_whose time thresholds have been reached_. If one or more timers are
ready, the event loop will wrap back to the **timers** phase to execute
those timers' callbacks.

Когда очередь **опроса** окажется пустой, цикл событий проверит, не подошло
ли время какого-нибудь таймера. Если один или более таймеров готовы,
цикл событий вернется назад на этап **таймеров**, чтобы исполнить их callback-функции.


### check
### проверка

This phase allows a person to execute callbacks immediately after the
**poll** phase has completed. If the **poll** phase becomes idle and
scripts have been queued with `setImmediate()`, the event loop may
continue to the **check** phase rather than waiting.

Этот этап позволяет программисту запустить callback-функции сразу же после
окончания окончания этапа **опроса**. Если этап **опроса** перешел в режим
ожидания, а какие-то скрипты были помещенны в очередь посредством `setImmediate()`,
цикл событий может перейти на этап **проверки**, вместо того чтобы ждать.

`setImmediate()` is actually a special timer that runs in a separate
phase of the event loop. It uses a libuv API that schedules callbacks to
execute after the **poll** phase has completed.

На самом деле `setImmediate()` это специальный таймер, который срабатывает на
отдельном этапе цикла событий. Он использует API libuv для установки
callback-функций, которые будут исполнены после завершения этапа **опроса**. 

Generally, as the code is executed, the event loop will eventually hit
the **poll** phase where it will wait for an incoming connection, request,
etc. However, if a callback has been scheduled with `setImmediate()`
and the **poll** phase becomes idle, it will end and continue to the
**check** phase rather than waiting for **poll** events.

В общем, по мере исполнения кода, цикл событий дойдет до этапа **опроса**,
на котором он будет ожидать входящие подключения, запросы и т.п. операции.
Однако, если при помощи `setImmediate()` была установлена callback-функция,
а этап **опроса** перешел в режим ожидания, он будет завершен, цикл событий
перейдет на этап **проверки**, вместо того чтобы ожидать события **опроса**.

### close callbacks
### callback-функции событий 'close' 

If a socket or handle is closed abruptly (e.g. `socket.destroy()`), the
`'close'` event will be emitted in this phase. Otherwise it will be
emitted via `process.nextTick()`.

Если сокет или обработка "оборванна" (например, посредством `socket.destroy()`),
событие `'close'` будет отправленно на этом этапе. Иначе оно будет отправленно
посредством `process.nextTick()`.

## `setImmediate()` vs `setTimeout()`
## `setImmediate()` против `setTimeout()`

`setImmediate()` and `setTimeout()` are similar, but behave in different
ways depending on when they are called.
`setImmediate()` и `setTimeout()` похожи, но ведут себя поразному в зависимости
от того когда они вызваны.

* `setImmediate()` is designed to execute a script once the
current **poll** phase completes.
* `setTimeout()` schedules a script to be run after a minimum threshold
in ms has elapsed.
* `setImmediate()` спроектирован для исполнения скрипта после завершения текущего этапа **опрос**.
current **poll** phase completes.
* `setTimeout()` устанавливает скрипт, который будет исполнен по прошествию минимально допустимого значения в миллисекундах.

The order in which the timers are executed will vary depending on the
context in which they are called. If both are called from within the
main module, then timing will be bound by the performance of the process
(which can be impacted by other applications running on the machine).

Порядок исполнения таймеров будет зависить от контекста, в котором они вызваны.
Если оба вызваны в основном модуле, тогда время срабатывания будет зависеть
от производительности процесса (которая зависит от исполняющихся)

For example, if we run the following script which is not within an I/O
cycle (i.e. the main module), the order in which the two timers are
executed is non-deterministic, as it is bound by the performance of the
process:

Например, если мы запустим нижеследующий скрипт, который расположен
вне I/O цикла, а в основном модуле, порядок в котором таймеры будут
исполнены не определен, поскольку он зависит от производительности процесса:


```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```

```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```

However, if you move the two calls within an I/O cycle, the immediate
callback is always executed first:
Однако, если вы разместита оба вызова внутри I/O цикла, callback-функция,
установленная в setImmediate(), всегда исполняется первой:

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```

Основное преимущество `setImmediate()` перед `setTimeout()` заключается в том,
что внутри I/O цикла `setImmediate()` всегда исполнится в первую очередь вне зависимости от того,
сколько таймеров было установлено.

## `process.nextTick()`

### Understanding `process.nextTick()`
### Объяснение `process.nextTick()`

You may have noticed that `process.nextTick()` was not displayed in the
diagram, even though it's a part of the asynchronous API. This is because
`process.nextTick()` is not technically part of the event loop. Instead,
the `nextTickQueue` will be processed after the current operation is
completed, regardless of the current phase of the event loop. Here, 
an *operation* is defined as a transition from the
underlying C/C++ handler, and handling the JavaScript that needs to be
executed.

Вы могли заметить, что `process.nextTick()` не обозначен на диаграмме, хотя
он является частью асинхронного API. Причина в том, что `process.nextTick()`
технически не является частью цикла событий. Очередь `nextTickQueue` будет обработана после
завершения текущей операции, безотносительно того на каком этапе находится цикл событий.
**Операцией** здесь означает переход от лежащего в основе обработчика C/C++ к
JS коду, который необходимо исполнить.

Looking back at our diagram, any time you call `process.nextTick()` in a
given phase, all callbacks passed to `process.nextTick()` will be
resolved before the event loop continues. This can create some bad
situations because **it allows you to "starve" your I/O by making
recursive `process.nextTick()` calls**, which prevents the event loop
from reaching the **poll** phase.

Обратимся к нашей диаграмме. Не важно на каком этапе цикла событий и в какой
момент вызван `process.nextTick()`, все callback-функции, переданные этому
методу, будут исполнены до перехода на следующий этап. **Здесь кроется опасность**,
можно так зациклить вызовы `process.nextTick()`, что они не позволят циклу событий
перейти на этап **опроса**. Это, в свою очередь, приведет к "голоданию" I/O операций. 


### Why would that be allowed?
### Зачем это разрешено?

Why would something like this be included in Node.js? Part of it is a
design philosophy where an API should always be asynchronous even where
it doesn't have to be. Take this code snippet for example:

Зачем подобное включено в Node.js? Часть причины заключается в философии
дизайна, которая гласит, что API всегда должен оставаться асинхронным, даже
в там, где он может и не быть таким. Возьмем следующий пример:

```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
                            new TypeError('argument should be string'));
}
```

The snippet does an argument check and if it's not correct, it will pass
the error to the callback. The API updated fairly recently to allow
passing arguments to `process.nextTick()` allowing it to take any
arguments passed after the callback to be propagated as the arguments to
the callback so you don't have to nest functions.

Эта функция проверяет аргумент, если аргумент неверный, она передаст
ошибку в callback-функцию. Чтобы избежать излишней вложенности функций,
API `process.nextTick()` был недавно обновлен: метод принимает любое количество
аргументов после callback-функции, которые будут переданы в эту самую callback-функцию.


What we're doing is passing an error back to the user but only *after*
we have allowed the rest of the user's code to execute. By using
`process.nextTick()` we guarantee that `apiCall()` always runs its
callback *after* the rest of the user's code and *before* the event loop
is allowed to proceed. To achieve this, the JS call stack is allowed to
unwind then immediately execute the provided callback which allows a
person to make recursive calls to `process.nextTick()` without reaching a
`RangeError: Maximum call stack size exceeded from v8`.

Мы передаем ошибку обратно пользавателю, но только **после** того как отработал
его оставшийся код. Применя `process.nextTick()` мы гарантируем, что `apiCall()`
всегда исполнит свою callback-функцию *после* кода пользователя, но **до** того
как циклу событий будет разрешено перейти к следующему этапу.



This philosophy can lead to some potentially problematic situations.
Take this snippet for example:

Эта философия может привести к потенциально проблемным ситуациям.
Возьмем следующий отрывок кода:

```js
let bar;

// this has an asynchronous signature, but calls callback synchronously
// у функции асинхронная сигнатура, но callback-функция вызывается синхронно
function someAsyncApiCall(callback) { callback(); }

// the callback is called before `someAsyncApiCall` completes.
// callback-функция вызвана до завершения `someAsyncApiCall`
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  // поскольку someAsyncApiCall завершилась, перенная bar еще не получила никакое значение
  console.log('bar', bar); // undefined
});

bar = 1;
```

The user defines `someAsyncApiCall()` to have an asynchronous signature,
but it actually operates synchronously. When it is called, the callback
provided to `someAsyncApiCall()` is called in the same phase of the
event loop because `someAsyncApiCall()` doesn't actually do anything
asynchronously. As a result, the callback tries to reference `bar` even
though it may not have that variable in scope yet, because the script has not
been able to run to completion.

?? may not have that variable in scope yet ?? могут не иметь ??

Программист определил `someAsyncApiCall()` с асинхронной сигнатурой, но
на самом дела она работает синхронно. Когда её вызовут, callback-функция,
переданная в `someAsyncApiCall()`, будет вызвана на том же этапе цикла событий,
так как `someAsyncApiCall()` не делает ничего асинхронного. В итоге, callback-функция
попытается обратиться к `bar`, хотя её значение ещё и не находится в поле видимости 
(потому что обработка кода не завершилась).

By placing the callback in a `process.nextTick()`, the script still has the
ability to run to completion, allowing all the variables, functions,
etc., to be initialized prior to the callback being called. It also has
the advantage of not allowing the event loop to continue. It may be
useful for the user to be alerted to an error before the event loop is
allowed to continue. Here is the previous example using `process.nextTick()`:

Резместив callback-функцию в `process.nextTick()`, код будет обработан до конца.
Это позволит завершить инициализацию всех переменных, функций и т.п., до вызова
callback-функции. Еще одно преимущество: цикл событий не продолжит свою работу. что
может быть использовано для опевещения пользователя об ошибке.

```js
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```

Here's another real world example:

```js
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```

When only a port is passed, the port is bound immediately. So, the
`'listening'` callback could be called immediately. The problem is that the
`.on('listening')` callback will not have been set by that time.

To get around this, the `'listening'` event is queued in a `nextTick()`
to allow the script to run to completion. This allows the user to set
any event handlers they want.

## `process.nextTick()` vs `setImmediate()`

We have two calls that are similar as far as users are concerned, but
their names are confusing.

* `process.nextTick()` fires immediately on the same phase
* `setImmediate()` fires on the following iteration or 'tick' of the
event loop

In essence, the names should be swapped. `process.nextTick()` fires more
immediately than `setImmediate()`, but this is an artifact of the past
which is unlikely to change. Making this switch would break a large
percentage of the packages on npm. Every day more new modules are being
added, which means every day we wait, more potential breakages occur.
While they are confusing, the names themselves won't change.

*We recommend developers use `setImmediate()` in all cases because it's
easier to reason about (and it leads to code that's compatible with a
wider variety of environments, like browser JS.)*

## Why use `process.nextTick()`?

There are two main reasons:

1. Allow users to handle errors, cleanup any then unneeded resources, or
perhaps try the request again before the event loop continues.

2. At times it's necessary to allow a callback to run after the call
stack has unwound but before the event loop continues.

One example is to match the user's expectations. Simple example:

```js
const server = net.createServer();
server.on('connection', (conn) => { });

server.listen(8080);
server.on('listening', () => { });
```

Say that `listen()` is run at the beginning of the event loop, but the
listening callback is placed in a `setImmediate()`. Unless a
hostname is passed, binding to the port will happen immediately. For
the event loop to proceed, it must hit the **poll** phase, which means
there is a non-zero chance that a connection could have been received
allowing the connection event to be fired before the listening event.

Another example is running a function constructor that was to, say,
inherit from `EventEmitter` and it wanted to call an event within the
constructor:

```js
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);
  this.emit('event');
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

You can't emit an event from the constructor immediately
because the script will not have processed to the point where the user
assigns a callback to that event. So, within the constructor itself,
you can use `process.nextTick()` to set a callback to emit the event
after the constructor has finished, which provides the expected results:

```js
const EventEmitter = require('events');
const util = require('util');

function MyEmitter() {
  EventEmitter.call(this);

  // use nextTick to emit the event once a handler is assigned
  process.nextTick(() => {
    this.emit('event');
  });
}
util.inherits(MyEmitter, EventEmitter);

const myEmitter = new MyEmitter();
myEmitter.on('event', () => {
  console.log('an event occurred!');
});
```

[libuv]: http://libuv.org
[REPL]: https://nodejs.org/api/repl.html#repl_repl
