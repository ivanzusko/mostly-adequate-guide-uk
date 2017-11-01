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

## flickr функціонального програмування

А тепер ми з вами побудуємо приклад програми у декларативному та композиційному стилі. І не дивлячись на те, що ми й досі будемо трохи мухлювати, використовуючи побічні ефекти, але ми триматимемо їх у мінімальній кількості і окрема від нашої чистої кодової бази. Ми з вами збираємось побудувати додаток до браузера, який витягає з flickr зображення та відображає їх. Давайте розпочнемо побудову додатку. Ось розмітка:


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

А ось кістяк flickr.js:

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
// тут буде додаток
});
```

Ми використовуватимемо [ramda](http://ramdajs.com) замість lodash чи якоїсь іншої допоміжної бібліотеки. В ній є `compose`, `curry`, та багато чого іншого. Я використав requirejs, що можливо трохи занадто, але ми використовуватимемо його в усіх розділах книги, тож постійсть - наш ключ. Також я розпочав написання програми з нашої гарненької `trace` функції, для більш легшого відлагодження.

Тепер, коли ми з цим розібрались - перейдемо до специфікації. Наш додаток робитиме 4 речі.

1. Будувати url для нашого певного пошувого слова
2. Робитиме запит до flickr api
3. Перетворюватиме отриманий json у html зображення
4. Відображатиме їх на екрані

Тут є 2 нечисті дії. Ви їх бачите? Ті частини в яких йдеться про запити та отримання даних від API flickr та відображення їх на екрані. Тож давайте спочатку їх визначимо, щоб їх було легше ізолювати.

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

Тут ми просто огорнули jQuery методи так, щоб вони були карровані, а також просто змінили місцями аргументи для більшої зручності. Я виокремив їх як `Impure`, щоб ми знали, що ці функції - небезпечні. У майбутньому прикладі ми зробимо ці функції чистими.

Далі нам потрібно побудувати URL, щоб передати його в нашу функцію `Impure.getJSON`.

```js
var url = function(term) {
	return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' +
		term + '&format=json&jsoncallback=?';
};
```

Існують більш гарні та занадто складні способи написання функції `url` у безточечному стилі, за допомогою моноїдів(ми вивчатимемо про них трохи згодом) чи комбінаторів. Але ми поки обрали більш читабельний варіант та зібрали цю строку у нормальний точечний спосіб.

Давайте напишемо функцію `app`, яка робить запит та відображає контент на екрані.

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

[Розділ 7: Hindley-Milner та я](ch7-uk.md)