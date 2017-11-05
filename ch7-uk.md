# Розділ 7: Hindley-Milner та я

## Який у тебе тип?
Якщо функціональний світ новий для вас, то дуже скоро ви зрозумієте, що погрузли в сигратурах типах. Типи - це мета мова, яка дозволяє людям з різною базою спілкуватись успішно та ефективно. Для переважній більшості вони написані за допомогою системи "Hindley-Milner" і з якими ми працюватимемо у цьому розділі.

Коли ви працюєте з чистими функціями, типи мають надзвичайну силу, для якої англійська мова не може тримати свічку. Ці сигнатури нашоптують вам у вухо найтаємничиші секрети функції. У одній компактній лінії вони розкривають поведінку та намір. Ми можемо вивести з них "вільні теореми". Типи можуть бути прогнозованими, так що не буде необхідності в явній аннотації типу. Вони можуть бути надзвичайно точними чи лишатись загальними та абстрактними. Вони корисні не лише для перевірок під час компайлінгу, але може так статися, що вони - найкраща з усіх можливих докумнетацій, які доступні. Тому сигнатури типів відіграють дуже важливу роль у функціональному програмуванні, набагато більшу, ніж ви можете очікувати.

JavaScript - динамічна мова, проте це не означає, що ми уникаємо усіх типів. Ми таки працюємо зі строками(strings), числами(numbers), логічними(booleans) і так далі. Справа лише в тому, що немає ніякої мовної інтреграції з ними, тож ми тримаємо їх в наших головах. Але не хвилюйтесь, оскільки ми використовуємо сигнатури для документації - ми можемо лишати коментарі для наших цілей.

Існують інструменти для перевірки типів у JavaScript, такі як [Flow](http://flowtype.org/) чи типізований діалект, [TypeScript](http://www.typescriptlang.org/). Мета цієї книги озброїти читача знаряддям для написання функціонального коду, тож ми візьмемо стандартну систему типів, яка використовуєтся серед різних функціональних мов програмування.


## Казки з загадковості

Починаючи з вкритих пулюкою математичних книжок, у величезному морі білих аркушів, серед звичайних суботніх публікацій, і закінчуючи у самому серці початкового коду, ми знаходимо сигнатури типів Hindley-Milner. Система досить проста, але потребує швидкого пояснення і деяких практичних заннять, щоб цілком поглинути маленьку мову.

```js
//  capitalize :: String -> String
var capitalize = function(s) {
  return toUpperCase(head(s)) + toLowerCase(tail(s));
}

capitalize("smurf");
//=> "Smurf"
```

Тут `capitalize` бере `String`(строку) та повертає `String`(строку). Не зважаючи на реалізацію - це саме та сигнатура типу яка нас цікавить.

У HM(_прим.пер.:_ Hindley-Milner) функції написані як `a -> b`, де `a` та `b` - змінні будь яких типів. Тож сигнатура фунції `capitalize` може бути прочитана як "функція від строки до строки". Інакше кажучи, вона бере строку(`String`) як вхідну величину та повертає строку(`String`) як вихідну.

Давайте розглянемо трохи більше сигнатур функцій:

```js
//  strLength :: String -> Number
var strLength = function(s) {
  return s.length;
}

//  join :: String -> [String] -> String
var join = curry(function(what, xs) {
  return xs.join(what);
});

//  match :: Regex -> String -> [String]
var match = curry(function(reg, s) {
  return s.match(reg);
});

//  replace :: Regex -> String -> String -> String
var replace = curry(function(reg, sub, s) {
  return s.replace(reg, sub);
});
```

`strLength` - тут таж сама ідея як і раніше: ми беремо `String` і повертаємо вам `Number`.

Інші можуть трохи вас збентежити спочатку. Без повного розуміння деталей ви завжди можете подивитись на найостанніший тип, який повертається в якості значення. Тож функцію `match` ви можете прочитати так: Вона бере `Regex`(регулярний вираз) та `String`(строку) і повертає вам `[String]`(масив строк). Але тут відбувається одна цікава річ, яку б я хотів зупинитись, на мить, та пояснити, якщо ви не проти.

Для `match` ми можемо згрупувати сигнатури так:

```js
//  match :: Regex -> (String -> [String])
var match = curry(function(reg, s) {
  return s.match(reg);
});
```

І так, групування останньої частини у дужках відкриває більше інформації. Тепер стає очевидним, що функція, яка приймає `Regex` і повертає нам функцію зі строки(`String`) до масиву строк(`[String]`). Насправді це можливе через карування:  надайте регулярний вирах `Regex` і ми отримуємо функцію, яка очікує на аргумент `String`. Звісно, ми не повинні думати про це таким чином, але добре розуміти, чому останній тип є саме таким, яким він повертається.

```js
//  match :: Regex -> (String -> [String])

//  onHoliday :: String -> [String]
var onHoliday = match(/holiday/ig);
```

Кожен аргумент відокремлює один тип від плочатку сигнатури. Фунція `onHoliday` - це функція `match`, яка вже має `Regex`.

```js
//  replace :: Regex -> (String -> (String -> String))
var replace = curry(function(reg, sub, s) {
  return s.replace(reg, sub);
});
```

Як ви можете бачити з усіма дужками у функції `replace`, додаткові описання можуть трохи заважати, саме тому ми просто їх пропускаємо. Якщо ми можемо, ми можемо надати всі аргументи за один раз, тож легше міркувати про це так: функція `replace` приймає регулярний вираз(`Regex`), строку(`String`), іншу строку(`String`) і повертає вам строку(`String`).

І ще тут є кілька моментів:


```js
//  id :: a -> a
var id = function(x) {
  return x;
}

//  map :: (a -> b) -> [a] -> [b]
var map = curry(function(f, xs) {
  return xs.map(f);
});
```

Функція `id` бере будь-який старий тип `a` і повертає щось з таким самим типом як у `a`. Ми можемо використовувати змінні в типах так само як у коді. Назви змінних `a` та `b` є загальноприятими, але вони довільні і можуть бути замінені будь-якими назвами, які вам більше подобаються. Якщо вони однакові змінні - вони мають бути однакового типу. Це дуже важливе правило, тож давйте повторимо: `a -> b` може бути будь-який ти `a` до будь-якого типу `b`, але `a -> a` означає, що вони мають бути одного типу. Наприклад, функція `id` може бути `String -> String` чи `Numnber -> Number`, але не `String -> Bool`.

Функція `map` використовує типові змінні дуже схоже, але цього разу ми представляємо `b`, яка може бути, а може і не бути одного типу з `a`. Ми можемо прочитати це так: функція `map` бере функція з будь-якого типу `a` до такого ж чи іншого типу `b`, потім бере масив типів `a` і повертає, в якості результату, масив типів `b`.

Сподіваюсь, вас підкорила виразна краса сигнатури йього типу. Вона буквально розповідає нам, що функція робить майже слово в слово. Дано функцію від `a` до `b`, масив елементів `а`, і вона повертає нам масив елементів `b`. Єдиною розумною річчю для функції є викликати кровопролитну функцію на кожному елементі `a`. Все інше буде чистою брехнею.

Можливість розмірковування щодо типів та їх наслідків - це навичка, яка заведе вас далеко в функціональний світ. Не лише журнали, блоги, документи і т.д. стануть більш засвоюваними, але також сигнатури зможуть вам детально розповідати про свою функціональність. Треба практикуватись, щоб мати змогу вільно читати сигнатури, але якщо будете притримуватись цього - багато інформації стане для вас доступною без RTFMing.

Ось ще кілька сигнатур, щоб побачити, чи можете ви розшифрувати їх самостійно.

```js
//  head :: [a] -> a
var head = function(xs) {
  return xs[0];
};

//  filter :: (a -> Bool) -> [a] -> [a]
var filter = curry(function(f, xs) {
  return xs.filter(f);
});

//  reduce :: (b -> a -> b) -> b -> [a] -> b
var reduce = curry(function(f, x, xs) {
  return xs.reduce(f, x);
});
```

Функція `reduce`, напевно, є набільш виразною з усіх. Проте, вона і не така проста, тож не відчувайте дивно, якщо вам доведеться трохи з нею поборотися. Для цікавості, я намагатимусь пояснити англійською мовою, хоча самостійна робота над сигнатурою є набагато більш повчальною.

Ах, тут нічого не відбувається... дивлячись на підпис, ми бачимо, що перший аргумент - це функція, яка очікує на `b`, `a` і видає `b`. До чого можуть привести ці `a` та `b`? Ну, наступні аргументи у сигнатурі є `b` і масив елементів `a`, тому ми можемо тільки припустити, що `b` і кожен з тих елементів `a` будуть передані. Ми також бачимо, що результат функції є `b`, тому розмірковування мають привести нас до висновку, що остаточний результат переданої функції і буде нашим вихідним значенням. Знаючи, що робить `reduce`, можна стверджувати, що наведене вище дослідження є точним.


## Зниження можливості

Після введення типу змінної виникає цікава властивість, яка називається *[параметричністю](https://en.wikipedia.org/wiki/Parametricity)*. Ця властивість стверджує, що функція *буде діяти на всіх типах однково*. Давайте це перевіримо:

```js
// head :: [a] -> a
```

Дивлячись на функцію `head` ми бачимо, що вона приймає `[a]` і повертає `a`. Окрім того конкретний тип `array`(масив) і більше жодної інформації, тому робота функції обмежена лиже з масивом. Що вона б могла робити зі змінною `a`, якщо вона більше нічого не знає про неї? Інакше кажучи, змінна `a` каже, що вона не може бути *конкретного* типу, що означає, що вона може бути будь-якого типу(прим.пер.: тип *any*), що лишає нас із функцією, котра має працювати однаково з *кожним* будь-яким типом. Це як раз те, що характерезує *параметричність*. Дивлячись на реалізацію, можна припустити, що єдине обгрунтоване припущення полягає в тому, що функція приймає перший, останній чи будь-який випадковий елемент з цього масиву. Назва функції `head` повинна нам надати підказку.

Ось іще одна:

```js
// reverse :: [a] -> [a]
```

Що можна сказати по одній тільки сигнатурі про функцію `reverse`? Знову ж таки, вона не може робити нічого специфічного з `a`. Вона не може змінити тип `a` на інший, бо інакше б нам подали `b`. Чи може вона сортувати? Щож, ні, бо не вистачило б достатньо інформації про всі можливі типи. Чи може вона переупорядкувати? Так, я припускаю, що може, але вона має це робити у точно такий самий прозорий шлях. Інша можливість, що вона може вирішити видалити, чи дублювати елемент. У будь-якому випадку, ідея така, що можлива поведінка суттєво звужена його поліморфним типом.

Це звуження можливості дозволяє нам використовувати пошуковий двигун сигнатури типу, такий як [Hoogle](https://www.haskell.org/hoogle), для того, щоб зрозуміти, яка функція буде потім. Інформація, ретельно запакована у сигнатуру є насправді дуже потужною.

## Вільно, як у теоремі

Besides deducing implementation possibilities, this sort of reasoning gains us *free theorems*. What follows are a few random example theorems lifted directly from [Wadler's paper on the subject](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf).

```js
// head :: [a] -> a
compose(f, head) == compose(head, map(f));

// filter :: (a -> Bool) -> [a] -> [a]
compose(map(f), filter(compose(p, f))) === compose(filter(p), map(f));
```


You don't need any code to get these theorems, they follow directly from the types. The first one says that if we get the `head` of our array, then run some function `f` on it, that is equivalent to, and incidentally, much faster than, if we first `map(f)` over every element then take the `head` of the result.

You might think, well that's just common sense. But last I checked, computers don't have common sense. Indeed, they must have a formal way to automate these kind of code optimizations. Maths has a way of formalizing the intuitive, which is helpful amidst the rigid terrain of computer logic.

The `filter` theorem is similar. It says that if we compose `f` and `p` to check which should be filtered, then actually apply the `f` via `map` (remember filter, will not transform the elements - its signature enforces that `a` will not be touched), it will always be equivalent to mapping our `f` then filtering the result with the `p` predicate.

These are just two examples, but you can apply this reasoning to any polymorphic type signature and it will always hold. In JavaScript, there are some tools available to declare rewrite rules. One might also do this via the `compose` function itself. The fruit is low hanging and the possibilities are endless.

## Constraints

One last thing to note is that we can constrain types to an interface.

```js
// sort :: Ord a => [a] -> [a]
```

What we see on the left side of our fat arrow here is the statement of a fact: `a` must be an `Ord`. Or in other words, `a` must implement the `Ord` interface. What is `Ord` and where did it come from? In a typed language it would be a defined interface that says we can order the values. This not only tells us more about the `a` and what our `sort` function is up to, but also restricts the domain. We call these interface declarations *type constraints*.

```js
// assertEqual :: (Eq a, Show a) => a -> a -> Assertion
```

Here, we have two constraints: `Eq` and `Show`. Those will ensure that we can check equality of our `a`s and print the difference if they are not equal.

We'll see more examples of constraints and the idea should take more shape in later chapters.


## In Summary

Hindley-Milner type signatures are ubiquitous in the functional world. Though they are simple to read and write, it takes time to master the technique of understanding programs through signatures alone. We will add type signatures to each line of code from here on out.

[Chapter 8: Tupperware](ch8.md)
