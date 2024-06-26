# Розділ 07: Hindley-Milner Та Я


## Який У Тебе Тип?
Якщо функціональний світ новий для вас, то дуже скоро ви зрозумієте, що погрузли в сигратурах типах. Типи - це мета мова, яка дозволяє людям з різною базою спілкуватись успішно та ефективно. Для переважній більшості вони написані за допомогою системи "Hindley-Milner" і з якими ми працюватимемо у цьому розділі.

Коли ви працюєте з чистими функціями, типи мають надзвичайну силу, для якої англійська мова не може тримати свічку. Ці сигнатури нашоптують вам у вухо найтаємничиші секрети функції. У одній компактній лінії вони розкривають поведінку та намір. Ми можемо вивести з них "вільні теореми". Типи можуть бути прогнозованими, так що не буде необхідності в явній аннотації типу. Вони можуть бути надзвичайно точними чи лишатись загальними та абстрактними. Вони корисні не лише для перевірок під час компайлінгу, але може так статися, що вони - найкраща з усіх можливих докумнетацій, які доступні. Тому сигнатури типів відіграють дуже важливу роль у функціональному програмуванні, набагато більшу, ніж ви можете очікувати.

JavaScript - динамічна мова, проте це не означає, що ми уникаємо усіх типів. Ми таки працюємо зі строками(strings), числами(numbers), логічними(booleans) і так далі. Справа лише в тому, що немає ніякої мовної інтреграції з ними, тож ми тримаємо їх в наших головах. Але не хвилюйтесь, оскільки ми використовуємо сигнатури для документації - ми можемо лишати коментарі для наших цілей.

Існують інструменти для перевірки типів у JavaScript, такі як [Flow](https://flow.org/) чи типізований діалект, [TypeScript](https://www.typescriptlang.org/). Мета цієї книги озброїти читача знаряддям для написання функціонального коду, тож ми візьмемо стандартну систему типів, яка використовуєтся серед різних функціональних мов програмування.


## Оповіді З Крипти

Починаючи з вкритих пулюкою математичних книжок, у величезному морі білих аркушів, серед звичайних суботніх публікацій, і закінчуючи у самому серці початкового коду, ми знаходимо сигнатури типів Hindley-Milner. Система досить проста, але потребує швидкого пояснення і деяких практичних заннять, щоб цілком поглинути маленьку мову.

```js
// capitalize :: String -> String
const capitalize = s => toUpperCase(head(s)) + toLowerCase(tail(s));

capitalize('smurf'); // 'Smurf'
```

Тут `capitalize` бере `String`(строку) та повертає `String`(строку). Не зважаючи на реалізацію - це саме та сигнатура типу яка нас цікавить.

У HM(_прим.пер.:_ Hindley-Milner) функції написані як `a -> b`, де `a` та `b` - змінні будь яких типів. Тож сигнатура фунції `capitalize` може бути прочитана як "функція від строки до строки". Інакше кажучи, вона бере строку(`String`) як вхідну величину та повертає строку(`String`) як вихідну.

Давайте розглянемо трохи більше сигнатур функцій:

```js
// strLength :: String -> Number
const strLength = s => s.length;

// join :: String -> [String] -> String
const join = curry((what, xs) => xs.join(what));

// match :: Regex -> String -> [String]
const match = curry((reg, s) => s.match(reg));

// replace :: Regex -> String -> String -> String
const replace = curry((reg, sub, s) => s.replace(reg, sub));
```

`strLength` - тут таж сама ідея як і раніше: ми беремо `String` і повертаємо вам `Number`.

Інші можуть трохи вас збентежити спочатку. Без повного розуміння деталей ви завжди можете подивитись на найостанніший тип, який повертається в якості значення. Тож функцію `match` ви можете прочитати так: Вона бере `Regex`(регулярний вираз) та `String`(строку) і повертає вам `[String]`(масив строк). Але тут відбувається одна цікава річ, яку б я хотів зупинитись, на мить, та пояснити, якщо ви не проти.

Для `match` ми можемо згрупувати сигнатури так:

```js
// match :: Regex -> (String -> [String])
const match = curry((reg, s) => s.match(reg));
```

І так, групування останньої частини у дужках відкриває більше інформації. Тепер стає очевидним, що функція, яка приймає `Regex` і повертає нам функцію зі строки(`String`) до масиву строк(`[String]`). Насправді це можливе через карування:  надайте регулярний вирах `Regex` і ми отримуємо функцію, яка очікує на аргумент `String`. Звісно, ми не повинні думати про це таким чином, але добре розуміти, чому останній тип є саме таким, яким він повертається.

```js
// match :: Regex -> (String -> [String])
// onHoliday :: String -> [String]
const onHoliday = match(/holiday/ig);
```

Кожен аргумент відокремлює один тип від плочатку сигнатури. Фунція `onHoliday` - це функція `match`, яка вже має `Regex`.

```js
// replace :: Regex -> (String -> (String -> String))
const replace = curry((reg, sub, s) => s.replace(reg, sub));
```

Як ви можете бачити з усіма дужками у функції `replace`, додаткові описання можуть трохи заважати, саме тому ми просто їх пропускаємо. Якщо ми можемо, ми можемо надати всі аргументи за один раз, тож легше міркувати про це так: функція `replace` приймає регулярний вираз(`Regex`), строку(`String`), іншу строку(`String`) і повертає вам строку(`String`).

І ще тут є кілька моментів:


```js
// id :: a -> a
const id = x => x;

// map :: (a -> b) -> [a] -> [b]
const map = curry((f, xs) => xs.map(f));
```

Функція `id` бере будь-який старий тип `a` і повертає щось з таким самим типом як у `a`. Ми можемо використовувати змінні в типах так само як у коді. Назви змінних `a` та `b` є загальноприятими, але вони довільні і можуть бути замінені будь-якими назвами, які вам більше подобаються. Якщо вони однакові змінні - вони мають бути однакового типу. Це дуже важливе правило, тож давйте повторимо: `a -> b` може бути будь-який ти `a` до будь-якого типу `b`, але `a -> a` означає, що вони мають бути одного типу. Наприклад, функція `id` може бути `String -> String` чи `Numnber -> Number`, але не `String -> Bool`.

Функція `map` використовує типові змінні дуже схоже, але цього разу ми представляємо `b`, яка може бути, а може і не бути одного типу з `a`. Ми можемо прочитати це так: функція `map` бере функція з будь-якого типу `a` до такого ж чи іншого типу `b`, потім бере масив типів `a` і повертає, в якості результату, масив типів `b`.

Сподіваюсь, вас підкорила виразна краса сигнатури йього типу. Вона буквально розповідає нам, що функція робить майже слово в слово. Дано функцію від `a` до `b`, масив елементів `а`, і вона повертає нам масив елементів `b`. Єдиною розумною річчю для функції є викликати кровопролитну функцію на кожному елементі `a`. Все інше буде чистою брехнею.

Можливість розмірковування щодо типів та їх наслідків - це навичка, яка заведе вас далеко в функціональний світ. Не лише журнали, блоги, документи і т.д. стануть більш засвоюваними, але також сигнатури зможуть вам детально розповідати про свою функціональність. Треба практикуватись, щоб мати змогу вільно читати сигнатури, але якщо будете притримуватись цього - багато інформації стане для вас доступною без RTFMing.

Ось ще кілька сигнатур, щоб побачити, чи можете ви розшифрувати їх самостійно.

```js
// head :: [a] -> a
const head = xs => xs[0];

// filter :: (a -> Bool) -> [a] -> [a]
const filter = curry((f, xs) => xs.filter(f));

// reduce :: ((b, a) -> b) -> b -> [a] -> b
const reduce = curry((f, x, xs) => xs.reduce(f, x));
```

Функція `reduce`, напевно, є набільш виразною з усіх. Проте, вона і не така проста, тож не відчувайте дивно, якщо вам доведеться трохи з нею поборотися. Для цікавості, я намагатимусь пояснити англійською мовою, хоча самостійна робота над сигнатурою є набагато більш повчальною.

Ах, тут нічого не відбувається... дивлячись на підпис, ми бачимо, що перший аргумент - це функція, яка очікує на `b` та `a` і видає `b`. До чого можуть привести ці `a` та `b`? Ну, наступні аргументи у сигнатурі є `b` і масив елементів `a`, тому ми можемо тільки припустити, що `b` і кожен з тих елементів `a` будуть передані. Ми також бачимо, що результат функції є `b`, тому розмірковування мають привести нас до висновку, що остаточний результат переданої функції і буде нашим вихідним значенням. Знаючи, що робить `reduce`, можна стверджувати, що наведене вище дослідження є точним.


## Зменшення Ймовірності

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

Це звуження можливості дозволяє нам використовувати пошуковий двигун сигнатури типу, такий як [Hoogle](https://hoogle.haskell.org/), для того, щоб зрозуміти, яка функція буде потім. Інформація, ретельно запакована у сигнатуру є насправді дуже потужною.

## Вільно, Як У Теоремі

Окрім того, виводячи можливості реалізації, подібні міркування надають нам *безкоштовні теореми*. Нижче наведено декілька випадкових теорем, які взяті безпосередньо з [статті Вадлера](http://ttic.uchicago.edu/~dreyer/course/papers/wadler.pdf) на цю тему.

```js
// head :: [a] -> a
compose(f, head) === compose(head, map(f));

// filter :: (a -> Bool) -> [a] -> [a]
compose(map(f), filter(compose(p, f))) === compose(filter(p), map(f));
```


Вам не потрібен жодний код, щоб зрозуміти ці теореми, оскільки вони випливають прямо з типів. Перша теорема каже, що якщо ми візьмемо `head` нашого масиву, потім запустимо певну функцію `f` - це буде еквівалентним, і, до речі, набагато швидшим, ніж якщо ми спочатку проженемо `map(f)` на кожному елементі і тільки потім візьмемо `head` від результату.

Ви можете подумати, що то просто загальний сенс. Але, коли я востаннє перевіряв - з'ясувалось, що компютери не мають загального сенсу. Насправді, вони мають формальний шлях автоматизації подібних роду оптимізації коду. Математика має спосіб формалізації інтуїтивно зрозумілого, що є корисним у жорсткому середовищі комп'ютерної логіки.

Теорема для функції `filter` схожа. Вона каже, що, якщо ми робимо композицію з `f` та `p`, щоб перевірити, яка має бути відфільтрованою, після цього застосовуємо `f` за допомогою `map`(пам'ятайте, `filter` не трансформуватиме елементи - його сигнатура чітко зазначає, що `a`не буде зачіпатись), то це буде завжди еквівалентно проходженню за допомогою `map` по нашій `f` і потім відфільтровування результату за допомогою предикату `p`. 

Це лише два приклади, але ви можете застосувати подібне розмірковування до будь-якої поліморфічної сигнатури і це завжди працюватиме. У JavaScript існують деякі інструменти для оголошення правил перезапису. Також це можна зробити за допомогою функції `compose`. Фрукт висить низько і можливості нескінченні.

## Обмеження

І останнє, що хотілося б відмітити, це те, що ми можемо прив'язувати типи до інтерфейсів.

```js
// sort :: Ord a => [a] -> [a]
```

Що ми можемо бачити зліва у нашій arrow-функції, це те, що тут є констатація факту: `a` повинно мати тип `Ord`. Чи іншими словами, `a` повинно впроваджувати інтерфейс `Ord`. Що за `Ord` такий і звідки він взявся? У типізованій мові це був би визначений інтерфейс, який казав би, що ми можемо впорядковувати значення. Це не лише каже нам більше про `a`, і що наша `sort`-функція робить, але також обмежує домейн. Ми називаємо такі визначення інтерфейсів - *прив'язування типів*.

```js
// assertEqual :: (Eq a, Show a) => a -> a -> Assertion
```

Тут ми маємо дві прив'язки: `Eq` та `Snow`. Вони забезпечать нам перевірку рівності наших `a` і виведуть різницю, якщо вони не рівні.

Ми побачимо більше прикладів прив'язки і ідея набуде більш визначених форм у подальших розділах.

## В завершення

Сигнатура типів Hindley-Milner є всюдисущою у світі функціонального програмування. І, не дивлячись на те, що їх легко читати та писати, потрібен час, щоб опанувати техніку розуміннія програм через самі тільки сигнатури. Віднині, ми будемо додавати сигнатури типів до кожної строки коду.

[Chapter 8: Tupperware](ch08-uk.md)
