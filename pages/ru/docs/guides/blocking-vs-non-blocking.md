---
title: Блокирующие и Неблокирующие Вызовы
layout: docs.hbs
---

# Обзор Блокирующих и Неблокирующих Вызовов

Этот обзор рассматривает разницу между **блокирующими** и **неблокирующими** вызовами в Node.js. Он ссылается на цикл событий (event loop) и библиотеку libuv, однако предварительное знание этих тем не требуется. Предполагается, что читатели имеют базовое понимание JavaScript и паттерна обратных вызовов (callback) в Node.js.

> Обозначение "I/O" (Ввод/Вывод) в первую очередь ссылается на взаимодействие с системным диском и сетью при поддержке [libuv](https://libuv.org/).

## Блокирование

О **блокировании** говорят, когда выполнение JS кода в Node.js приостановлено до тех пор, пока не завершится работа сторонней операции (например, чтение какого-нибудь файла). Так происходит потому, что цикл событий не может продолжить исполнение JavaScript, так как работает **блокирующая** операция.

В Node.js медленно исполняемый JS код не принято называть **блокирующим**, если причиной тому высокая нагрузка кода на процессор, а не ожидание завершения сторонней операции. Синхронные методы в стандартной библиотеке Node.js, которые используют libuv — наиболее часто применяемые **блокирующие** операции. Нативные модули также могут иметь **блокирующие** методы.

Все I/O методы в стандартной библиотеке Node.js предоставляют свои асинхронные версии, которые являются **неблокирующими** и принимают функции обратного вызова в качестве аргумента. Некоторые методы также имеют свои **блокирующие** аналоги. Названия таких методов заканчиваются на `Sync`.

## Сравнение Кода

**Блокирующие** методы исполняются **синхронно**, а **неблокирующие** методы исполняются **асинхронно**.

Возьмем модуль File System для примера. Вот пример **синхронного** чтения файла:

```js
const fs = require('fs');
// исполнение кода заблокировано, пока файл не будет полностью считан
const data = fs.readFileSync('/file.md');
```

А вот эквивалентный **асинхронный** пример:

```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
});
```

Первый пример выглядит проще чем второй, но он имеет один недостаток: вторая строка **блокирует** исполнение любого нижеследующего кода, до тех пор, пока весь file.md не будет считан. Обратите внимание, если синхронная версия кода сгенерирует исключение, его нужно обработать, иначе процесс Node.js "упадёт". В асинхронном варианте выбор — сгенерировать исключение или нет — оставлен на усмотрение программиста.

Давайте немного расширим наш пример:

```js
const fs = require('fs');
// исполнение кода заблокировано, пока файл не будет полностью считан
const data = fs.readFileSync('/file.md');
console.log(data);
moreWork(); // функция будет исполнена, после console.log
```

А вот похожий, но не эквивалентный асинхронный пример:

```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
moreWork(); // функция будет исполнена до console.log
```

В первом примере метод `console.log` будет вызван до срабатывания функции `moreWork()`. Во втором примере метод `fs.readFile()` является **неблокирующим**, поэтому исполнение JavaScript может продолжаться, не дожидаясь окончания его работы. Как следствие функция `moreWork()` сработает раньше `console.log`. Эта возможность — отсутствие необходимости дожидаться окончания чтения файла и других системных вызовов — ключевое инженерное решение, которое обеспечивает высокую пропускную способность Node.js.

## Конкурентность и Пропускная Способность

Исполнение JavaScript в Node.js является однопоточным. Поэтому, говоря о конкурентности (параллельности вычислений) в Node.js, подразумевают, что после того, как цикл событий обработал синхронный код, он способен обработать функции обратного вызова. Подобно сторонним операциям (таким как I/O), любой конкурентный код должен позволять циклу событий продолжать свою работу.

В качестве примера возьмем запросы к веб-серверу. Допустим, обработка сервером одного запроса занимает 50мс. Из этих 50мс, 45мс уходит на операции чтения/записи в базу данных. С базой данных можно взаимодействовать и **асинхронно**. При таком подходе на каждый запрос к веб-серверу **неблокирующая** асинхронная операция высвободит 45мс для обработки других запросов, а это существенная разница.

Обработка конкурентной (параллельной) работы при помощи цикла событий в Node.js отличается от подходов во многих других языках программирования, в которых могут создаваться дополнительные потоки.

## Опасность смешивания Блокирующего и Неблокирующего Кода

Существуют паттерны, которые следует избегать при работе с I/O. Взглянем на пример:

```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
fs.unlinkSync('/file.md');
```

В вышеуказанном примере метод `fs.unlinkSync()` с высокой вероятностью будет исполнен до `fs.readFile()`. Это приведет к удалению файла до его прочтения. Лучше переписать этот код в **неблокирующем** виде, что гарантирует правильный порядок исполнения методов:

```js
const fs = require('fs');
fs.readFile('/file.md', (readFileErr, data) => {
  if (readFileErr) throw readFileErr;
  console.log(data);
  fs.unlink('/file.md', unlinkErr => {
    if (unlinkErr) throw unlinkErr;
  });
});
```

В последнем примере **неблокирующий** вызов метода `fs.unlink()` расположен внутри функции обратного вызова `fs.readFile()`. Такой подход гарантирует правильную последовательность операций.

## Дополнительные ресурсы

- [libuv](https://libuv.org/)
