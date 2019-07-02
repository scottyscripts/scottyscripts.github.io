---
layout: post
title:  "JavaScript Will Be Dead in 5 Years"
date: 2019-07-02
---

## He Was Wrong

At the end of 2015, I was on my first job hunt for a software development role after completing General Assembly's Web Development Immersive Program. I applied at many places and had the opportunity to meet some really interesting people.

One day, I was at some sort of hiring event and talked to this guy from a large company. We started with introductions, but then he asked me what type of work I was looking for. At the time, I was really into front end development, so that was my answer.
He then began to tell me that I'd be wasting my time and said,
> "JavaScript will be dead in 5 years."

![Wrong](https://media.giphy.com/media/ceeN6U57leAhi/giphy.gif)

This was in 2015, so unless anything crazy happens in the next 6 months, it's safe to say this guy was wrong.

## Where Are We At Now?

As much as we all complain about JavaScript *(is this just me?)*, the language continues to grow and I think we can all agree that writing JavaScript in 2019 is better than it's ever been. I think we can also agree that JavaScript isn't going anywhere for the forseeable future.

I've been writing more JavaScript recently and had to look up a feature I was not familiar with when I remembered this guy. Then I started to think about all of the great things that have been added to JavaScript since that point.

In this post, I'm reflecting on some of my favorite improvements to JavaScript over the last few years.

## 1. Template Literals

We were all waiting for string interpolation with JS for a long time. This may the feature on this list I use the most.

```js
let feeling = 'hate'
console.log('I ' + feeling ' hate the old way.')

feeling = 'LOVE'
console.log(`I ${feeling} template literals!`)
```

## 2. Default Function Parameters

Allows parameters to be initialized with default values if one is not provided on function invocation.

Instead of doing:

```js
function greet(name, greeting) {
  greeting = greeting || 'Hello'

  console.log(`${greeting} ${name}`)
}
```

We can do:

```js
function greet(name, greeting = 'Hello') {
  console.log(`${greeting} ${name}`)
}

// uses default set for 'greeting' parameter
greet("Scott") // prints 'Hello Scott',

// can still pass 'greeting' argument
greet("Scott", "Hola") // prints 'Hola Scott'
```


## 3. Destructuring Arrays and Objects

Unpack array and object values directly into variables.

### Arrays

```js
// define variable containing list of top 5 NBA draft picks in 2019
let draftPicks2019 = [
  'Zion Williamson',
  'Ja Morant',
  'RJ Barrett',
  'De\'Andre Hunter',
  'Darius Garland'
]

// You can initialize the variables where values are unpacked,
//   but this is not required
// let firstPick, secondPick, otherPicks

// if I only want the first two picks
[firstPick, secondPick] = draftPicks2019

firstPick
// => 'Zion Williamson'
secondPick
// => 'Ja Morant'

// if I want the first two picks, and an array with remaining picks in my list
[firstPick, secondPick, ...otherPicks] = draftPicks2019

firstPick
// => 'Zion Williamson'
secondPick
// => 'Ja Morant'
otherPicks
// => [ 'RJ Barrett', "De'Andre Hunter", 'Darius Garland' ]
```

### Objects

```js
// define object of 5 NBA draft picks in 2019 and their teams
let draftPicks2019 = {
  pelicans: 'Zion Williamson',
  grizzlies: 'Ja Morant',
  knicks: 'RJ Barrett',
  lakers: 'De\'Andre Hunter',
  cavaliers: 'Darius Garland'
}

// if I only want the Grizzlies' and Cavaliers' draft picks
({ grizzlies, cavaliers } = draftPicks2019)

grizzlies
// => 'Ja Morant'
cavaliers
// => 'Darius Garland'

// if I want the Grizzlies' and Cavaliers' draft picks
//   and an object containing the other values from my object
[grizzlies, cavaliers, ...otherPicks] = draftPicks2019

grizzlies
// => 'Ja Morant'
cavaliers
// => 'Darius Garland'
otherPicks
/* => {
  pelicans: 'Zion Williamson',
  knicks: 'RJ Barrett',
  lakers: "De'Andre Hunter"
}
*/
```

### In Functions

```js
let hero = {
  realName: 'Tony Stark',
  alias: 'Iron Man'
}

let incorrectHero = {
  foo: 'bar'
}

// single parameter
function keepHeroAnonymous(hero) {
  console.log(`Don't tell anyone that ${hero.realName} is actually ${hero.alias}!`)
}

// with object
function keepHeroAnonymous({ realName, alias }) {
  console.log(`Don't tell anyone that ${realName} is actually ${alias}!`)
}

// combine with default function parameters
function keepHeroAnonymous3({
  realName = 'Diana Prince',
  alias = 'Wonder Woman'
}) {
  console.log(`Don't tell anyone that ${realName} is actually ${alias}!`)
}
```

## 4. Arrow Functions

I won't describe all the greatness of arrow functions, but how cool does this look?

```js
let double = (num) => num * 2

double(10) // returns 20
```

## 5. Classes

**JavaScript has classes!**

...well, sort of

JavaScript is still a proto-type based language and classes are just syntactic sugar. I 100% working with the new Class syntax.

### Prototype

```js
function Car(year, brand, color, mileage) {
  this.year = year;
  this.brand = brand;
  this.color = color;
  this.mileage = mileage;
}

// subtracts 'miles' from car's current mileage
Car.prototype.goForDrive = function(miles) {
  this.mileage -= miles;
}
```

### Class

```js
class Car {
  constructor(year, brand, color, mileage){
    this.year = year;
    this.brand = brand;
    this.color = color;
    this.mileage = mileage;
  }

  goForDrive(miles) {
    this.mileage -= miles;
  }
}
```

## Reflecting

I'm going to try to start taking an easy on JavaScript. Of course it has its problems, but I don't think it's going anywhere soon. I think we will be using it for a while to come, and as long as improvements like the ones I pointed out above are being made, I don't mind it.

I highly recommend everyone goes and checks out some of the active proposals for ECMAScript.
https://github.com/tc39/proposals

There are some pretty cool things coming to the language soon!

Hopefully taking a look back has made you less mad at JavaScript.

What's your favorite new feature to come to JavaScript in the last couple years, what are you excited for in the future?
