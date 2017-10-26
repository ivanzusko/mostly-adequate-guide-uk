# Частина 5: Написання коди за допомогою композиції

## Функціональне господарство

Ось `compose`:

```js
var compose = function(f, g) {
	return function(x) {
		return f(g(x));
	};
};
```

`f` та `g` - функції і `x` це значення яке "проженеться" через них.

Композиція нагадує функціональне господарство. Ви, заводчик функцій, обрали дві функції з рисами, які ви хотіли б поєднати, щоб створити нову. Використання композиції полягає в наступному:

```js
var toUpperCase = function(x) {
	return x.toUpperCase();
};
var exclaim = function(x) {
	return x + '!';
};
var shout = compose(exclaim, toUpperCase);

shout("send in the clowns");
//=> "SEND IN THE CLOWNS!"
```

Композиція двох функцій повертає нову функцію. У цілком має сенс: поєднання двох одиниць якогось типу (у цьому фипадку функція) має призвести до появи нової одиниці того ж типу. Ви не з'єднуєте дві детальки Lego, щоб отримати Lincoln. Є одна теорія, основний закон, який ми відкриємо свого часу.

У нашому визначенні функції `compose` функція `g` буде виконана перед функцією `f`, утворюючи напрямок предачі даних зправа наліво. Так набагато зручніше читати, ніж вкладання низки викликів функцій. Без `compose` попередній код можна зобразити так:

```js
var shout = function(x) {
	return exclaim(toUpperCase(x));
};
```

Замість руху зсередини назовні ми рухаємось зправа наліво, що, як мені здається, є кроком у напрямку "на ліво"(Буу). Давайте розглянемо приклад, де послідовність важлива:

```js
var head = function(x) {
	return x[0];
};
var reverse = reduce(function(acc, x) {
		return [x].concat(acc);
		}, []);
var last = compose(head, reverse);

last(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'uppercut'
```

Функція `reverse` поверне список в зворотньому порядку, в той час як `head` витягає лише початковий елемент. Послідовність функцій у цій композиції має бути очевидною. Ми можемо створити версію зліва направо, однак, ми відображаємо математичну версію набагато чіткіше у тому вигляді, в якому вона наведена вище. Правильно, композиція прямо з математичних книг. Насправді, вже, можливо, пора подивитися на властивість, яка зберігається для будь-якої композиції.

```js
// associativity
var associative = compose(f, compose(g, h)) == compose(compose(f, g), h);
// true
```

Композиція - асоціативна. Це значить, що не важливо, як ви поєднаєте дві композиції одна з одною. Тому, якщо ми вирішили перевести строку у верхній регістр, ми можемо написати так:

```js
compose(toUpperCase, compose(head, reverse));

// or
compose(compose(toUpperCase, head), reverse);
```

А оскільки немає значення, як ми поєднуємо наші виклики `compose` - результат буде тим самим. Це дозволяє нам писати різноманітні композиції і використовувати їх, як наприклад ось тут:

```js
// раніше нам потрібно було писати дві композиції, але оскільки композиція асоціативна, ми можемо передавати в композицію стільки функцій скільки нам заманеться і дозволяти їй вирішувати як їх групувати.
var lastUpper = compose(toUpperCase, head, reverse);

lastUpper(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'UPPERCUT'


var loudLastUpper = compose(exclaim, toUpperCase, head, reverse);

loudLastUpper(['jumpkick', 'roundhouse', 'uppercut']);
//=> 'UPPERCUT!'
```

Застовування асоціативної властивості композиції дає нам гнучкість і впевненість, що результат лишиться однаковим. Трохи сткалніше визначення композиції включено у допоміжні бібліотеки для цієї книги і є звичайним визначенням яке ви зможете зустріти у таких бібліотеках як [lodash][lodash-website], [underscore][underscore-website], та [ramda][ramda-website].

Одна приємна перевага асоціативності це те, що будь-яка група функцій може бути витягнута і згрупована разом у їхню особисту композицію. Давайте трохи пограємось з переробкою нашого попереднього прикладу:

```js
var loudLastUpper = compose(exclaim, toUpperCase, head, reverse);

// або
var last = compose(head, reverse);
var loudLastUpper = compose(exclaim, toUpperCase, last);

// або
var last = compose(head, reverse);
var angry = compose(exclaim, toUpperCase);
var loudLastUpper = compose(angry, last);

// більше варіацій...
```

Тут немає ні правильних ні хибних відповідей - ми просто поєднуємо наші детальки lego таким чином, яким нам хочеться. Зазвичай, найкраще групувати речі таким чином, щоб їх можна було перевикористати в подальшому, наприклад `last` і `angry`. Хто знайомий з книгою Фаулера(пер.: _Fowler_) "[Refactoring][refactoring-book]", той може впізнати у цьому процесі "[метод вилучення][extract-method-refactor]"...окрім того, що не потрібно хвилюватись про стан програми.

## Безточечність 

Безточечний стиль означає - ніколи не потрібно повідомляти ваші дані. Перепрошую. Це означає, що функції ніколи не згадують дані над якими вони працюють. Функції першого класу, каррування та композиція, співпрацюють над створенням цього стилю.

```js
//не безточечна, бо ми згадуємо дані: word
var snakeCase = function(word) {
	return word.toLowerCase().replace(/\s+/ig, '_');
};

//безточечна
var snakeCase = compose(replace(/\s+/ig, '_'), toLowerCase);
```

Бачите як ми частково застосували `replace`? Що ми робимо, так це проганяємо наші дані через кожну функцію з одного аргументу. Каррування дозволяє нам підготувати кожну функцію лише на прийом її даних, виконання певних маніпуляцій над ними та їх передачу далі. Ще на що варто звернути увагу - це те, що нам не потрібні дані, щоб побудувати нашу функцію у безточечній версії, в той час як в точечній версії ми повинні мати наші дані(`word`) перед тим, як почати щось робити.

Давайте розглянемо інший приклад.

```js
//не безточечно, бо ми згадуємо дані: name
var initials = function(name) {
	return name.split(' ').map(compose(toUpperCase, head)).join('. ');
};

//безточечно
var initials = compose(join('. '), map(compose(toUpperCase, head)), split(' '));

initials("hunter stockton thompson");
// 'H. S. T'
```

Безточечний код, знову ж таки, може допомогти нам прибрати непотрібні імена та бути нам більш лаконічними та загальними. Безточечність - це гарний лакмусовий папірець, щоб перевіриьти код на наявність функціонального підходу, оскільки це дозволяє нам знати, що у нас є невеликі функції, які перетворюють вхідні величини на вихідні. Наприклад, не можливо побудувати композицію з `while` циклом. Однак, будьте обачні, безточечність - це лезо з твома загостреними сторонами, які можуть ввести в оману. Не весь функціональний код безточечний, і це абсолютно нормально. Проте ми намагатимемось використовувати безточечність усюди де тільки можливо, а де не зможемо - використовуватимемо звичайні функції.

## Debugging
A common mistake is to compose something like `map`, a function of two arguments, without first partially applying it.

```js
//wrong - we end up giving angry an array and we partially applied map with god knows what.
var latin = compose(map, angry, reverse);

latin(['frog', 'eyes']);
// error


// right - each function expects 1 argument.
var latin = compose(map(angry), reverse);

latin(['frog', 'eyes']);
// ['EYES!', 'FROG!'])
```

If you are having trouble debugging a composition, we can use this helpful, but impure trace function to see what's going on.

```js
var trace = curry(function(tag, x) {
		console.log(tag, x);
		return x;
		});

var dasherize = compose(join('-'), toLower, split(' '), replace(/\s{2,}/ig, ' '));

dasherize('The world is a vampire');
// TypeError: Cannot read property 'apply' of undefined
```

Something is wrong here, let's `trace`

```js
var dasherize = compose(join('-'), toLower, trace('after split'), split(' '), replace(/\s{2,}/ig, ' '));
// after split [ 'The', 'world', 'is', 'a', 'vampire' ]
```

Ah! We need to `map` this `toLower` since it's working on an array.

```js
var dasherize = compose(join('-'), map(toLower), split(' '), replace(/\s{2,}/ig, ' '));

dasherize('The world is a vampire');

// 'the-world-is-a-vampire'
```

The `trace` function allows us to view the data at a certain point for debugging purposes. Languages like haskell and purescript have similar functions for ease of development.

Composition will be our tool for constructing programs and, as luck would have it, is backed by a powerful theory that ensures things will work out for us. Let's examine this theory.


## Category theory

Category theory is an abstract branch of mathematics that can formalize concepts from several different branches such as set theory, type theory, group theory, logic, and more. It primarily deals with objects, morphisms, and transformations, which mirrors programming quite closely. Here is a chart of the same concepts as viewed from each separate theory.

<img src="images/cat_theory.png" alt="category theory" />

Sorry, I didn't mean to frighten you. I don't expect you to be intimately familiar with all these concepts. My point is to show you how much duplication we have so you can see why category theory aims to unify these things.

In category theory, we have something called... a category. It is defined as a collection with the following components:

* A collection of objects
* A collection of morphisms
* A notion of composition on the morphisms
* A distinguished morphism called identity

Category theory is abstract enough to model many things, but let's apply this to types and functions, which is what we care about at the moment.

**A collection of objects**
The objects will be data types. For instance, ``String``, ``Boolean``, ``Number``, ``Object``, etc. We often view data types as sets of all the possible values. One could look at ``Boolean`` as the set of `[true, false]` and ``Number`` as the set of all possible numeric values. Treating types as sets is useful because we can use set theory to work with them.


**A collection of morphisms**
The morphisms will be our standard every day pure functions.

**A notion of composition on the morphisms**
This, as you may have guessed, is our brand new toy - `compose`. We've discussed that our `compose` function is associative which is no coincidence as it is a property that must hold for any composition in category theory.

Here is an image demonstrating composition:

<img src="images/cat_comp1.png" alt="category composition 1" />
<img src="images/cat_comp2.png" alt="category composition 2" />

Here is a concrete example in code:

```js
var g = function(x) {
	return x.length;
};
var f = function(x) {
	return x === 4;
};
var isFourLetterWord = compose(f, g);
```

**A distinguished morphism called identity**
Let's introduce a useful function called `id`. This function simply takes some input and spits it back at you. Take a look:

```js
var id = function(x) {
	return x;
};
```

You might ask yourself "What in the bloody hell is that useful for?". We'll make extensive use of this function in the following chapters, but for now think of it as a function that can stand in for our value - a function masquerading as every day data.

`id` must play nicely with compose. Here is a property that always holds for every unary (unary: a one-argument function) function f:

```js
// identity
compose(id, f) == compose(f, id) == f;
// true
```

Hey, it's just like the identity property on numbers! If that's not immediately clear, take some time with it. Understand the futility. We'll be seeing `id` used all over the place soon, but for now we see it's a function that acts as a stand in for a given value. This is quite useful when writing pointfree code.

So there you have it, a category of types and functions. If this is your first introduction, I imagine you're still a little fuzzy on what a category is and why it's useful. We will build upon this knowledge throughout the book. As of right now, in this chapter, on this line, you can at least see it as providing us with some wisdom regarding composition - namely, the associativity and identity properties.

What are some other categories, you ask? Well, we can define one for directed graphs with nodes being objects, edges being morphisms, and composition just being path concatenation. We can define with Numbers as objects and `>=` as morphisms (actually any partial or total order can be a category). There are heaps of categories, but for the purposes of this book, we'll only concern ourselves with the one defined above. We have sufficiently skimmed the surface and must move on.


## In Summary
Composition connects our functions together like a series of pipes. Data will flow through our application as it must - pure functions are input to output after all, so breaking this chain would disregard output, rendering our software useless.

We hold composition as a design principle above all others. This is because it keeps our app simple and reasonable. Category theory will play a big part in app architecture, modelling side effects, and ensuring correctness.

We are now at a point where it would serve us well to see some of this in practice. Let's make an example application.

[Chapter 6: Example Application](ch6.md)

## Exercises

```js
var _ = require('ramda');
var accounting = require('accounting');

// Example Data
var CARS = [{
name: 'Ferrari FF',
	      horsepower: 660,
	      dollar_value: 700000,
	      in_stock: true,
}, {
name: 'Spyker C12 Zagato',
	      horsepower: 650,
	      dollar_value: 648000,
	      in_stock: false,
}, {
name: 'Jaguar XKR-S',
	      horsepower: 550,
	      dollar_value: 132000,
	      in_stock: false,
}, {
name: 'Audi R8',
	      horsepower: 525,
	      dollar_value: 114200,
	      in_stock: false,
}, {
name: 'Aston Martin One-77',
	      horsepower: 750,
	      dollar_value: 1850000,
	      in_stock: true,
}, {
name: 'Pagani Huayra',
	      horsepower: 700,
	      dollar_value: 1300000,
	      in_stock: false,
}];

// Exercise 1:
// ============
// Use _.compose() to rewrite the function below. Hint: _.prop() is curried.
var isLastInStock = function(cars) {
	var last_car = _.last(cars);
	return _.prop('in_stock', last_car);
};

// Exercise 2:
// ============
// Use _.compose(), _.prop() and _.head() to retrieve the name of the first car.
var nameOfFirstCar = undefined;


// Exercise 3:
// ============
// Use the helper function _average to refactor averageDollarValue as a composition.
var _average = function(xs) {
	return _.reduce(_.add, 0, xs) / xs.length;
}; // <- leave be

var averageDollarValue = function(cars) {
	var dollar_values = _.map(function(c) {
			return c.dollar_value;
			}, cars);
	return _average(dollar_values);
};


// Exercise 4:
// ============
// Write a function: sanitizeNames() using compose that returns a list of lowercase and underscored car's names: e.g: sanitizeNames([{name: 'Ferrari FF', horsepower: 660, dollar_value: 700000, in_stock: true}]) //=> ['ferrari_ff'].

var _underscore = _.replace(/\W+/g, '_'); //<-- leave this alone and use to sanitize

var sanitizeNames = undefined;


// Bonus 1:
// ============
// Refactor availablePrices with compose.

var availablePrices = function(cars) {
	var available_cars = _.filter(_.prop('in_stock'), cars);
	return available_cars.map(function(x) {
			return accounting.formatMoney(x.dollar_value);
			}).join(', ');
};


// Bonus 2:
// ============
// Refactor to pointfree. Hint: you can use _.flip().

var fastestCar = function(cars) {
	var sorted = _.sortBy(function(car) {
			return car.horsepower;
			}, cars);
	var fastest = _.last(sorted);
	return fastest.name + ' is the fastest';
};
```

[lodash-website]: https://lodash.com/
[underscore-website]: http://underscorejs.org/
[ramda-website]: http://ramdajs.com/
[refactoring-book]: http://martinfowler.com/books/refactoring.html
[extract-method-refactor]: http://refactoring.com/catalog/extractMethod.html
