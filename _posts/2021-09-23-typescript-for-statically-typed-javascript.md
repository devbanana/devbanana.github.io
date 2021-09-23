---
layout: post
title: TypeScript for Statically-Typed JavaScript
date: 2021-09-22 09:53:20 -0400
categories: static-typing
tags: gulp phpstan typescript javascript
---

I'm a PHP developer. Nearly 100% of the coding I do is in PHP.

But every so often, as I'm sure others can relate with, I need to use JavaScript.

My skill with JavaScript is decent at best. But sometimes at work there are just times I need to use it.

One of those times is for building gulp tasks.

If you're not familiar with it, [gulp](https://github.com/gulpjs/gulp) is a tool that helps you to automate development tasks.

Examples include minifying JavaScript, compiling Sass to CSS, building packages, running tests, or pretty much anything else that can be automated.

For a long while I avoided touching gulp if I could possibly help it, because again: not great with JavaScript.

But recently I had to add a task to our gulpfile. I'm doing some integration testing with Google Drive using PHPUnit, and needed a way to fetch the credentials to be used for the test. Node has a Google Authentication library, so I thought it'd be an ideal way to fetch and cache credentials.

But, I like to make sure I'm doing things the best way possible.

## Static Typing

I've been obsessed with [PHPStan](https://github.com/phpstan/phpstan) lately and the importance of static typing. I ensure that 100% of the new PHP code I write passes PHPStan's strictest analysis, and I truly believe it's prevented many potential bugs.

So I thought, if I could have the same thing in JavaScript, I'd have more of an assurance that I wasn't writing buggy code. Not that bugs are impossible with static typing of course, but it'd prevent anything glaring and let me know I was headed on the right track.

I had heard of TypeScript over the years but never done anything with it (remember: not big into JavaScript). But now I finally looked it up and started learning the basics. And indeed, I found that it provided exactly what I was looking for: strictly-typed JavaScript.

All JavaScript is valid TypeScript, but TypeScript adds far more to the language that can be taken advantage of.

So, I decided to convert our gulpfile into TypeScript and wanted to document how that process went, as it wasn't nearly as hard as I would have expected.

**Note**: Initially I was expecting to mostly discuss the conversion process of turning our gulpfile into TypeScript. But my TypeScript example below got away from me so this is going to be a three-part post.

In part 1 (this post), I'll give a brief introduction to TypeScript and why you might want to use it. I'll discuss some of its most basic features and how they can help to catch potential bugs in your code.

Part 2 will discuss some more advanced features of TypeScript including type guards, interfaces/custom types, type predicates, generics, and so on.

In part 3, I'll discuss how I converted our gulpfile to use TypeScript.

## Introduction to TypeScript

If you want to follow along, run `npm install -g typescript` and you'll get the `tsc` executable to analyze and transpile TypeScript code.

For our example, we're going to create a function that accepts a string, and returns the n<sup>th</sup> word of that string.

First, let's start with plain JavaScript:

```typescript
function getWord(text, word) {
  if (word < 1) throw new Error('word must be no less than 1');

  const wordIndex = word - 1;

  // https://regex101.com/r/1zDG8a/2
  const words = text.toLowerCase().match(/\w+(?:'\w+)*/g);
  if (wordIndex >= words.length) {
    throw new Error(`Text only has ${words.length} words`);
  }

  return words[wordIndex];
}

console.log(getWord('This is a test.', 2));

```

If you save this as `index.ts` (remember, all JavaScript is valid TypeScript), then run `tsc index.ts`, you'll notice it creates another file, `index.js`, with pretty much the same code with just a few changes:

```javascript
function getWord(text, word) {
    if (word < 1)
        throw new Error('word must be no less than 1');
    var wordIndex = word - 1;
    // https://regex101.com/r/1zDG8a/2
    var words = text.toLowerCase().match(/\w+(?:'\w+)*/g);
    if (wordIndex >= words.length) {
        throw new Error("Text only has " + words.length + " words");
    }
    return words[wordIndex];
}
console.log(getWord('This is a test.', 2));

```

Now we can execute this file by running:

`node .`

You'll get:

`is`

The only real difference in this `.js` file is that TypeScript made some different decisions about indentation and quotes, which don't really matter.

Other than that, it turned our `const`s into `var`, because by default TypeScript transpiles code into ES3.

If we instead run:

`tsc --target ES2015 index.ts`

Now it maintains our `const`s. This command targets ES6 (or ES2015), but you can target any version of JavaScript up to ES2021.

If you're running in node, then you'll probably want the latest version that your version of node supports. If you're targeting the browser, then you'll probably either want ES2015, or ES5 if you really want to support old browsers.

All this is cool, but it doesn't really show what TypeScript can do.

## Catching Errors

 What happens if someone comes along later on and calls our function in a way we didn't intend?

```typescript
console.log(getWord(['This', 'is', 'a', 'test.'], 2));
```

This throws this error when run in node:

```
$ node .
index.js:6
    const words = text.toLowerCase().match(/\w+(?:'\w+)*/g);
                       ^

TypeError: text.toLowerCase is not a function

```

Of course, because our function expected a string and got an array instead. It'd be nice to be warned of that before the code ever got run.

So now let's try this:

```typescript
function getWord(text: string, word) {
  // ...
}
```

Notice that we changed the `text` parameter into `text: string`.

Now if we run:

`tsc --target ES2015 index.ts`

We'll get:

```
index.ts:15:21 - error TS2345: Argument of type 'string[]' is not assignable to parameter of type 'string'.

15 console.log(getWord(['This', 'is', 'a', 'test.'], 2));
                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Found 1 error.

```

This just means that the function expects a `string`, but got an array of strings (denoted as `string[]`).

You'll notice it still creates `index.js`. You can fix that with the ``--noEmitOnError` flag, which won't create the `.js` file if there are any errors.

But there's one more potential error we're not prepared for yet. Try calling the function like this:

```typescript
console.log(getWord('This is a test.', 'foo'));
```

`tsc` will happily parse the code, and node won't even throw any errors. But it'll give a strange output: `undefined`.

The reason is that `word - 1` will result in `NaN` when `word` is a string.

So we can perform one more fix:

```typescript
function getWord(text: string, word: number) {
  // ...
}
```

Now we've told TypeScript that `word` needs to be a number. So running `tsc` on this will give us:

```
$ tsc --target ES2015 --noEmitOnError index.ts
index.ts:15:40 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.

15 console.log(getWord('This is a test.', 'foo'));
                                          ~~~~~


Found 1 error.

```

Now we can be certain we will always get `text` as a string, and `word` as a number.

There's one more thing we can do if we wanted to (though it's not required).

We can specify the return type of the function like this:

```typescript
function getWord(text: string, word: number): string {
  // ...
}
```

This will ensure our function always returns a string (or else throws an exception if it's unable to).

The neat thing is that if you take a look at `index.js` again, you'll notice it looks no different than before we added type declarations. Of course, JavaScript doesn't know anything about such type declarations, so they are all stripped out. TypeScript ensures that your code is properly typed **before** it ever runs as JavaScript. But once it's turned to JavaScript, it's technically just as loosely typed as before.

## Strict Mode

It would have been nice, however, to know about the potential for such bugs even before our function was misused.

Let's go back to square one and see if TypeScript can help us:

```typescript
function getWord(text, word) {
  if (word < 1) throw new Error('word must be no less than 1');

  const wordIndex = word - 1;

  // https://regex101.com/r/1zDG8a/2
  const words = text.toLowerCase().match(/\w+(?:'\w+)*/g);
  if (wordIndex >= words.length) {
    throw new Error(`Text only has ${words.length} words`);
  }

  return words[wordIndex];
}

console.log(getWord('This is a test.', 2));

```

Now just add the `--strict` parameter to `tsc`:

```
$ tsc --target ES2015 --noEmitOnError --strict index.ts
index.ts:1:18 - error TS7006: Parameter 'text' implicitly has an 'any' type.

1 function getWord(text, word) {
                   ~~~~

index.ts:1:24 - error TS7006: Parameter 'word' implicitly has an 'any' type.

1 function getWord(text, word) {
                         ~~~~


Found 2 errors.

```

TypeScript is saying that because we didn't specify the type for either parameter of our function, it implicitly has the `any` type. `any` just means that anything can be passed to that parameter without it complaining. Generally, we want to avoid using `any` either implicitly or explicitly, since TypeScript won't validate anything that uses `any`.

So, it can foresee the potential issues we could have and lets us know how to solve it. Now if we return our function signature to:

```typescript
function getWord(text: string, word: number) {
  // ...
}
```

Actually, now we still get an error, just not the same one. I'm making these changes live, so a potential error slipped through that I didn't even catch:

```
$ tsc --target ES2015 --noEmitOnError --strict index.ts
index.ts:8:20 - error TS2531: Object is possibly 'null'.

8   if (wordIndex >= words.length) {
                     ~~~~~

index.ts:9:38 - error TS2531: Object is possibly 'null'.

9     throw new Error(`Text only has ${words.length} words`);
                                       ~~~~~

index.ts:12:10 - error TS2531: Object is possibly 'null'.

12   return words[wordIndex];
            ~~~~~


Found 3 errors.

```

This looks a bit intimidating at first glance, but it's really just all the same error:

`Object is possibly 'null'.`

What's this mean?

The `.match()` method doesn't always return an array. There are some instances when it could return null. And if it does, we'd be in trouble.

When can `.match()` return null? When there are no matches.

Let's put that to the test:

```typescript
console.log(getWord('', 2));
```

If you run this, without the `--strict` parameter, here's what we get:

```
$ node .
index.js:7
    if (wordIndex >= words.length) {
                           ^

TypeError: Cannot read property 'length' of null
```

Well look at that, TypeScript was totally right. If someone passes an empty string, `words` will be null, not an empty array like I would have assumed.

This is why I always like having strict mode enabled.

So let's modify our code and see what happens:

```typescript
function getWord(text: string, word: number) {
  if (word < 1) throw new Error('word must be no less than 1');

  const wordIndex = word - 1;

  // https://regex101.com/r/1zDG8a/2
  const words = text.toLowerCase().match(/\w+(?:'\w+)*/g);
  if (words === null) {
    throw new Error('Please provide some text');
  }

  if (wordIndex >= words.length) {
    throw new Error(`Text only has ${words.length} words`);
  }

  return words[wordIndex];
}

console.log(getWord('', 2));
```

Now if we run `tsc`:

`tsc --target ES2015 --noEmitOnError --strict index.ts`

And yay! No errors!

Of course because an empty string is passed, node will throw an error with our custom error message. But the point is, no one can use our function in a way we did not intend.

## So Many Options

You may have noticed we keep adding more and more options to `tsc`, which can get quite annoying.

It's simple to fix that. You can add all these options to a file called `tsconfig.json`, like this:

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "noEmitOnError": true,
    "strict": true,
  },
  "files": ["./index.ts"]
}
```

We just took all the options we were already passing to `tsc`, and added them to a `compilerOptions` key in `tsconfig.json`. We also included our `index.ts` in a `files` array, so that now all we have to do is run:

`tsc`

And our file will be compiled automatically with our desired settings.

The other nice thing about creating this file is that if you use an IDE that supports TypeScript, such as WebStorm/PHPStorm or VS Code, it'll detect your settings when checking your code for errors.

## Inferred Types

You may notice that very few of our variables actually have explicit types defined, and TypeScript didn't complain, even in strict mode.

That's because in most cases, it's able to detect what types our variables are.

For instance, it knows that `.match()` returns (for all intents and purposes) a string of arrays (technically, it returns a type of the `RegExpMatchArray` interface, but that's beyond the scope of this post).

It knows that if `word` is a number, then `word - 1` is also a number, so `wordIndex` must therefore be a number.

And since `words` is an array of strings, then `words[wordIndex]` must also be a string, and so it knows that our function returns a string.

It's up to you how explicit you want to be with your types. Generally I like to define the types of all parameters and returns, and anything that might not be clear from the context.

I find that doing so prevents many accidental coding errors, as I'll show in my next post.

## Up Next

For the code that was used in this post, see [the gist here](https://gist.github.com/devbanana/812e2a06e77ba15d19eaf2344ebe7861)

That's it for this post, as it's already gotten to be long enough.

As mentioned, this will be a three-part post, and in my next post I will discuss some more advanced features of TypeScript.

Would love to read your comments on the above! As mentioned, i'm quite new to TypeScript, so this is just what I've picked up after some basic fiddling around with it. But, hopefully it is useful anyway.
