# Розділ 11: Знову перетворення, природньо

Ми збираємося обговорити *природні перетворення* в контексті практичної корисності в повсякденному коді. Так вже сталось, що вони є основою теорії категорій і абсолютно незамінні при застосуванні математики для розмірковування про наш код і його рефакторинг. Тому, я вважаю своїм обов'язком повідомити вас про прикру несправедливість, свідками якої ви станете, безсумнівно, через обмеженість моїх можливостей. Почнемо.

## Прокляття Цьому Гнізду

Я хотів би торкнутися питання гніздування (Тут гра слів, бо "nesting" може також бути перекладене як "вкладеність"). Не інстинктивне бажання, яке відчувають майбутні батьки, коли вони прибирають і переставляють речі з нав'язливою ідеєю, але... ну, насправді, якщо подумати, то це не так вже й далеко від істини, як ми побачимо в наступних розділах... У будь-якому випадку, що я маю на увазі під *вкладеністю*, це коли два або більше різних типи зібрані разом навколо значення, як новонародженого, так би мовити.

```js
Right(Maybe('b'));

IO(Task(IO(1000)));

[Identity('bee thousand')];
```

До цього часу нам вдавалося уникнути цього поширеного сценарію за допомогою ретельно підібраних прикладів, але на практиці, коли пишеш код, типи схильні заплутуватись між собою, як навушники під час екзорцизму. Якщо ми не будемо ретельно тримати наші типи організованими по ходу справи, наш код стане таким заплутаним, як бородатий поет в котячому кафе.

## Ситуаційна Комедія

```js
// getValue :: Selector -> Task Error (Maybe String)
// postComment :: String -> Task Error Comment
// validate :: String -> Either ValidationError String

// saveComment :: () -> Task Error (Maybe (Either ValidationError (Task Error Comment)))
const saveComment = compose(
  map(map(map(postComment))),
  map(map(validate)),
  getValue('#comment'),
);
```

Вся банда в зборі, на превеликий жах нашого типового підпису. Дозвольте мені коротко пояснити код. Ми починаємо з отримання користувацького вводу за допомогою `getValue('#comment')`, що є дією, яка отримує текст з елемента. При цьому може виникнути помилка при пошуку елемента або значення рядка може не існувати, тому повертається `Task Error (Maybe String)`. Після цього ми повинні скористатись `map` як для `Task`, так і для `Maybe`, щоб передати наш текст у `validate`, який у свою чергу, повертає нам `Either` `ValidationError` або наш `String`. Далі ми використовуємо мапінг для того, щоб відправити `String` у нашому поточному `Task Error (Maybe (Either ValidationError String))` в `postComment`, який повертає наш кінцевий `Task`.

Яка жахливий безлад. Колаж абстрактних типів, аматорський типовий експресіонізм, поліморфний Поллок, монолітний Мондріан. Існує багато рішень цієї поширеної проблеми. Ми можемо скомпонувати типи в один монструозний контейнер, сортувати та "обʼєднати" (`join`) кілька, гомогенізувати їх, деконструювати їх і так далі. У цьому розділі ми зосередимося на гомогенізації їх за допомогою *природних перетворень*.

## Все Природньо

*Природнє перетворення* — це "морфізм між функторими", тобто функція, яка оперує самими контейнерами. Типово це функція `(Functor f, Functor g) => f a -> g a`. Особливістю цієї функції є те, що ми не можемо з будь-якої причини заглядати всередину нашого функтора. Подумайте про це як про обмін високо засекреченою інформацією — дві сторони не знають, що знаходиться в запечатаному конверті з позначкою "цілком таємно". Це структурна операція. Функторіальна зміна костюма. Формально, *природнє перетворення* — це будь-яка функція, для якої виконується наступне:

<img width=600 src="images/natural_transformation.png" alt="схема природнього перетворення" />

або в коді:

```js
// nt :: (Functor f, Functor g) => f a -> g a
compose(map(f), nt) === compose(nt, map(f));
```

Як діаграма, так і код кажуть одне й те саме: Ми можемо спочатку виконати наше природнє перетворення, а потім використати `map`, або спочатку застосувати `map`, а потім виконати наше природнє перетворення та отримати той самий результат. До речі, це випливає з [вільної теореми](ch07-uk.md#вільно-як-у-теоремі), хоча природні перетворення (і функції) не обмежуються функціями на типах.

## Принципові Перетворення Типів

Як програмісти, ми знайомі з перетвореннями типів. Ми перетворюємо типи, як наприклад, `Strings` в `Booleans` і `Integers` в `Floats` (хоча в JavaScript є тільки `Numbers`). Різниця тут полягає в тому, що ми працюємо з алгебраїчними контейнерами і у нас є деяка теорія в нашому розпорядженні.

Давайте розглянемо деякі з них як приклади:

```js
// idToMaybe :: Identity a -> Maybe a
const idToMaybe = x => Maybe.of(x.$value);

// idToIO :: Identity a -> IO a
const idToIO = x => IO.of(x.$value);

// eitherToTask :: Either a b -> Task a b
const eitherToTask = either(Task.rejected, Task.of);

// ioToTask :: IO a -> Task () a
const ioToTask = x => new Task((reject, resolve) => resolve(x.unsafePerform()));

// maybeToTask :: Maybe a -> Task () a
const maybeToTask = x => (x.isNothing ? Task.rejected() : Task.of(x.$value));

// arrayToMaybe :: [a] -> Maybe a
const arrayToMaybe = x => Maybe.of(x[0]);
```

Бачите ідею? Ми просто змінюємо один функтор на інший. Нам дозволено втрачати інформацію по ходу справи, доки значення, яке ми будемо мапити, не втрачається в процесі зміни форми. У цьому полягає вся суть: `map` повинен продовжувати працювати, відповідно до нашого визначення, навіть після перетворення.

Один із способів поглянути на це полягає в тому, що ми трансформуємо наші ефекти. У цьому світлі, ми можемо розглядати `ioToTask` як перетворення синхронного на асинхронне або `arrayToMaybe` від недетермінованості до можливої невдачі. Зверніть увагу, що ми не можемо перетворити асинхронне на синхронне в JavaScript, тому ми не можемо написати `taskToIO` - це було б надприродним перетворенням.

## Функція Заздрість

Припустимо, ми хочемо використовувати деякі функції з іншого типу, як наприклад `sortBy` для `List`. *Природні перетворення* надають чудовий спосіб перетворити в цільовий тип, знаючи, що наш `map` буде працювати належним чином.

```js
// arrayToList :: [a] -> List a
const arrayToList = List.of;

const doListyThings = compose(sortBy(h), filter(g), arrayToList, map(f));
const doListyThings_ = compose(sortBy(h), filter(g), map(f), arrayToList); // law applied
```

Трішки поворушимо носом, тричі стукнемо чарівною паличкою, додамо `arrayToList`, і вуаля! Наш `[a]` стає `List a`, і ми можемо використовувати `sortBy`, якщо хочемо.

Також стає легше оптимізувати або об'єднувати операції, переміщуючи `map(f)` ліворуч від *природнього перетворення*, як показано в `doListyThings_`.

## Isomorphic JavaScript

When we can completely go back and forth without losing any information, that is considered an *isomorphism*. That's just a fancy word for "holds the same data". We say that two types are *isomorphic* if we can provide the "to" and "from" *natural transformations* as proof:

```js
// promiseToTask :: Promise a b -> Task a b
const promiseToTask = x => new Task((reject, resolve) => x.then(resolve).catch(reject));

// taskToPromise :: Task a b -> Promise a b
const taskToPromise = x => new Promise((resolve, reject) => x.fork(reject, resolve));

const x = Promise.resolve('ring');
taskToPromise(promiseToTask(x)) === x;

const y = Task.of('rabbit');
promiseToTask(taskToPromise(y)) === y;
```

Q.E.D. `Promise` and `Task` are *isomorphic*. We can also write a `listToArray` to complement our `arrayToList` and show that they are too. As a counter example, `arrayToMaybe` is not an *isomorphism* since it loses information:

```js
// maybeToArray :: Maybe a -> [a]
const maybeToArray = x => (x.isNothing ? [] : [x.$value]);

// arrayToMaybe :: [a] -> Maybe a
const arrayToMaybe = x => Maybe.of(x[0]);

const x = ['elvis costello', 'the attractions'];

// not isomorphic
maybeToArray(arrayToMaybe(x)); // ['elvis costello']

// but is a natural transformation
compose(arrayToMaybe, map(replace('elvis', 'lou')))(x); // Just('lou costello')
// ==
compose(map(replace('elvis', 'lou')), arrayToMaybe)(x); // Just('lou costello')
```

They are indeed *natural transformations*, however, since `map` on either side yields the same result. I mention *isomorphisms* here, mid-chapter while we're on the subject, but don't let that fool you, they are an enormously powerful and pervasive concept. Anyways, let's move on.

## A Broader Definition

These structural functions aren't limited to type conversions by any means.

Here are a few different ones:

```hs
reverse :: [a] -> [a]

join :: (Monad m) => m (m a) -> m a

head :: [a] -> a

of :: a -> f a
```

The natural transformation laws hold for these functions too. One thing that might trip you up is that `head :: [a] -> a` can be viewed as `head :: [a] -> Identity a`. We are free to insert `Identity` wherever we please whilst proving laws since we can, in turn, prove that `a` is isomorphic to `Identity a` (see, I told you *isomorphisms* were pervasive).

## One Nesting Solution

Back to our comedic type signature. We can sprinkle in some *natural transformations* throughout the calling code to coerce each varying type so they are uniform and, therefore, `join`able.

```js
// getValue :: Selector -> Task Error (Maybe String)
// postComment :: String -> Task Error Comment
// validate :: String -> Either ValidationError String

// saveComment :: () -> Task Error Comment
const saveComment = compose(
  chain(postComment),
  chain(eitherToTask),
  map(validate),
  chain(maybeToTask),
  getValue('#comment'),
);
```

So what do we have here? We've simply added `chain(maybeToTask)` and `chain(eitherToTask)`. Both have the same effect; they naturally transform the functor our `Task` is holding into another `Task` then `join` the two. Like pigeon spikes on a window ledge, we avoid nesting right at the source. As they say in the city of light, "Mieux vaut prévenir que guérir" - an ounce of prevention is worth a pound of cure.

## In Summary

*Natural transformations* are functions on our functors themselves. They are an extremely important concept in category theory and will start to appear everywhere once more abstractions are adopted, but for now, we've scoped them to a few concrete applications. As we saw, we can achieve different effects by converting types with the guarantee that our composition will hold. They can also help us with nested types, although they have the general effect of homogenizing our functors to the lowest common denominator, which in practice, is the functor with the most volatile effects (`Task` in most cases).

This continual and tedious sorting of types is the price we pay for having materialized them - summoned them from the ether. Of course, implicit effects are much more insidious and so here we are fighting the good fight. We'll need a few more tools in our tackle before we can reel in the larger type amalgamations. Next up, we'll look at reordering our types with *Traversable*.

[Chapter 12: Traversing the Stone](ch12.md)


## Exercises

{% exercise %}  
Write a natural transformation that converts `Either b a` to `Maybe a`
  
{% initial src="./exercises/ch11/exercise_a.js#L3;" %}  
```js  
// eitherToMaybe :: Either b a -> Maybe a  
const eitherToMaybe = undefined;  
```  
  
  
{% solution src="./exercises/ch11/solution_a.js" %}  
{% validation src="./exercises/ch11/validation_a.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  
  
  
---  


```js
// eitherToTask :: Either a b -> Task a b
const eitherToTask = either(Task.rejected, Task.of);
```

{% exercise %}  
Using `eitherToTask`, simplify `findNameById` to remove the nested `Either`.
  
{% initial src="./exercises/ch11/exercise_b.js#L6;" %}  
```js  
// findNameById :: Number -> Task Error (Either Error User)  
const findNameById = compose(map(map(prop('name'))), findUserById);  
```  
  
  
{% solution src="./exercises/ch11/solution_b.js" %}  
{% validation src="./exercises/ch11/validation_b.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  
  
  
---  


As a reminder, the following functions are available in the exercise's context:

```hs
split :: String -> String -> [String]
intercalate :: String -> [String] -> String
```

{% exercise %}  
Write the isomorphisms between String and [Char].
  
{% initial src="./exercises/ch11/exercise_c.js#L8;" %}  
```js  
// strToList :: String -> [Char]  
const strToList = undefined;  
  
// listToStr :: [Char] -> String  
const listToStr = undefined;  
```  
  
  
{% solution src="./exercises/ch11/solution_c.js" %}  
{% validation src="./exercises/ch11/validation_c.js" %}  
{% context src="./exercises/support.js" %}  
{% endexercise %}  
