# Chapter 08: Tupperware

## The Mighty Container

<img src="images/jar.jpg" alt="http://blog.dwinegar.com/2011/06/another-jar.html" />

Ми бачили, як писати програми, які передають дані через серію чистих функцій. Це декларативні специфікації поведінки. Але як щодо робити з управлінням потоком, обробкою помилок, асинхронними діями, станом і, наважуся сказати, ефектами?! У цьому розділі ми відкриємо основу, на якій побудовані всі ці корисні абстракції.

Спочатку ми створимо контейнер. Цей контейнер повинен тримати будь-який тип значення; ziplock, в якому може міститись лише тільки пудинг тапіока, рідко буває корисним. Це буде об'єкт, але ми не надамо йому властивостей та методів у сенсі ООП. Ні, ми будемо ставитися до нього як до скарбнички — особливої скриньки, яка оберігає наші цінні дані.

```js
class Container {
  constructor(x) {
    this.$value = x;
  }
  
  static of(x) {
    return new Container(x);
  }
}
```

Ось наш перший контейнер. Ми обдумано назвали його `Container`. Ми використовуватимемо `Container.of` як конструктор, який врятує нас від необхідності всюди писати це жахливе слово `new`. У функції `of` є більше, ніж може здатися на перший погляд, але наразі думайте про неї як про правильний спосіб розміщення значень у нашому контейнері.

Давайте протестуємо нашу абсолютно нову скриньку...

```js
Container.of(3);
// Container(3)

Container.of('hotdogs');
// Container("hotdogs")

Container.of(Container.of({ name: 'yoda' }));
// Container(Container({ name: 'yoda' }))
```

Якщо ви використовуєте Node, ви побачите `{$value: x}`, хоча у нас є `Container(x)`. Chrome виведе тип правильно, але це не так важливо; до тих пір допоки ми розуміємо, як виглядає `Container` - з нами все буде добре. В деяких середовищах ви можете перевизначити метод `inspect`, якщо хочете, але ми не будемо так детально зупинятися на цьому. Для цієї книги ми будемо писати концептуальний вивід так, ніби ми перевизначили `inspect`, оскільки це набагато повчальніше, ніж `{$value: x}`, з педагогічних, а також естетичних причин.

Давайте уточнимо кілька речей, перш ніж продовжити:

* `Container` це об'єкт з однією властивістю. Багато контейнерів містять лише одну річ, хоча вони не обмежені однією. Ми довільно назвали його властивість `$value`.

* `$value` не може бути одного конкретного типу, інакше наш `Container` навряд чи виправдовував би свою назву.

* Як тільки дані потрапляють у `Container`, вони залишаються там. Ми можемо *витягнути* їх, використовуючи `.$value`, але це суперечило б меті.

Причини, чому ми це робимо, стануть прозорими немов склянна банка, але наразі просто побудьте зі мною.


## Мій Перший Функтор

Як тільки наше значення, яким би воно не було, потрапило до контейнеру, нам подрібен спосіб запускати на ньому функції. 

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function (f) {
  return Container.of(f(this.$value));
};
```

Це точно так само, як відомий `map` у масивах, тільки у нас є `Container a` замість `[a]`. І це працює по суті так само:

```js
Container.of(2).map(two => two + 2); 
// Container(4)

Container.of('flamethrowers').map(s => s.toUpperCase()); 
// Container('FLAMETHROWERS')

Container.of('bombs').map(append(' away')).map(prop('length')); 
// Container(10)
```

Ми можемо працювати з нашим значенням, ніколи не виходячи з `Container`. Це неймовірна річ. Наше значення в `Container` передається функції `map`, щоб ми могли з ним погратись, а потім повертається до свого `Container` для безпечного зберігання. Через те, що ми ніколи не залишаємо `Container`, ми можемо продовжувати застосовувати `map`, запускаючи функції на свій розсуд. Ми навіть можемо змінювати тип по ходу, як показано в третьому з трьох прикладів.

Заждіть хвилинку, якщо ми продовжуємо викликати `map`, це виглядає як якась композиція! Яка математична магія тут працює? Ну що ж, друзі, ми щойно відкрили *Функтори*.

> Функтор — це тип, який реалізує `map` та дотримується деяких законів

Так, *Функтор* — це просто інтерфейс із договором. Ми могли б так само легко назвати його *Mappable*, але де тоді в цьому будуть *веселощі*? Функтори походять з теорії категорій, і ми розглянемо математику детальніше ближче до кінця розділу, але зараз давайте зосередимося на інтуїції та практичному застосуванні цього дивно названого інтерфейсу.

З якої причини ми можемо захотіти запечатати значення та використовувати `map` для доступу до нього? Відповідь зʼявиться, якщо ми підберемо краще питання: Що ми отримуємо, попросивши наш контейнер застосовувати функції замість нас? Що ж... абстракція застосування функцій. Коли ми застосовуємо `map` до функції, ми просимо тип контейнера виконати її за нас. Це, насправді, дуже потужна концепція.

## Можливо Шрьодінгера

<img src="images/cat.png" alt="cool cat, need reference" />

`Container` is fairly boring. In fact, it is usually called `Identity` and has about the same impact as our `id` function (again there is a mathematical connection we'll look at when the time is right). However, there are other functors, that is, container-like types that have a proper `map` function, which can provide useful behaviour whilst mapping. Let's define one now.
`Container` є досить нудним. Насправді його зазвичай називають `Identity` і він має приблизно такий же вплив, як наша функція `id` (знову ж таки, є математичний зв'язок, на який ми подивимося, коли настане час). Однак, існують інші функтори, тобто типи, схожі на контейнери, які мають належну функцію `map`, що може забезпечити корисну поведінку під час мапінгу. Давайте визначимо один зараз.

> A complete implementation is given in the [Appendix B](./appendix_b.md#Maybe)

```js
class Maybe {
  static of(x) {
    return new Maybe(x);
  }

  get isNothing() {
    return this.$value === null || this.$value === undefined;
  }

  constructor(x) {
    this.$value = x;
  }

  map(fn) {
    return this.isNothing ? this : Maybe.of(fn(this.$value));
  }

  inspect() {
    return this.isNothing ? 'Nothing' : `Just(${inspect(this.$value)})`;
  }
}
```

Тепер, `Maybe` виглядає майже як `Container` з однією маленькою відмінністю: воно спочатку перевірить чи існує значення перед тим як викликати передану функцію. Це дозволяє позбутися тих набридливих `null` в ході використання `map` (Зауважте, що імлементація цього є спрощеною задля навчальних цілей).

```js
Maybe.of('Malkovich Malkovich').map(match(/a/ig));
// Just(True)

Maybe.of(null).map(match(/a/ig));
// Nothing

Maybe.of({ name: 'Boris' }).map(prop('age')).map(add(10));
// Nothing

Maybe.of({ name: 'Dinah', age: 14 }).map(prop('age')).map(add(10));
// Just(24)
```

Зверніть увагу, що наша программа не вибухає через помилки, коли ми застосовуємо функції в процесі мапінгу на наших нульових значень. Це відбувається завдяки тому, що `Maybe` буде ретельно перевіряти наявність значення кожного разу, коли застосовується функція.

Цей синтакс із точкою цілком нормальний та функціональний, але, через причини зазначені в Розіділі 1, ми б хотіли зберегти наш безточковий стиль. Як виявляється, `map` повністю обладнаний для делегування будь-якому функтору, який він отримує:

```js
// map :: Functor f => (a -> b) -> f a -> f b
const map = curry((f, anyFunctor) => anyFunctor.map(f));
```

Це чудово, оскільки ми можемо продовжувати використовувати композицію, як зазвичай, і `map` працюватиме, як очікується. Це стосується і `map` з бібліотеки Ramda. Ми будемо використовувати точкову нотацію, коли це повчально, і безточкову версію, коли це зручно. Ви помітили це? Я хитро ввів додаткову нотацію в нашу сигнатуру типу. `Functor f =>` вказує нам, що `f` повинен бути функтором. Це не так складно, але я відчував, що повинен це згадати.

## Випадки Використання

У реальних умовах ми зазвичай можемо побачити, що `Maybe` використовується у функціях, які можуть не повернути результат.

```js
// safeHead :: [a] -> Maybe(a)
const safeHead = xs => Maybe.of(xs[0]);

// streetName :: Object -> Maybe String
const streetName = compose(map(prop('street')), safeHead, prop('addresses'));

streetName({ addresses: [] });
// Nothing

streetName({ addresses: [{ street: 'Shady Ln.', number: 4201 }] });
// Just('Shady Ln.')
```

`safeHead` подібна до нашої звичайної `head`, але з додатковою типобезпекою. Цікава річ трапляється, коли до нашого коду додається `Maybe`; ми змушені мати справу з тими підступними нульовими значеннями `null`. Функція `safeHead` чесно і відкрито повідомляє про можливу невдачу — в цьому немає нічого ганебного — і тому вона повертає `Maybe`, щоб поінформувати нас про це. Однак ми не просто *поінформовані*, тому що змушені використовувати `map`, щоб дістатися до потрібного значення, яке сховане всередині об'єкта `Maybe`. По суті, це перевірка на `null`, яку здійснює сама функція `safeHead`. Тепер ми можемо спокійніше спати, знаючи, що значення `null` не з'явиться несподівано, коли ми цього найменше очікуємо. Такі API перетворюють ненадійний додаток із паперу та кнопок на міцну конструкцію з дерева та цвяхів. Вони гарантують безпечніше програмне забезпечення.

Іноді функція може явно повертати `Nothing`, щоб сигналізувати про невдачу. Наприклад:

```js
// withdraw :: Number -> Account -> Maybe(Account)
const withdraw = curry((amount, { balance }) =>
  Maybe.of(balance >= amount ? { balance: balance - amount } : null));

// Ця функція є гіпотетичною, не реалізованою тут... і ніде інде.
// updateLedger :: Account -> Account 
const updateLedger = account => account;

// remainingBalance :: Account -> String
const remainingBalance = ({ balance }) => `Your balance is $${balance}`;

// finishTransaction :: Account -> String
const finishTransaction = compose(remainingBalance, updateLedger);


// getTwenty :: Account -> Maybe(String)
const getTwenty = compose(map(finishTransaction), withdraw(20));

getTwenty({ balance: 200.00 }); 
// Just('Your balance is $180')

getTwenty({ balance: 10.00 });
// Nothing
```

`withdraw` поверне `Nothing`, якщо у нас недостатньо коштів. Ця функція також повідомляє про свою непостійність і залишає нам лише один варіант - використовувати `map` для всього, що йде після цього. Різниця в тому, що тут `null` був навмисним. Замість `Just('..')`, ми отримуємо `Nothing`, щоб сигналізувати про невдачу, і наш додаток фактично зупиняється. Це важливо зазначити: якщо `withdraw` не вдається, тоді `map` припиняє решту обчислень, оскільки він не виконує відображені функції, а саме `finishTransaction`. Це саме та поведінка, яку ми очікуємо, оскільки ми б не хотіли оновлювати наші рахунки або показувати новий баланс, якщо ми не встигли успішно зняти кошти.

## Вивільнення Значення

Одна річ, яку люди часто пропускають, це те, що завжди буде кінцева точка; якась функція, що відправляє JSON, або друкує на екран, або змінює файлову систему, чи щось подібне. Ми не можемо доставити вихідні дані за допомогою `return`, ми повинні виконати якусь функцію, щоб відправити їх у світ. Ми можемо сформулювати це як коан дзен-буддиста: "Якщо програма не має видимого ефекту, чи вона взагалі виконується?". Чи виконується вона правильно? Я підозрюю, що вона просто проходить якимись циклами і знову засинає...

Завдання нашого додатка - отримувати, трансформувати і переносити дані доти, поки не прийде час попрощатися, і функція, яка це робить, може бути відображена, таким чином, значення не обов'язково повинно залишати теплий "середовище" свого контейнера. Справді, поширена помилка - намагатися будь-яким способом видалити значення з нашого `Maybe`, ніби можливе значення всередині раптом матеріалізується і все буде прощено. Ми повинні розуміти, що може бути гілка коду, де наше значення не зможе виконати свою долю. Наш код, подібно до кота Шредінгера, одночасно перебуває у двох станах і повинен підтримувати цей факт до завершальної функції. Це надає нашому коду лінійний потік, незважаючи на логічне розгалуження.

Однак, є запасний вихід. Якщо ми хочемо повернути власне значення і продовжити, ми можемо використовувати невелику допоміжну функцію під назвою `maybe`.

```js
// maybe :: b -> (a -> b) -> Maybe a -> b
const maybe = curry((v, f, m) => {
  if (m.isNothing) {
    return v;
  }

  return f(m.$value);
});

// getTwenty :: Account -> String
const getTwenty = compose(maybe('You\'re broke!', finishTransaction), withdraw(20));

getTwenty({ balance: 200.00 }); 
// 'Your balance is $180.00'

getTwenty({ balance: 10.00 }); 
// 'You\'re broke!'
```

Тепер ми можемо або повернути статичне значення (того ж типу, що й `finishTransaction`), або продовжити весело завершувати транзакцію без `Maybe`. За допомогою `maybe` ми спостерігаємо еквівалент оператора `if/else`, тоді як з `map` імперативний аналог був би: `if (x !== null) { return f(x) }`.

Введення `Maybe` може викликати початковий дискомфорт. Користувачі Swift і Scala зрозуміють, про що я говорю, оскільки це вбудовано в основні бібліотеки під виглядом `Option(al)`. Коли нас змушують постійно мати справу з перевірками `null` (а іноді ми з абсолютною впевненістю знаємо, що значення існує), більшість людей не можуть не відчувати, що це трохи клопітно. Однак з часом це стане другою натурою, і ви, ймовірно, оціните безпеку. Зрештою, найчастіше це дозволить уникнути ризиків і зберегти наші зусилля.

Написання небезпечного програмного забезпечення схоже на те, щоб ретельно фарбувати кожне яйце пастельними кольорами перед тим, як кинути його в потік транспорту; або на будівництво будинку для престарілих з матеріалів, від яких застерігали три маленькі поросята. Нам буде корисно додати трохи безпеки в наші функції, і `Maybe` допомагає нам у цьому.

Я б був недбалим, якби не згадав, що "реальна" реалізація розділить `Maybe` на два типи: один для чогось і інший для нічого. Це дозволяє нам дотримуватися параметричності в `map`, щоб такі значення, як `null` і `undefined`, все ще можна було обробити, і загальна кваліфікація значення в функторі буде дотримана. Ви часто побачите типи, такі як `Some(x) / None` або `Just(x) / Nothing`, замість `Maybe`, який робить перевірку на `null` свого значення.

## Чиста Обробка Помилок

<img src="images/fists.jpg" alt="pick a hand... need a reference" />

Вас це може шокувати, але `throw/catch` не є дуже чистим методом. Коли виникає помилка, замість того, щоб повернути вихідне значення, ми підіймаємо тривогу! Функція атакує, розкидаючи тисячі 0 і 1, як щити та списи в електричній битві проти нашого загарбницького вводу. З нашим новим другом `Either`, ми можемо діяти краще, ніж оголошувати війну вводу, ми можемо відповісти ввічливим повідомленням. Давайте подивимося:

> Повна реалізація продемонстрована в [Appendix B](./appendix_b.md#Either)

```js
class Either {
  static of(x) {
    return new Right(x);
  }

  constructor(x) {
    this.$value = x;
  }
}

class Left extends Either {
  map(f) {
    return this;
  }

  inspect() {
    return `Left(${inspect(this.$value)})`;
  }
}

class Right extends Either {
  map(f) {
    return Either.of(f(this.$value));
  }

  inspect() {
    return `Right(${inspect(this.$value)})`;
  }
}

const left = x => new Left(x);
```

`Left` та `Right` є двома підкласами абстрактного типу, який ми називаємо `Either`. Я пропустив церемонію створення суперкласу `Either`, оскільки ми ніколи не будемо його використовувати, але добре знати про його існування. Отже, тут немає нічого нового, крім двох типів. Давайте подивимося, як вони діють:

```js
Either.of('rain').map(str => `b${str}`); 
// Right('brain')

left('rain').map(str => `It's gonna ${str}, better bring your umbrella!`); 
// Left('rain')

Either.of({ host: 'localhost', port: 80 }).map(prop('host'));
// Right('localhost')

left('rolls eyes...').map(prop('host'));
// Left('rolls eyes...')
```

`Left` є підлітковим типом та ігнорує наш запит на застосування `map` до нього. `Right` буде працювати так само, як `Container` (відомий також як `Identity`). Сила `Either` полягає в здатності вбудовувати повідомлення про помилку всередині `Left`.

Припустімо, у нас є функція, яка може не досягти успіху. Наприклад, обчислимо вік за датою народження. Ми могли б використовувати `Nothing`, щоб сигналізувати про невдачу та розгалужувати програму, однак це не дає нам багато інформації. Можливо, ми хотіли б знати, чому сталася помилка. Напишемо це, використовуючи `Either`.

```js
const moment = require('moment');

// getAge :: Date -> User -> Either(String, Number)
const getAge = curry((now, user) => {
  const birthDate = moment(user.birthDate, 'YYYY-MM-DD');

  return birthDate.isValid()
    ? Either.of(now.diff(birthDate, 'years'))
    : left('Birth date could not be parsed');
});

getAge(moment(), { birthDate: '2005-12-12' });
// Right(9)

getAge(moment(), { birthDate: 'July 4, 2001' });
// Left('Birth date could not be parsed')
```

Тепер, так само як і з `Nothing`, ми припиняємо виконання нашого додатку, коли повертаємо `Left`. Різниця в тому, що тепер у нас є підказка, чому наша програма зазнала невдачі. Зверніть увагу, що ми повертаємо `Either(String, Number)`, де `String` є значенням зліва, а `Number` — значенням справа (як його `Right`). Ця сигнатура типу трохи неформальна, оскільки ми не витратили час на визначення реального суперкласу `Either`, проте ми багато чого дізнаємося з цього типу. Він повідомляє нам, що ми або отримуємо повідомлення про помилку, або вік.

```js
// fortune :: Number -> String
const fortune = compose(concat('If you survive, you will be '), toString, add(1));

// zoltar :: User -> Either(String, _)
const zoltar = compose(map(console.log), map(fortune), getAge(moment()));

zoltar({ birthDate: '2005-12-12' });
// 'If you survive, you will be 10'
// Right(undefined)

zoltar({ birthDate: 'balloons!' });
// Left('Birth date could not be parsed')
```

Коли дата народження (`birthDate`) є дійсною, програма виводить своє містичне передбачення на екран для нас. Інакше ми отримуємо `Left` з повідомленням про помилку, яке добре видно, хоча й заховане в контейнері. Це діє так, ніби ми кинули помилку, але в спокійній, стриманій манері, а не кричачи, як дитина, коли щось йде не так.

У цьому прикладі ми логічно розгалужуємо потік управління залежно від дійсності дати народження, однак це читається як один лінійний рух зправа налівої, а не як карабкання через фігурні дужки умовного оператора. Зазвичай ми виносили б `console.log` за межі нашої функції `zoltar` і використовували б `map` під час виклику, але корисно побачити, як відрізняється гілка `Right`. Ми використовуємо `_` в сигнатурі типу правої гілки, щоб вказати, що це значення слід ігнорувати (в деяких браузерах потрібно використовувати `console.log.bind(console)`, щоб використовувати його як перший клас).

Хочу скористатися цією можливістю, щоб звернути увагу на щось, що ви могли пропустити: `fortune`, незважаючи на своє використання з `Either` у цьому прикладі, абсолютно не знає про наявність будь-яких функцій. Це також стосується `finishTransaction` у попередньому прикладі. Під час виклику функція може бути обгорнута в `map`, що перетворює її з нефункторної функції на функцію з функтором, в неформальних термінах. Ми називаємо цей процес *підйомом* (*lifting*). Функції зазвичай краще працюють з нормальними типами даних, а не з контейнерними типами, а потім піднімаються в потрібний контейнер за необхідності. Це веде до простіших, більш універсальних функцій, які можна змінювати для роботи з будь-яким функтором на вимогу.

`Either` чудово підходить для обробки помилок, таких як валідація, а також більш серйозних, зупиняючих виконання програми помилок, таких як відсутні файли або зламані сокети. Спробуйте замінити деякі приклади з `Maybe` на `Either`, щоб надати кращий зворотний зв'язок.

Зараз я не можу не відчувати, що я заподіяв `Either` шкоду, представивши його лише як контейнер для повідомлень про помилки. Насправді `Either` захоплює логічну диз'юнкцію (відоме як `||`) у типі. Він також кодує ідею копродукту (*Coproduct*) з теорії категорій, яку ми не будемо розглядати в цій книзі, але варто ознайомитися з нею, оскільки там є властивості, які можна використати. Це канонічний сумарний тип (або диз'юнктне об'єднання множин), оскільки кількість можливих значень дорівнює сумі двох вкладених типів (я знаю, що це трохи абстрактно, тому ось [чудова стаття](https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/sum-types)). `Either` може бути багатьма речами, але як функтор він використовується для обробки помилок.

Так само, як і з `Maybe`, у нас є маленький `either`, який поводиться схоже, але приймає дві функції замість однієї і статичне значення. Кожна функція повинна повертати один і той самий тип:

```js
// either :: (a -> c) -> (b -> c) -> Either a b -> c
const either = curry((f, g, e) => {
  let result;

  switch (e.constructor) {
    case Left:
      result = f(e.$value);
      break;

    case Right:
      result = g(e.$value);
      break;

    // No Default
  }

  return result;
});

// zoltar :: User -> _
const zoltar = compose(console.log, either(id, fortune), getAge(moment()));

zoltar({ birthDate: '2005-12-12' });
// 'If you survive, you will be 10'
// undefined

zoltar({ birthDate: 'balloons!' });
// 'Birth date could not be parsed'
// undefined
```

Нарешті, знайшлося застосування для тієї загадкової функції `id`. Вона просто повертає значення з `Left`, щоб передати повідомлення про помилку до `console.log`. Ми зробили наш додаток для передбачення майбутнього більш надійним, впровадивши обробку помилок безпосередньо у функцію `getAge`. Ми або надаємо користувачеві жорстку правду, як облизень від ворожки, або продовжуємо процес. І з цим усим ми готові перейти до абсолютно іншого типу функтора.

## Old McDonald Had Effects...

<img src="images/dominoes.jpg" alt="dominoes.. need a reference" />

In our chapter about purity we saw a peculiar example of a pure function. This function contained a side-effect, but we dubbed it pure by wrapping its action in another function. Here's another example of this:

```js
// getFromStorage :: String -> (_ -> String)
const getFromStorage = key => () => localStorage[key];
```

Had we not surrounded its guts in another function, `getFromStorage` would vary its output depending on external circumstance. With the sturdy wrapper in place, we will always get the same output per input: a function that, when called, will retrieve a particular item from `localStorage`. And just like that (maybe throw in a few Hail Mary's) we've cleared our conscience and all is forgiven.

Except, this isn't particularly useful now is it. Like a collectible action figure in its original packaging, we can't actually play with it. If only there were a way to reach inside of the container and get at its contents... Enter `IO`.

```js
class IO {
  static of(x) {
    return new IO(() => x);
  }

  constructor(fn) {
    this.$value = fn;
  }

  map(fn) {
    return new IO(compose(fn, this.$value));
  }

  inspect() {
    return `IO(${inspect(this.$value)})`;
  }
}
```

`IO` differs from the previous functors in that the `$value` is always a function. We don't
think of its `$value` as a function, however - that is an implementation detail and we best
ignore it. What is happening is exactly what we saw with the `getFromStorage` example: `IO`
delays the impure action by capturing it in a function wrapper. As such, we think of `IO` as
containing the return value of the wrapped action and not the wrapper itself. This is apparent
in the `of` function: we have an `IO(x)`, the `IO(() => x)` is just necessary to
avoid evaluation. Note that, to simplify reading, we'll show the hypothetical value contained
in the `IO` as result; however in practice, you can't tell what this value is until you've
actually unleashed the effects!

Let's see it in use:

```js
// ioWindow :: IO Window
const ioWindow = new IO(() => window);

ioWindow.map(win => win.innerWidth);
// IO(1430)

ioWindow
  .map(prop('location'))
  .map(prop('href'))
  .map(split('/'));
// IO(['http:', '', 'localhost:8000', 'blog', 'posts'])


// $ :: String -> IO [DOM]
const $ = selector => new IO(() => document.querySelectorAll(selector));

$('#myDiv').map(head).map(div => div.innerHTML);
// IO('I am some inner html')
```

Here, `ioWindow` is an actual `IO` that we can `map` over straight away, whereas `$` is a function that returns an `IO` after it's called. I've written out the *conceptual* return values to better express the `IO`, though, in reality, it will always be `{ $value: [Function] }`. When we `map` over our `IO`, we stick that function at the end of a composition which, in turn, becomes the new `$value` and so on. Our mapped functions do not run, they get tacked on the end of a computation we're building up, function by function, like carefully placing dominoes that we don't dare tip over. The result is reminiscent of Gang of Four's command pattern or a queue.

Take a moment to channel your functor intuition. If we see past the implementation details, we should feel right at home mapping over any container no matter its quirks or idiosyncrasies. We have the functor laws, which we will explore toward the end of the chapter, to thank for this pseudo-psychic power. At any rate, we can finally play with impure values without sacrificing our precious purity.

Now, we've caged the beast, but we'll still have to set it free at some point. Mapping over our `IO` has built up a mighty impure computation and running it is surely going to disturb the peace. So where and when can we pull the trigger? Is it even possible to run our `IO` and still wear white at our wedding? The answer is yes, if we put the onus on the calling code. Our pure code, despite the nefarious plotting and scheming, maintains its innocence and it's the caller who gets burdened with the responsibility of actually running the effects. Let's see an example to make this concrete.

```js
// url :: IO String
const url = new IO(() => window.location.href);

// toPairs :: String -> [[String]]
const toPairs = compose(map(split('=')), split('&'));

// params :: String -> [[String]]
const params = compose(toPairs, last, split('?'));

// findParam :: String -> IO Maybe [String]
const findParam = key => map(compose(Maybe.of, find(compose(eq(key), head)), params), url);

// -- Impure calling code ----------------------------------------------

// run it by calling $value()!
findParam('searchTerm').$value();
// Just(['searchTerm', 'wafflehouse'])
```

Our library keeps its hands clean by wrapping `url` in an `IO` and passing the buck to the caller. You might have also noticed that we have stacked our containers; it's perfectly reasonable to have a `IO(Maybe([x]))`, which is three functors deep (`Array` is most definitely a mappable container type) and exceptionally expressive.

There's something that's been bothering me and we should rectify it immediately: `IO`'s `$value` isn't really its contained value, nor is it a private property. It is the pin in the grenade and it is meant to be pulled by a caller in the most public of ways. Let's rename this property to `unsafePerformIO` to remind our users of its volatility.

```js
class IO {
  constructor(io) {
    this.unsafePerformIO = io;
  }

  map(fn) {
    return new IO(compose(fn, this.unsafePerformIO));
  }
}
```

There, much better. Now our calling code becomes `findParam('searchTerm').unsafePerformIO()`, which is clear as day to users (and readers) of the application.

`IO` will be a loyal companion, helping us tame those feral impure actions. Next, we'll see a type similar in spirit, but having a drastically different use case.


## Asynchronous Tasks

Callbacks are the narrowing spiral staircase to hell. They are control flow as designed by M.C. Escher. With each nested callback squeezed in between the jungle gym of curly braces and parenthesis, they feel like limbo in an oubliette (how low can we go?!). I'm getting claustrophobic chills just thinking about them. Not to worry, we have a much better way of dealing with asynchronous code and it starts with an "F".

The internals are a bit too complicated to spill out all over the page here so we will use `Data.Task` (previously `Data.Future`) from Quildreen Motta's fantastic [Folktale](https://folktale.origamitower.com/). Behold some example usage:

```js
// -- Node readFile example ------------------------------------------

const fs = require('fs');

// readFile :: String -> Task Error String
const readFile = filename => new Task((reject, result) => {
  fs.readFile(filename, (err, data) => (err ? reject(err) : result(data)));
});

readFile('metamorphosis').map(split('\n')).map(head);
// Task('One morning, as Gregor Samsa was waking up from anxious dreams, he discovered that
// in bed he had been changed into a monstrous verminous bug.')


// -- jQuery getJSON example -----------------------------------------

// getJSON :: String -> {} -> Task Error JSON
const getJSON = curry((url, params) => new Task((reject, result) => {
  $.getJSON(url, params, result).fail(reject);
}));

getJSON('/video', { id: 10 }).map(prop('title'));
// Task('Family Matters ep 15')


// -- Default Minimal Context ----------------------------------------

// We can put normal, non futuristic values inside as well
Task.of(3).map(three => three + 1);
// Task(4)
```

The functions I'm calling `reject` and `result` are our error and success callbacks, respectively. As you can see, we simply `map` over the `Task` to work on the future value as if it was right there in our grasp. By now `map` should be old hat.

If you're familiar with promises, you might recognize the function `map` as `then` with `Task` playing the role of our promise. Don't fret if you aren't familiar with promises, we won't be using them anyhow because they are not pure, but the analogy holds nonetheless.

Like `IO`, `Task` will patiently wait for us to give it the green light before running. In fact, because it waits for our command, `IO` is effectively subsumed by `Task` for all things asynchronous; `readFile` and `getJSON` don't require an extra `IO` container to be pure. What's more, `Task` works in a similar fashion when we `map` over it: we're placing instructions for the future like a chore chart in a time capsule - an act of sophisticated technological procrastination.

To run our `Task`, we must call the method `fork`. This works like `unsafePerformIO`, but as the name suggests, it will fork our process and evaluation continues on without blocking our thread. This can be implemented in numerous ways with threads and such, but here it acts as a normal async call would and the big wheel of the event loop keeps on turning. Let's look at `fork`:

```js
// -- Pure application -------------------------------------------------
// blogPage :: Posts -> HTML
const blogPage = Handlebars.compile(blogTemplate);

// renderPage :: Posts -> HTML
const renderPage = compose(blogPage, sortBy(prop('date')));

// blog :: Params -> Task Error HTML
const blog = compose(map(renderPage), getJSON('/posts'));


// -- Impure calling code ----------------------------------------------
blog({}).fork(
  error => $('#error').html(error.message),
  page => $('#main').html(page),
);

$('#spinner').show();
```

Upon calling `fork`, the `Task` hurries off to find some posts and render the page. Meanwhile, we show a spinner since `fork` does not wait for a response. Finally, we will either display an error or render the page onto the screen depending if the `getJSON` call succeeded or not.

Take a moment to consider how linear the control flow is here. We just read bottom to top, right to left even though the program will actually jump around a bit during execution. This makes reading and reasoning about our application simpler than having to bounce between callbacks and error handling blocks.

Goodness, would you look at that, `Task` has also swallowed up `Either`! It must do so in order to handle futuristic failures since our normal control flow does not apply in the async world. This is all well and good as it provides sufficient and pure error handling out of the box.

Even with `Task`, our `IO` and `Either` functors are not out of a job. Bear with me on a quick example that leans toward the more complex and hypothetical side, but is useful for illustrative purposes.

```js
// Postgres.connect :: Url -> IO DbConnection
// runQuery :: DbConnection -> ResultSet
// readFile :: String -> Task Error String

// -- Pure application -------------------------------------------------

// dbUrl :: Config -> Either Error Url
const dbUrl = ({ uname, pass, host, db }) => {
  if (uname && pass && host && db) {
    return Either.of(`db:pg://${uname}:${pass}@${host}5432/${db}`);
  }

  return left(Error('Invalid config!'));
};

// connectDb :: Config -> Either Error (IO DbConnection)
const connectDb = compose(map(Postgres.connect), dbUrl);

// getConfig :: Filename -> Task Error (Either Error (IO DbConnection))
const getConfig = compose(map(compose(connectDb, JSON.parse)), readFile);


// -- Impure calling code ----------------------------------------------

getConfig('db.json').fork(
  logErr('couldn\'t read file'),
  either(console.log, map(runQuery)),
);
```

In this example, we still make use of `Either` and `IO` from within the success branch of `readFile`. `Task` takes care of the impurities of reading a file asynchronously, but we still deal with validating the config with `Either` and wrangling the db connection with `IO`. So you see, we're still in business for all things synchronous.

I could go on, but that's all there is to it. Simple as `map`.

In practice, you'll likely have multiple asynchronous tasks in one workflow and we haven't yet acquired the full container apis to tackle this scenario. Not to worry, we'll look at monads and such soon, but first, we must examine the maths that make this all possible.


## A Spot of Theory

As mentioned before, functors come from category theory and satisfy a few laws. Let's first explore these useful properties.

```js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

The *identity* law is simple, but important. These laws are runnable bits of code so we can try them on our own functors to validate their legitimacy.

```js
const idLaw1 = map(id);
const idLaw2 = id;

idLaw1(Container.of(2)); // Container(2)
idLaw2(Container.of(2)); // Container(2)
```

You see, they are equal. Next let's look at composition.

```js
const compLaw1 = compose(map(append(' world')), map(append(' cruel')));
const compLaw2 = map(compose(append(' world'), append(' cruel')));

compLaw1(Container.of('Goodbye')); // Container('Goodbye cruel world')
compLaw2(Container.of('Goodbye')); // Container('Goodbye cruel world')
```

In category theory, functors take the objects and morphisms of a category and map them to a different category. By definition, this new category must have an identity and the ability to compose morphisms, but we needn't check because the aforementioned laws ensure these are preserved.

Perhaps our definition of a category is still a bit fuzzy. You can think of a category as a network of objects with morphisms that connect them. So a functor would map the one category to the other without breaking the network. If an object `a` is in our source category `C`, when we map it to category `D` with functor `F`, we refer to that object as `F a` (If you put it together what does that spell?!). Perhaps, it's better to look at a diagram:

<img src="images/catmap.png" alt="Categories mapped" />

For instance, `Maybe` maps our category of types and functions to a category where each object may not exist and each morphism has a `null` check. We accomplish this in code by surrounding each function with `map` and each type with our functor. We know that each of our normal types and functions will continue to compose in this new world. Technically, each functor in our code maps to a sub category of types and functions which makes all functors a particular brand called endofunctors, but for our purposes, we'll think of it as a different category.

We can also visualize the mapping of a morphism and its corresponding objects with this diagram:

<img src="images/functormap.png" alt="functor diagram" />

In addition to visualizing the mapped morphism from one category to another under the functor `F`, we see that the diagram commutes, which is to say, if you follow the arrows each route produces the same result. The different routes mean different behavior, but we always end at the same type. This formalism gives us principled ways to reason about our code - we can boldly apply formulas without having to parse and examine each individual scenario. Let's take a concrete example.

```js
// topRoute :: String -> Maybe String
const topRoute = compose(Maybe.of, reverse);

// bottomRoute :: String -> Maybe String
const bottomRoute = compose(map(reverse), Maybe.of);

topRoute('hi'); // Just('ih')
bottomRoute('hi'); // Just('ih')
```

Or visually:

<img src="images/functormapmaybe.png" alt="functor diagram 2" />

We can instantly see and refactor code based on properties held by all functors.

Functors can stack:

```js
const nested = Task.of([Either.of('pillows'), left('no sleep for you')]);

map(map(map(toUpperCase)), nested);
// Task([Right('PILLOWS'), Left('no sleep for you')])
```

What we have here with `nested` is a future array of elements that might be errors. We `map` to peel back each layer and run our function on the elements. We see no callbacks, if/else's, or for loops; just an explicit context. We do, however, have to `map(map(map(f)))`. We can instead compose functors. You heard me correctly:

```js
class Compose {
  constructor(fgx) {
    this.getCompose = fgx;
  }

  static of(fgx) {
    return new Compose(fgx);
  }

  map(fn) {
    return new Compose(map(map(fn), this.getCompose));
  }
}

const tmd = Task.of(Maybe.of('Rock over London'));

const ctmd = Compose.of(tmd);

const ctmd2 = map(append(', rock on, Chicago'), ctmd);
// Compose(Task(Just('Rock over London, rock on, Chicago')))

ctmd2.getCompose;
// Task(Just('Rock over London, rock on, Chicago'))
```

There, one `map`. Functor composition is associative and earlier, we defined `Container`, which is actually called the `Identity` functor. If we have identity and associative composition we have a category. This particular category has categories as objects and functors as morphisms, which is enough to make one's brain perspire. We won't delve too far into this, but it's nice to appreciate the architectural implications or even just the simple abstract beauty in the pattern.


## In Summary

We've seen a few different functors, but there are infinitely many. Some notable omissions are iterable data structures like trees, lists, maps, pairs, you name it. Event streams and observables are both functors. Others can be for encapsulation or even just type modelling. Functors are all around us and we'll use them extensively throughout the book.

What about calling a function with multiple functor arguments? How about working with an order sequence of impure or async actions? We haven't yet acquired the full tool set for working in this boxed up world. Next, we'll cut right to the chase and look at monads.

[Chapter 09: Monadic Onions](ch09.md)

## Exercises

{% exercise %}  
Use `add` and `map` to make a function that increments a value inside a functor.  
  
{% initial src="./exercises/ch08/exercise_a.js#L3;" %}  
```js  
// incrF :: Functor f => f Int -> f Int  
const incrF = undefined;  
```  
  
{% solution src="./exercises/ch08/solution_a.js" %}  
{% validation src="./exercises/ch08/validation_a.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  


---

  
Given the following User object:  
  
```js  
const user = { id: 2, name: 'Albert', active: true };  
```  
  
{% exercise %}  
Use `safeProp` and `head` to find the first initial of the user.  
  
{% initial src="./exercises/ch08/exercise_b.js#L7;" %}  
```js  
// initial :: User -> Maybe String  
const initial = undefined;  
```  
  
{% solution src="./exercises/ch08/solution_b.js" %}  
{% validation src="./exercises/ch08/validation_b.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  


---


Given the following helper functions:

```js
// showWelcome :: User -> String
const showWelcome = compose(concat('Welcome '), prop('name'));

// checkActive :: User -> Either String User
const checkActive = function checkActive(user) {
  return user.active
    ? Either.of(user)
    : left('Your account is not active');
};
```

{% exercise %}  
Write a function that uses `checkActive` and `showWelcome` to grant access or return the error.

{% initial src="./exercises/ch08/exercise_c.js#L15;" %}  
```js
// eitherWelcome :: User -> Either String String
const eitherWelcome = undefined;
```


{% solution src="./exercises/ch08/solution_c.js" %}  
{% validation src="./exercises/ch08/validation_c.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  


---


We now consider the following functions:

```js
// validateUser :: (User -> Either String ()) -> User -> Either String User
const validateUser = curry((validate, user) => validate(user).map(_ => user));

// save :: User -> IO User
const save = user => new IO(() => ({ ...user, saved: true }));
```

{% exercise %}  
Write a function `validateName` which checks whether a user has a name longer than 3 characters
or return an error message. Then use `either`, `showWelcome` and `save` to write a `register`
function to signup and welcome a user when the validation is ok.

Remember either's two arguments must return the same type.

{% initial src="./exercises/ch08/exercise_d.js#L15;" %}  
```js
// validateName :: User -> Either String ()
const validateName = undefined;

// register :: User -> IO String
const register = compose(undefined, validateUser(validateName));
```


{% solution src="./exercises/ch08/solution_d.js" %}  
{% validation src="./exercises/ch08/validation_d.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  
