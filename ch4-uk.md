# Розділ 4: Каррування

## Не можу жити, якщо жити треба без тебе
Якось мій батько пояснив мені, що існують речі, без яких людина може спокійно жити до тих пір, поки їх не спробує. Наприклад - мікрохвильова піч. Або смартфони. Старші люди пригадають повноцінне і переповнене подіями життя без інтернету. А для мене одна з таких речей - карування.

Концепція проста: Ви можете викликати функцію з меншою кількістю аргументів, ніж вона очікує. Вона повертає іншу функцію, яка приймає решту аргументів. 

Ви можете обрати, чи викликати її з усіма аргументами одразу, чи просто згодувати аргументи один за одним.

```js
var add = function(x) {
  return function(y) {
    return x + y;
  };
};

var increment = add(1);
var addTen = add(10);

increment(2);
// 3

addTen(2);
// 12
```

Ми створили функцію `add`, яка приймає один параметр та повертає функцію. Викликавши її, функція, яка буде повернена, запам'ятовує перший аргумент звідти за допомогою замикання. Викликавши її з обома аргументами за один раз - трохи болюче, але, ми можемо використати спеціальну допоміжну функцію, яка називається `curry`. Це спростить нам задачу у оголошенні та викликані функцій як вищенаведена.

Давайте налаштуємо кілька каррованних функцій для нашого задоволення.

```js
var curry = require('lodash/curry');

var match = curry(function(what, str) {
  return str.match(what);
});

var replace = curry(function(what, replacement, str) {
  return str.replace(what, replacement);
});

var filter = curry(function(f, ary) {
  return ary.filter(f);
});

var map = curry(function(f, ary) {
  return ary.map(f);
});
```

Шаблон, який я використав, дуже простий, але також важливий. Я завбачливо розмістив дані, над якими ми проводимо дії (строка, масив), в якості останнього аргументу. Трохи згодом стане зрозуміло як і коли їх використовувати.

```js
match(/\s+/g, 'hello world');
// [ ' ' ]

match(/\s+/g)('hello world');
// [ ' ' ]

var hasSpaces = match(/\s+/g);
// function(x) { return x.match(/\s+/g) }

hasSpaces('hello world');
// [ ' ' ]

hasSpaces('spaceless');
// null

filter(hasSpaces, ['tori_spelling', 'tori amos']);
// ['tori amos']

var findSpaces = filter(hasSpaces);
// function(xs) { return xs.filter(function(x) { return x.match(/\s+/g) }) }

findSpaces(['tori_spelling', 'tori amos']);
// ['tori amos']

var noVowels = replace(/[aeiouy]/ig);
// function(replacement, x) { return x.replace(/[aeiouy]/ig, replacement) }

var censored = noVowels("*");
// function(x) { return x.replace(/[aeiouy]/ig, '*') }

censored('Chocolate Rain');
// 'Ch*c*l*t* R**n'
```

Тут продемонстрована можливість "пре-завантажити" функцію з аргументом чи кількома, в певному порядку, щоб отримати нову функцію, яка пам'ятає ті аргументи.

Я пропоную вам запустити `npm install lodash`, скопіювати код зверху і запустити його в REPL. Ви також можете зробити це в браузері, де доступні lodash чи ramda. 

## Більше ніж каламбур чи спеціальний соус

Каррування корисне для багатьох речей. Ми можемо робити нові функції просто передаючи нашим базовим функціям деякі аргументи, як от наприклад в `hasSpaces`, `findSpaces` та `censored`.

Ми також маємо можливість трансформувати будь-яку функцію, що працює над одинарними елементами у функцію, котра працюватиме з масивами, просто огорнувши її `map`-ом:

```js
var getChildren = function(x) {
  return x.childNodes;
};

var allTheChildren = map(getChildren);
```

Передача у функцію меншої кількості аргументів аніж вона очікує - називається *часткове застосування*. Часткове застосування функції може прибрати багато шаблонного(пер.: _boiler plate_) коду. Розглянемо якою може бути вищенаведенна функція `allTheChildren` з некаррованним методом `map` з бібліотеки lodash (зауважте, що аргументи знаходяться у іншій послідовності):

```js
var allTheChildren = function(elements) {
  return _.map(elements, getChildren);
};
```

Як правило, ми не визначаємо функції, які працюють на масивах, тому що ми можемо просто викликати `map(getChildren)`. Це ж стосується `sort`, `filter` та іншиї функцій вищого порядку(Функція вищого порядку - функція, яка приймає або повертає іншу функцію).

Коли ми говорили про *чисті функції*, ми сказали, що вони приймають одну вхідну величину і повертають одну вихідну. Каррування робить як раз це саме: кожен одинарний параметр повертає нову функцію, яка очікує на решту аргументів. А це якраз один на вході - один на виході.

І не важливо, чи вхідна величина є функцією - вона все одно класифікується як чиста. Ми дозволяємо більше ніж один аргумент за раз, проте, здається, що це лише для зручності і видалення додаткових `()`.


## В завершення

Каррування дуже зручне і я отримую величезне задоволення використовуючи карровані функції на щоденних засадах. Це інструмент на моєму поясі, який робить програмування у функціональному стилі менш багатослівним і нудним.

Ми можемо зробити нову функцію, так би мовити "на льоту", просто передавши у функцію кілька аргументів і як бонус, ми збережемо визначення математичної функції, незважаючи на численні аргументи.

Давайте озброємось іншим важливим інструментом, який називається `compose`.

[Частина 5: Написання коду за допомогою композиції](ch5-uk.md)

## Вправи

Кілька слів перед тим, як ми почнемо. Ми будемо використовувати бібліотеку, яка називається [Ramda](http://ramdajs.com). Вона каррує кожну функцію за замовчуванням. Як альтернативу, ви можете використовувати [lodash/fp](https://github.com/lodash/lodash/wiki/FP-Guide), яка робить теж саме і написана та підтримується творцем бібліотеки lodash. Обидві бібліотеки чудово підійдуть, тож різниця лише у вподобаннях.

Також є [unit тести](https://github.com/DrBoolean/mostly-adequate-guide/tree/master/code/part1_exercises), за допомогою яких ви можете перевіряти свій код, або ж, для початку, ви можете просто скопіювати код та вставити його у JavaScript REPL.

Відповіді з кодом знаходяться у [репозиторії до цієї книги](https://github.com/DrBoolean/mostly-adequate-guide/tree/master/code/part1_exercises/answers). Найкращим шляхом для виконання вправ є цикл миттєвого зворотнього зв'язку ([immediate feedback loop](feedback_loop.md)).

```js
var _ = require('ramda');


// Exercise 1
//==============
// Refactor to remove all arguments by partially applying the function.

var words = function(str) {
  return _.split(' ', str);
};

// Exercise 1a
//==============
// Use map to make a new words fn that works on an array of strings.

var sentences = undefined;


// Exercise 2
//==============
// Refactor to remove all arguments by partially applying the functions.

var filterQs = function(xs) {
  return _.filter(function(x) {
    return match(/q/i, x);
  }, xs);
};


// Exercise 3
//==============
// Use the helper function _keepHighest to refactor max to not reference any
// arguments.

// LEAVE BE:
var _keepHighest = function(x, y) {
  return x >= y ? x : y;
};

// REFACTOR THIS ONE:
var max = function(xs) {
  return _.reduce(function(acc, x) {
    return _keepHighest(acc, x);
  }, -Infinity, xs);
};


// Bonus 1:
// ============
// Wrap array's slice to be functional and curried.
// //[1, 2, 3].slice(0, 2)
var slice = undefined;


// Bonus 2:
// ============
// Use slice to define a function "take" that returns n elements from the beginning of an array. Make it curried.
// For ['a', 'b', 'c'] with n=2 it should return ['a', 'b'].
var take = undefined;
```
