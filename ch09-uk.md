# Chapter 09: Монадна Цибуля

## Фабрика Направлених Функторів

Перш ніж ми підемо далі, я маю зробити зізнання: я не був повністю чесним щодо методу `of`, який ми помістили в кожен з наших типів. Виявляється, він не для того, щоб уникнути ключового слова `new`, а щоб помістити значення в те, що називається *мінімальним стандартним контекстом*. Так, `of` насправді не замінює конструктор - це частина важливого інтерфейсу, який ми називаємо *Направленим (Pointed)*.

> *Направлений функтор* — це функтор з методом `of`

Тут важливою є можливість кинути будь-яке значення в наш тип щоб одразу ж розпочати мапінг.

```js
IO.of('tetris').map(concat(' master'));
// IO('tetris master')

Maybe.of(1336).map(add(1));
// Maybe(1337)

Task.of([{ id: 2 }, { id: 3 }]).map(map(prop('id')));
// Task([2,3])

Either.of('The past, present and future walk into a bar...').map(concat('it was tense.'));
// Right('The past, present and future walk into a bar...it was tense.')
```

Якщо ви пам'ятаєте, конструктори `IO` і `Task` очікують функцію як свій аргумент, але `Maybe` і `Either` — ні. Мотивація для цього інтерфейсу полягає в тому, щоб мати загальний, послідовний спосіб поміщення значення у наш функтор без складнощів і специфічних вимог конструкторів. Термін "дефолтний мінімальний контекст" не є точним, але добре передає ідею: ми хочемо підняти будь-яке значення у наш тип і використовувати `map` як зазвичай з очікуваною поведінкою будь-якого функтора.

Одне важливе виправлення, яке я повинен зробити на цьому етапі, це те, що `Left.of` не має сенсу. Кожен функтор повинен мати один спосіб поміщення значення всередину, і для `Either` це `new Right(x)`. Ми визначаємо `of` за допомогою `Right`, тому що якщо наш тип може використовувати `map`, він повинен використовувати `map`. Дивлячись на наведені вище приклади, ми можемо мати уявлення, як зазвичай працює `of`, і `Left` ламає цей шаблон.

Можливо, ви чули про функції, такі як `pure`, `point`, `unit` і `return`. Це різні назви для нашого методу `of`, міжнародної функції-загадки. `of` стане важливим, коли ми почнемо використовувати монади, тому що, як ми побачимо, це наша відповідальність — вручну повертати значення у тип.

Щоб уникнути ключового слова `new`, існує кілька стандартних трюків або бібліотек JavaScript, тому давайте використовувати їх і відтепер використовувати `of` як відповідальна доросла людина. Я рекомендую використовувати екземпляри функтора з бібліотек `folktale`, `ramda` або `fantasy-land`, оскільки вони забезпечують правильний метод `of`, а також хороші конструктори, які не залежать від `new`.


## Змішування Метафор

<img src="images/onion.png" alt="onion" />

Розумієте, окрім космічних буріто (якщо ви чули чутки), монади схожі на цибулю. Дозвольте продемонструвати це на поширеній ситуації:

```js
const fs = require('fs');

// readFile :: String -> IO String
const readFile = filename => new IO(() => fs.readFileSync(filename, 'utf-8'));

// print :: String -> IO String
const print = x => new IO(() => {
  console.log(x);
  return x;
});

// cat :: String -> IO (IO String)
const cat = compose(map(print), readFile);

cat('.git/config');
// IO(IO('[core]\nrepositoryformatversion = 0\n'))
```

Що ми маємо тут, це `IO`, який потрапив всередину іншого `IO`, тому що `print` предвставив другий `IO` під час нашого `map`. Щоб продовжити роботу з нашим рядком, ми повинні використовувати `map(map(f))`, а щоб спостерігати ефект, ми повинні викликати `unsafePerformIO().unsafePerformIO()`.

```js
// cat :: String -> IO (IO String)
const cat = compose(map(print), readFile);

// catFirstChar :: String -> IO (IO String)
const catFirstChar = compose(map(map(head)), cat);

catFirstChar('.git/config');
// IO(IO('['))
```

Хоча приємно бачити, що ми маємо два ефекти, запаковані та готові до використання в нашій програмі, це відчувається так, ніби ми працюємо у двох захисних костюмах, і в результаті отримуємо незручно громіздкий API. Давайте розглянемо іншу ситуацію:

```js
// safeProp :: Key -> {Key: a} -> Maybe a
const safeProp = curry((x, obj) => Maybe.of(obj[x]));

// safeHead :: [a] -> Maybe a
const safeHead = safeProp(0);

// firstAddressStreet :: User -> Maybe (Maybe (Maybe Street))
const firstAddressStreet = compose(
  map(map(safeProp('street'))),
  map(safeHead),
  safeProp('addresses'),
);

firstAddressStreet({
  addresses: [{ street: { name: 'Mulburry', number: 8402 }, postcode: 'WC2N' }],
});
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```

Знову ми бачимо цю ситуацію з вкладеними функторами, де приємно бачити, що у нашій функції є три можливі помилки, але очікувати від того, хто викликав, що він буде використовувати `map` три рази, щоб отримати значення - це трохи зухвало, особливо коли ми тільки познайомилися. Цей шаблон буде з’являтися знову і знову, і саме в таких ситуаціях нам потрібно висвітлити могутній символ монади в нічному небі.

Я сказав, що монади схожі на цибулю, тому що сльози навертаються, коли ми знімаємо кожен шар вкладеного функтора за допомогою `map`, щоб дістатися до внутрішнього значення. Ми можемо витерти очі, глибоко вдихнути і використати метод під назвою `join`.

```js
const mmo = Maybe.of(Maybe.of('nunchucks'));
// Maybe(Maybe('nunchucks'))

mmo.join();
// Maybe('nunchucks')

const ioio = IO.of(IO.of('pizza'));
// IO(IO('pizza'))

ioio.join();
// IO('pizza')

const ttt = Task.of(Task.of(Task.of('sewers')));
// Task(Task(Task('sewers')));

ttt.join();
// Task(Task('sewers'))
```

Якщо у нас є два шари одного типу, ми можемо об'єднати їх за допомогою `join`. Ця здатність об'єднуватися, цей функторний шлюб, є тим, що робить монаду монадою. Давайте перейдемо до повного визначення, використовуючи більш точне формулювання:

> Монади — це направлені функтори, які можуть бути сплощені

Будь-який функтор, який визначає метод `join`, має метод `of` і дотримується кількох законів, є монадою. Визначити `join` не надто складно, тому давайте зробимо це для `Maybe`:

```js
Maybe.prototype.join = function join() {
  return this.isNothing() ? Maybe.of(null) : this.$value;
};
```

Ось, просто як поглинання свого близнюка в утробі. Якщо у нас є `Maybe(Maybe(x))`, то `.$value` просто видалить зайвий шар, і ми можемо безпечно використовувати `map` звідти. Інакше у нас буде лише один `Maybe`, оскільки нічого не було б відображено спочатку.

Тепер, коли у нас є метод `join`, давайте посипемо трохи магічного монадного пилу на приклад `firstAddressStreet` і побачимо його в дії:

```js
// join :: Monad m => m (m a) -> m a
const join = mma => mma.join();

// firstAddressStreet :: User -> Maybe Street
const firstAddressStreet = compose(
  join,
  map(safeProp('street')),
  join,
  map(safeHead), safeProp('addresses'),
);

firstAddressStreet({
  addresses: [{ street: { name: 'Mulburry', number: 8402 }, postcode: 'WC2N' }],
});
// Maybe({name: 'Mulburry', number: 8402})
```

Ми додали `join` всюди, де зустрічали вкладені `Maybe`, щоб вони не виходили з-під контролю. Давайте зробимо те ж саме з `IO`.

```js
IO.prototype.join = function() {
  const $ = this;
  return new IO(() => $.unsafePerformIO().unsafePerformIO());
};
```

Ми просто об'єднуємо виконання двох шарів IO послідовно: зовнішній, потім внутрішній. Зверніть увагу, ми не відмовилися від чистоти, а лише перепакували зайві два шари захисної плівки в одну легшу для відкриття упаковки.

```js
// log :: a -> IO a
const log = x => new IO(() => {
  console.log(x);
  return x;
});

// setStyle :: Selector -> CSSProps -> IO DOM
const setStyle =
  curry((sel, props) => new IO(() => jQuery(sel).css(props)));

// getItem :: String -> IO String
const getItem = key => new IO(() => localStorage.getItem(key));

// applyPreferences :: String -> IO DOM
const applyPreferences = compose(
  join,
  map(setStyle('#main')),
  join,
  map(log),
  map(JSON.parse),
  getItem,
);

applyPreferences('preferences').unsafePerformIO();
// Object {backgroundColor: "green"}
// <div style="background-color: 'green'"/>
```

`getItem` повертає `IO String`, тому ми використовуємо `map` для його розбору. Як `log`, так і `setStyle` повертають `IO`, тому ми повинні використовувати `join`, щоб тримати наше вкладення під контролем.

## My Chain Hits My Chest

<img src="images/chain.jpg" alt="chain" />

You might have noticed a pattern. We often end up calling `join` right after a `map`. Let's abstract this into a function called `chain`.

```js
// chain :: Monad m => (a -> m b) -> m a -> m b
const chain = curry((f, m) => m.map(f).join());

// or

// chain :: Monad m => (a -> m b) -> m a -> m b
const chain = f => compose(join, map(f));
```

We'll just bundle up this map/join combo into a single function. If you've read about monads previously, you might have seen `chain` called `>>=` (pronounced bind) or `flatMap` which are all aliases for the same concept. I personally think `flatMap` is the most accurate name, but we'll stick with `chain` as it's the widely accepted name in JS. Let's refactor the two examples above with `chain`:

```js
// map/join
const firstAddressStreet = compose(
  join,
  map(safeProp('street')),
  join,
  map(safeHead),
  safeProp('addresses'),
);

// chain
const firstAddressStreet = compose(
  chain(safeProp('street')),
  chain(safeHead),
  safeProp('addresses'),
);

// map/join
const applyPreferences = compose(
  join,
  map(setStyle('#main')),
  join,
  map(log),
  map(JSON.parse),
  getItem,
);

// chain
const applyPreferences = compose(
  chain(setStyle('#main')),
  chain(log),
  map(JSON.parse),
  getItem,
);
```

I swapped out any `map/join` with our new `chain` function to tidy things up a bit. Cleanliness is nice and all, but there's more to `chain` than meets the eye - it's more of a tornado than a vacuum. Because `chain` effortlessly nests effects, we can capture both *sequence* and *variable assignment* in a purely functional way.

```js
// getJSON :: Url -> Params -> Task JSON
getJSON('/authenticate', { username: 'stale', password: 'crackers' })
  .chain(user => getJSON('/friends', { user_id: user.id }));
// Task([{name: 'Seimith', id: 14}, {name: 'Ric', id: 39}]);

// querySelector :: Selector -> IO DOM
querySelector('input.username')
  .chain(({ value: uname }) =>
    querySelector('input.email')
      .chain(({ value: email }) => IO.of(`Welcome ${uname} prepare for spam at ${email}`))
  );
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');

Maybe.of(3)
  .chain(three => Maybe.of(2).map(add(three)));
// Maybe(5);

Maybe.of(null)
  .chain(safeProp('address'))
  .chain(safeProp('street'));
// Maybe(null);
```

We could have written these examples with `compose`, but we'd need a few helper functions and this style lends itself to explicit variable assignment via closure anyhow. Instead we're using the infix version of `chain` which, incidentally, can be derived from `map` and `join` for any type automatically: `t.prototype.chain = function(f) { return this.map(f).join(); }`. We can also define `chain` manually if we'd like a false sense of performance, though we must take care to maintain the correct functionality - that is, it must equal `map` followed by `join`. An interesting fact is that we can derive `map` for free if we've created `chain` simply by bottling the value back up when we're finished with `of`. With `chain`, we can also define `join` as `chain(id)`. It may feel like playing Texas Hold em' with a rhinestone magician in that I'm just pulling things out of my behind, but, as with most mathematics, all of these principled constructs are interrelated. Lots of these derivations are mentioned in the [fantasyland](https://github.com/fantasyland/fantasy-land) repo, which is the official specification for algebraic data types in JavaScript.

Anyways, let's get to the examples above. In the first example, we see two `Task`'s chained in a sequence of asynchronous actions - first it retrieves the `user`, then it finds the friends with that user's id. We use `chain` to avoid a `Task(Task([Friend]))` situation.

Next, we use `querySelector` to find a few different inputs and create a welcoming message. Notice how we have access to both `uname` and `email` at the innermost function - this is functional variable assignment at its finest. Since `IO` is graciously lending us its value, we are in charge of putting it back how we found it - we wouldn't want to break its trust (and our program). `IO.of` is the perfect tool for the job and it's why Pointed is an important prerequisite to the Monad interface. However, we could choose to `map` as that would also return the correct type:

```js
querySelector('input.username').chain(({ value: uname }) =>
  querySelector('input.email').map(({ value: email }) =>
    `Welcome ${uname} prepare for spam at ${email}`));
// IO('Welcome Olivia prepare for spam at olivia@tremorcontrol.net');
```

Finally, we have two examples using `Maybe`. Since `chain` is mapping under the hood, if any value is `null`, we stop the computation dead in its tracks.

Don't worry if these examples are hard to grasp at first. Play with them. Poke them with a stick. Smash them to bits and reassemble. Remember to `map` when returning a "normal" value and `chain` when we're returning another functor. In the next chapter, we'll approach `Applicatives` and see nice tricks to make this kind of expressions nicer and highly readable.

As a reminder, this does not work with two different nested types. Functor composition and later, monad transformers, can help us in that situation.

## Power Trip

Container style programming can be confusing at times. We sometimes find ourselves struggling to understand how many containers deep a value is or if we need `map` or `chain` (soon we'll see more container methods). We can greatly improve debugging with tricks like implementing `inspect` and we'll learn how to create a "stack" that can handle whatever effects we throw at it, but there are times when we question if it's worth the hassle.

I'd like to swing the fiery monadic sword for a moment to exhibit the power of programming this way.

Let's read a file, then upload it directly afterward:

```js
// readFile :: Filename -> Either String (Task Error String)
// httpPost :: String -> String -> Task Error JSON
// upload :: Filename -> Either String (Task Error JSON)
const upload = compose(map(chain(httpPost('/uploads'))), readFile);
```

Here, we are branching our code several times. Looking at the type signatures I can see that we protect against 3 errors - `readFile` uses `Either` to validate the input (perhaps ensuring the filename is present), `readFile` may error when accessing the file as expressed in the first type parameter of `Task`, and the upload may fail for whatever reason which is expressed by the `Error` in `httpPost`. We casually pull off two nested, sequential asynchronous actions with `chain`.

All of this is achieved in one linear left to right flow. This is all pure and declarative. It holds equational reasoning and reliable properties. We aren't forced to add needless and confusing variable names. Our `upload` function is written against generic interfaces and not specific one-off apis. It's one bloody line for goodness sake.

For contrast, let's look at the standard imperative way to pull this off:

```js
// upload :: Filename -> (String -> a) -> Void
const upload = (filename, callback) => {
  if (!filename) {
    throw new Error('You need a filename!');
  } else {
    readFile(filename, (errF, contents) => {
      if (errF) throw errF;
      httpPost('/uploads', contents, (errH, json) => {
        if (errH) throw errH;
        callback(json);
      });
    });
  }
};
```

Well isn't that the devil's arithmetic. We're pinballed through a volatile maze of madness. Imagine if it were a typical app that also mutated variables along the way! We'd be in the tar pit indeed.

## Theory

The first law we'll look at is associativity, but perhaps not in the way you're used to it.

```js
// associativity
compose(join, map(join)) === compose(join, join);
```

These laws get at the nested nature of monads so associativity focuses on joining the inner or outer types first to achieve the same result. A picture might be more instructive:

<img src="images/monad_associativity.png" alt="monad associativity law" />

Starting with the top left moving downward, we can `join` the outer two `M`'s of `M(M(M a))` first then cruise over to our desired `M a` with another `join`. Alternatively, we can pop the hood and flatten the inner two `M`'s with `map(join)`. We end up with the same `M a` regardless of if we join the inner or outer `M`'s first and that's what associativity is all about. It's worth noting that `map(join) != join`. The intermediate steps can vary in value, but the end result of the last `join` will be the same.

The second law is similar:

```js
// identity for all (M a)
compose(join, of) === compose(join, map(of)) === id;
```

It states that, for any monad `M`, `of` and `join` amounts to `id`. We can also `map(of)` and attack it from the inside out. We call this "triangle identity" because it makes such a shape when visualized:

<img src="images/triangle_identity.png" alt="monad identity law" />

If we start at the top left heading right, we can see that `of` does indeed drop our `M a` in another `M` container. Then if we move downward and `join` it, we get the same as if we just called `id` in the first place. Moving right to left, we see that if we sneak under the covers with `map` and call `of` of the plain `a`, we'll still end up with `M (M a)` and `join`ing will bring us back to square one.

I should mention that I've just written `of`, however, it must be the specific `M.of` for whatever monad we're using.

Now, I've seen these laws, identity and associativity, somewhere before... Hold on, I'm thinking...Yes of course! They are the laws for a category. But that would mean we need a composition function to complete the definition. Behold:

```js
const mcompose = (f, g) => compose(chain(f), g);

// left identity
mcompose(M, f) === f;

// right identity
mcompose(f, M) === f;

// associativity
mcompose(mcompose(f, g), h) === mcompose(f, mcompose(g, h));
```

They are the category laws after all. Monads form a category called the "Kleisli category" where all objects are monads and morphisms are chained functions. I don't mean to taunt you with bits and bobs of category theory without much explanation of how the jigsaw fits together. The intention is to scratch the surface enough to show the relevance and spark some interest while focusing on the practical properties we can use each day.


## In Summary

Monads let us drill downward into nested computations. We can assign variables, run sequential effects, perform asynchronous tasks, all without laying one brick in a pyramid of doom. They come to the rescue when a value finds itself jailed in multiple layers of the same type. With the help of the trusty sidekick "pointed", monads are able to lend us an unboxed value and know we'll be able to place it back in when we're done.

Yes, monads are very powerful, yet we still find ourselves needing some extra container functions. For instance, what if we wanted to run a list of api calls at once, then gather the results? We can accomplish this task with monads, but we'd have to wait for each one to finish before calling the next. What about combining several validations? We'd like to continue validating to gather the list of errors, but monads would stop the show after the first `Left` entered the picture.

In the next chapter, we'll see how applicative functors fit into the container world and why we prefer them to monads in many cases.

[Chapter 10: Applicative Functors](ch10.md)


## Exercises


Considering a User object as follow:

```js
const user = {
  id: 1,
  name: 'Albert',
  address: {
    street: {
      number: 22,
      name: 'Walnut St',
    },
  },
};
```

{% exercise %}
Use `safeProp` and `map/join` or `chain` to safely get the street name when given a user

{% initial src="./exercises/ch09/exercise_a.js#L16;" %}
```js
// getStreetName :: User -> Maybe String
const getStreetName = undefined;
```


{% solution src="./exercises/ch09/solution_a.js" %}
{% validation src="./exercises/ch09/validation_a.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---


We now consider the following items:

```js
// getFile :: IO String
const getFile = IO.of('/home/mostly-adequate/ch09.md');

// pureLog :: String -> IO ()
const pureLog = str => new IO(() => console.log(str));
```

{% exercise %}
Use getFile to get the filepath, remove the directory and keep only the basename,
then purely log it. Hint: you may want to use `split` and `last` to obtain the
basename from a filepath.

{% initial src="./exercises/ch09/exercise_b.js#L13;" %}
```js
// logFilename :: IO ()
const logFilename = undefined;

```


{% solution src="./exercises/ch09/solution_b.js" %}
{% validation src="./exercises/ch09/validation_b.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}


---

For this exercise, we consider helpers with the following signatures:

```js
// validateEmail :: Email -> Either String Email
// addToMailingList :: Email -> IO([Email])
// emailBlast :: [Email] -> IO ()
```

{% exercise %}
Use `validateEmail`, `addToMailingList` and `emailBlast` to create a function
which adds a new email to the mailing list if valid, and then notify the whole
list.

{% initial src="./exercises/ch09/exercise_c.js#L11;" %}
```js
// joinMailingList :: Email -> Either String (IO ())
const joinMailingList = undefined;
```


{% solution src="./exercises/ch09/solution_c.js" %}
{% validation src="./exercises/ch09/validation_c.js" %}
{% context src="./exercises/support.js" %}
{% endexercise %}
