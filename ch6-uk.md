# Розділ 6: Приклад Застосування

## Декларативне написання коду

Ми з вами збираємось змінити наш світогляд. Відтепер, ми приминимо казати комп'ютеру, як робити свою роботу, і натомість ми почнемо писати специфікацію до того, що ми хочемо мати в якості результату. Я переконаний, що ви з'ясуєте, що це менш стресово, ніж намагатись проконтролювати одночасно усе на найдрібніших рівнях.

Декларативність, на противагу імперативності, означає, що ми писатимемо вирази, а не покрокові інструкції.

Подумайте про SQL. Там немає "спочатку зроби це, а потім оте". Там є один вираз, який вкузує, щоб ми хотіли отримати з бази даних. Ми не вирішуємо, як зробити те, що воно робить. Коли база даних оновлена та SQL двигун оптимізовано, ми не повинні змінювати наш запит. Це тому, що існує багато шляхів інтерпретувати наші вимоги та досягти того ж самого результату. 

Для декого, в тому числі і для мене, важко одразу охопити концепцію декларативного написання коду, тому, щоб більше проникнутись цим, давайте розглянемо кілька прикладів.

```js
// імперативно
var makes = [];
for (var i = 0; i < cars.length; i++) {
	makes.push(cars[i].make);
}


// декларативно
var makes = cars.map(function(car) { return car.make; });
```

Імперативний цикл має спочатку ініціалізувати масив. Інтерпретатор має оцінити це твердження, перед тим як продовжувати рух. Потім цикл напряму перебирає список автомобілів, вручну збільшуючи лічильник і демонрує нам свою шматки та шматочки у своєму вульгарному відтворенні явної ітерації.

Версія з `map` - це один вираз. Він не потребує жодного порядку чи оцінки. Тут набагато більше свободи у тому, як map функція пребирає і як може складатись повертаємий масив. Це визначає *що*, а не *як*. Отже, він носить блискучий декларативний тюрбан.

До тогож, за бажанням, щоб бути чільш чіткішою і лаконічнішою, map-функція може бути оптимізована і при цьому наш дорогоцінний код не потрібно змінювати.

Тим з вас, хто думає "Так, але ж це набагато швидше написати імперативний цикл", я пропоную трохи позайматись самоосвітою і ознайомитись з тим, як JavaScript двигун оптимізує ваш код. Ось [приголомшливе відео, яке може пролити трохи світла](https://www.youtube.com/watch?v=65-RbBwZQdU)

Ось інший приклад.

```js
// імперативно
var authenticate = function(form) {
	var user = toUser(form);
	return logIn(user);
};

// декларативно
var authenticate = compose(logIn, toUser);
```

Хоча в імперативній версії немає нічого стовідсотково невірного, в ній все ще присутня поетапна оцінка. Вираз `compose` просто констатує факт: функція authenticate - це композиція `toUser` та `logIn`. Знову ж таки, це лишає нам простір для зміни коду підтримки та призводить до того, що код нашої програми являється специфікацією високго рівня.

Оскільки ми не задаємо послідовність оцінки коду, декларативне написання коду забезпечує паралельне обчислення. Це, в поєднанні з чистими функціями, демонструє чому функціональне програмування є хорошим варіантом для паралельного майбутнього - нам не потрібно робити нічого особливого, щоб досягти паралельних систем.

## A flickr of functional programming

We will now build an example application in a declarative, composable way. We'll still cheat and use side effects for now, but we'll keep them minimal and separate from our pure codebase. We are going to build a browser widget that sucks in flickr images and displays them. Let's start by scaffolding the app. Here's the html:


```html
<!DOCTYPE html>
<html>
<head>
<script src="https://cdnjs.cloudflare.com/ajax/libs/require.js/2.1.11/require.min.js"></script>
<script src="flickr.js"></script>
</head>
<body></body>
</html>
```

And here's the flickr.js skeleton:

```js
requirejs.config({
paths: {
ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
},
});

require([
		'ramda',
		'jquery',
],
function(_, $) {
var trace = _.curry(function(tag, x) {
		console.log(tag, x);
		return x;
		});
// app goes here
});
```

We're pulling in [ramda](http://ramdajs.com) instead of lodash or some other utility library. It includes `compose`, `curry`, and more. I've used requirejs, which may seem like overkill, but we'll be using it throughout the book and consistency is key. Also, I've started us off with our nice `trace` function for easy debugging.

Now that that's out of the way, on to the spec. Our app will do 4 things.

1. Construct a url for our particular search term
2. Make the flickr api call
3. Transform the resulting json into html images
4. Place them on the screen

There are 2 impure actions mentioned above. Do you see them? Those bits about getting data from the flickr api and placing it on the screen. Let's define those first so we can quarantine them.

```js
var Impure = {
getJSON: _.curry(function(callback, url) {
			 $.getJSON(url, callback);
			 }),

setHtml: _.curry(function(sel, html) {
			 $(sel).html(html);
			 })
};
```

Here we've simply wrapped jQuery's methods to be curried and we've swapped the arguments to a more favorable position. I've namespaced them with `Impure` so we know these are dangerous functions. In a future example, we will make these two functions pure.

Next we must construct a url to pass to our `Impure.getJSON` function.

```js
var url = function(term) {
	return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' +
		term + '&format=json&jsoncallback=?';
};
```

There are fancy and overly complex ways of writing `url` pointfree using monoids(we'll learn about these later) or combinators. We've chosen to stick with a readable version and assemble this string in the normal pointful fashion.

Let's write an app function that makes the call and places the contents on the screen.

```js
var app = _.compose(Impure.getJSON(trace('response')), url);

app('cats');
```

This calls our `url` function, then passes the string to our `getJSON` function, which has been partially applied with `trace`. Loading the app will show the response from the api call in the console.

<img src="images/console_ss.png" alt="console response" />

We'd like to construct images out of this json. It looks like the srcs are buried in `items` then each `media`'s `m` property.

Anyhow, to get at these nested properties we can use a nice universal getter function from ramda called `_.prop()`. Here's a homegrown version so you can see what's happening:

```js
var prop = _.curry(function(property, object) {
		return object[property];
		});
```

It's quite dull actually. We just use `[]` syntax to access a property on whatever object. Let's use this to get at our srcs.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));
```

Once we gather the `items`, we must `map` over them to extract each media url. This results in a nice array of srcs. Let's hook this up to our app and print them on the screen.

```js
var renderImages = _.compose(Impure.setHtml('body'), srcs);
var app = _.compose(Impure.getJSON(renderImages), url);
```

All we've done is make a new composition that will call our `srcs` and set the body html with them. We've replaced the `trace` call with `renderImages` now that we have something to render besides raw json. This will crudely display our srcs directly in the body.

Our final step is to turn these srcs into bonafide images. In a bigger application, we'd use a template/dom library like Handlebars or React. For this application though, we only need an img tag so let's stick with jQuery.

```js
var img = function(url) {
	return $('<img />', {
src: url
});
};
```

jQuery's `html()` method will accept an array of tags. We only have to transform our srcs into images and send them along to `setHtml`.

```js
var images = _.compose(_.map(img), srcs);
var renderImages = _.compose(Impure.setHtml('body'), images);
var app = _.compose(Impure.getJSON(renderImages), url);
```

And we're done!

<img src="images/cats_ss.png" alt="cats grid" />

Here is the finished script:
```js
requirejs.config({
paths: {
ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min',
},
});

require([
		'ramda',
		'jquery',
],
function(_, $) {
////////////////////////////////////////////
// Utils

var Impure = {
getJSON: _.curry(function(callback, url) {
			 $.getJSON(url, callback);
			 }),

setHtml: _.curry(function(sel, html) {
			 $(sel).html(html);
			 }),
};

var img = function(url) {
return $('<img />', {
src: url,
});
};

var trace = _.curry(function(tag, x) {
		console.log(tag, x);
		return x;
		});

////////////////////////////////////////////

var url = function(t) {
	return 'http://api.flickr.com/services/feeds/photos_public.gne?tags=' +
		t + '&format=json&jsoncallback=?';
};

var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

var images = _.compose(_.map(img), srcs);

var renderImages = _.compose(Impure.setHtml('body'), images);

var app = _.compose(Impure.getJSON(renderImages), url);

app('cats');
});
```

Now look at that. A beautifully declarative specification of what things are, not how they come to be. We now view each line as an equation with properties that hold. We can use these properties to reason about our application and refactor.

## A Principled Refactor

There is an optimization available - we map over each item to turn it into a media url, then we map again over those srcs to turn them into img tags. There is a law regarding map and composition:


```js
// map's composition law
var law = compose(map(f), map(g)) === map(compose(f, g));
```

We can use this property to optimize our code. Let's have a principled refactor.

```js
// original code
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

var images = _.compose(_.map(img), srcs);

```

Let's line up our maps. We can inline the call to `srcs` in `images` thanks to equational reasoning and purity.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(img), _.map(mediaUrl), _.prop('items'));
```

Now that we've lined up our `map`'s we can apply the composition law.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(_.compose(img, mediaUrl)), _.prop('items'));
```

Now the bugger will only loop once while turning each item into an img. Let's just make it a little more readable by extracting the function out.

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var mediaToImg = _.compose(img, mediaUrl);

var images = _.compose(_.map(mediaToImg), _.prop('items'));
```

## In Summary

We have seen how to put our new skills into use with a small, but real world app. We've used our mathematical framework to reason about and refactor our code. But what about error handling and code branching? How can we make the whole application pure instead of merely namespacing destructive functions? How can we make our app safer and more expressive? These are the questions we will tackle in part 2.

[Chapter 7: Hindley-Milner and Me](ch7.md)
