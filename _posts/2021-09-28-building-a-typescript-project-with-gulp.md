---
layout: post
title: Building a TypeScript Project with Gulp
categories: static-typing
tags: gulp typescript javascript
---

In my [last post](https://www.devbanana.me/static-typing/2021/09/23/typescript-for-statically-typed-javascript.html), I gave a brief introduction to TypeScript and why you might think about using it.

This will be part 2 of that series, where I will demonstrate how to set up a new TypeScript project and enable it to be built using gulp.

And in the next post, we'll then build upon this foundation to create a simple TypeScript project to demonstrate some more of the language features.

I've created this project as a repository on GitHub, so you can easily follow along with the process. I'll link to each commit that performs the task I'm outlining.

Repository: [devbanana/typescript-example](https://github.com/devbanana/typescript-example)

## Required Tools

As in my previous post, you'll need both node and npm. If you don't already have them installed, you can [learn how to install them here](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm). I'll assume from here that these tools are successfully installed.

You'll also need gulp installed globally, which you can get with:

```bash
npm install -g gulp-cli
```

## Creating the Package

Commit: [eb0b22d](https://github.com/devbanana/typescript-example/commit/eb0b22d815b4fd96325eff8c053fbcc1fe31ed91)

I'll be using PhpStorm, which is why you will see the [.idea](https://github.com/devbanana/typescript-example/tree/master/.idea) directory. Of course the example will work just fine without that, but it's there in case you are using the same editor.

First we need to create the node package. In an empty directory, run:

```bash
npm init
```

You'll be prompted for the package name, description, author, etc. Most of these don't matter, so you can set whatever value you like. 

The only one that matters is the **entry point**. It'll default to `index.js`, but we'll set it to `build/index.js`, since we'll have gulp build our TypeScript to this directory.

You can see our full [`package.json`](https://github.com/devbanana/typescript-example/blob/eb0b22d815b4fd96325eff8c053fbcc1fe31ed91/package.json) here.

Setting the entry point should set a key like this in your `package.json`:

```json
{
  "main": "build/index.js"
}
```

## Installing TypeScript

Commit: [9590d63](https://github.com/devbanana/typescript-example/commit/9590d63c9846953f3c458b3c1e0d03876a711d1b)

Next we need to install some basic packages.

Run:

```bash
npm install --save-dev typescript @types/node
```

Even if you have [TypeScript](https://www.npmjs.com/package/typescript) installed globally, it's a good idea to have it installed as a local dev dependency. It'll be required later on when we want to compile TypeScript using gulp.

What's this `@types/node` all about?

TypeScript has type information about all the built-in JavaScript functions.

But when you use other libraries, including node, it offers additional functions that TypeScript doesn't know about, since such libraries are written in plain JavaScript. so, it can't guess what types a function might accept or return.

Therefore, you can download declaration files which specify type information for a library's functions.

Some libraries are nice enough to package these declaration files in the library itself. But if they don't, there is often a separate library prefixed with `@types/` which includes these declaration files.

`@types/node` is one such library, which offers declaration files for Node.js.

With these two libraries installed, we will now have a `node_modules` directory, which stores these dependencies. We'll add that to `.gitignore` so it's not committed to git:

```
/node_modules/

```

## Create Example TypeScript File

Commit: [e652fe6](https://github.com/devbanana/typescript-example/commit/e652fe673eaa86f69723f227280ba4b37e71233b)

In the next post, we will create a sample project that demonstrates some of the more advanced features of TypeScript. However, just to verify that our environment is working as it should, let's create a simple [`greeting.ts`](https://github.com/devbanana/typescript-example/blob/e652fe673eaa86f69723f227280ba4b37e71233b/src/greeting.ts) file in the `src` directory:

```typescript
console.log('Hello!');
```

You can verify it compiles with:

```bash
npx tsc src/greeting.ts
```

`npx` just allows us to run `tsc` without having to install the library globally.

It should successfully compile the TypeScript file into `src/greeting.js`. But go ahead and delete it since we'll be compiling to a different directory later on.

## Create `tsconfig.json`

Commit: [cf24d86](https://github.com/devbanana/typescript-example/commit/cf24d8629ee27ae58c26379ddd898e172b2bc73b)

Remember from the previous post that we can create a `tsconfig.json` file to store our compiler options for TypeScript.

`gulp-typescript` (which I'll introduce soon) allows you to pass compiler options within gulp itself, but we may as well store them in the right place.

So, let's create a [`tsconfig.json`](https://github.com/devbanana/typescript-example/blob/cf24d8629ee27ae58c26379ddd898e172b2bc73b/tsconfig.json) as follows:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node",
    "target": "ES2019",
    "lib": ["ES2019"],
    "strict": true,
    "noEmitOnError": true,
    "declaration": true
  },
  "include": ["./src"]
}
```

As before, we have a list of compiler options. But some of them might look unfamiliar, so let's go through them one-by-one:

* `"module": "commonjs"` - This tells TypeScript that we are targeting the CommonJS module format (as opposed to ES Modules). While Node.js now has support for the new ES Module format, it's not as well-supported as CommonJS. TypeScript will automatically convert our code into the CommonJS format for us. If you're not sure what the difference is between these two module systems, see these great guides on [CommonJS](https://flaviocopes.com/commonjs/) and [ES Modules](https://flaviocopes.com/es-modules/).
* `"moduleResolution": "node"` - This tells TypeScript how to resolve module references. Setting `moduleResolution` to `node` mimics node's module resolution mechanism. In particular, it'll look for imported modules inside the `node_modules` directory.
* `"target": "ES2019"` - This tells TypeScript that it's OK to generate JavaScript that conforms to the ES2019 standard. If you are using at least node version 12, then this should be fine.
* `"lib": ["ES2019"]` - This tells TypeScript to load type information for the specified libraries.
* `"strict": true` - You should already be familiar with this from the last post. This tells TypeScript to enable all strict checks.
* `"noEmitOnError": true` - This should also be familiar to you. This tells TypeScript to not compile the code to JavaScript if there were any errors.
* `"declaration": true` - This generates declaration files (`.d.ts`) for each emitted JavaScript file. This ensures that the type information from our project isn't lost after compilation.

Finally, there's this line:

```json
"include": ["./src"]
```

This just tells TypeScript that all of our files are inside of `src`.

## Install Gulp Libraries

Commit: [28120b7](https://github.com/devbanana/typescript-example/commit/28120b7b4022bf349aed6fb6bd0dce6f05b6dca3)

Next we'll run:

```bash
npm install --save-dev ts-node gulp @types/gulp gulp-typescript
```

One more set of libraries to install before we can get started with our gulp tasks.

First is [`ts-node`](https://npmjs.org/package/ts-node). This enables us to execute TypeScript without actually compiling it to plain JavaScript.

Sure we could run `tsc` every time we wanted to run `gulp`, but that'd get quite annoying. It's easier just to run `gulp` using `gulpfile.ts`, and gulp does allow this as long as we have `ts-node` available.

You can see how it works with our existing TypeScript file by running:

```bash
npx ts-node src/greeting.ts
```

You should see:

```
Hello!
```

Not very exciting, but at least it works. :stuck_out_tongue:

If our `gulpfile` is named `gulpfile.ts` and `ts-node` is installed, `gulp` will automatically use it to run our tasks for us.

Next we install [`gulp`](https://www.npmjs.com/package/gulp), plus the type definitions in `@types/gulp`.

And finally we install [`gulp-typescript`](https://www.npmjs.com/package/gulp-typescript), which is a gulp library that allows us to compile TypeScript from within a task.

## Create `gulpfile.ts`

Commit: [6e2dea5](https://github.com/devbanana/typescript-example/commit/6e2dea55c4ef64ad9388c5f2e242f183726a980c)

Finally we get to creating the actual `gulpfile`.

Here is our preliminary [`gulpfile.ts`](https://github.com/devbanana/typescript-example/blob/6e2dea55c4ef64ad9388c5f2e242f183726a980c/gulpfile.ts):

```typescript
// noinspection JSUnusedGlobalSymbols

import { dest } from 'gulp';
import * as ts from 'gulp-typescript';

const project = ts.createProject('tsconfig.json');

export function compile(): NodeJS.ReadWriteStream {
  return project.src().pipe(project()).pipe(dest('build'));
}

```

It's a pretty short file but let's see what's going on.

The comment on line 1 just tells PhpStorm to ignore unused exports. Any export defined in this file will be picked up by `gulp` as a task, so it's not really "unused".

Next we have two import statements:

```typescript
import { dest } from 'gulp';
import * as ts from 'gulp-typescript';
```

If you've created gulp tasks before, you might be used to a different syntax — something like this:

```javascript
const gulp = require('gulp');
```

That is CommonJS syntax, and indeed our typescript will be turned into something like that when it's compiled.

But TypeScript allows us to use ES Modules syntax. I prefer to use `import` when possible because TypeScript will validate that we're loading something that actually exists.

With `require`, TypeScript won't complain if the module doesn't exist. You could accidentally type something like:

```typescript
const gulp = require('gilp');
```

And TypeScript wouldn't complain at all.

The first import, `import { dest } from 'gulp';`, is just loading the `dest` function from `gulp`. We don't need the whole gulp library loaded, just the `dest` function, since that's all we're using right now.

The second import, `import * as ts from 'gulp-typescript';`, loads all exports from `gulp-typescript` and puts them in an object pointed to by `ts`.

Then we create the TS project:

```typescript
const project = ts.createProject('tsconfig.json');
```

As I said earlier, `gulp-typescript` allows us to pass compiler options directly, but it also supports reading them from a `tsconfig.json` file. This seems cleaner to me so I prefer to do it this way.

Then we have our compile task:

```typescript
export function compile(): NodeJS.ReadWriteStream {
  return project.src().pipe(project()).pipe(dest('build'));
}
```

Again you may be used to the CommonJS style of exports:

```javascript
exports.compile = function () {
  // ...
}
```

But in ES Modules, if we just stick the `export` keyword in front of any function, class, constant, etc., it'll mark it as an export of our module.

Instead of using `gulp.src()` as we normally would, we use the `.src()` method on `project`. Since we are having `gulp-typescript` read our configuration from `tsconfig.json`, it also knows which files we want to build, so there's no need to define them again.

`.pipe(project())` is where the files are actually compiled. `project` points to a function that does the compilation.

And finally we have the usual call to `.dest()`, which writes our files to `build/`.

The only thing you might not be sure about is the `NodeJS.ReadWriteStream`. That is the return type of our function. As I mentioned in the previous post, return types are generally optional, but I'm using [`typescript-eslint`](https://github.com/typescript-eslint/typescript-eslint) to lint my TypeScript, and it has a default rule that [requires return types on all exported functions](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/eslint-plugin/docs/rules/explicit-module-boundary-types.md).

PhpStorm has an option to specify the return type automatically, but you can also find it yourself. However it does take a bit of detective work when you're returning the return type of another library's function. The good news is that TypeScript will certainly complain if you get it wrong.

We'll discuss return types a bit more in the next post when we build out the project more, but for now just trust me it is `NodeJS.ReadWriteStream`.

To see it for yourself, see the return type of [`dest()`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/29216607f433cbb900acdcf4a0e4ea69e9db5376/types/vinyl-fs/index.d.ts#L165) from vinyl-fs:

```typescript
export function dest(folder: string, opt?: DestOptions): NodeJS.ReadWriteStream;
```

So now we have a very basic gulp task. To run it, just run:

```bash
gulp compile
```

You'll see something like this:

```bash
[21:21:50] Requiring external module ts-node/register
[21:21:52] Using gulpfile ~/code/typescript-example/gulpfile.ts
[21:21:52] Starting 'compile'...
[21:21:53] Finished 'compile' after 1.84 s
```

Now you should be able to look in the `build/` directory and see `greeting.js` and `greeting.d.ts`.

Let's run our script:

```bash
$ node build/greeting.js 
Hello!
```

Certainly not groundbreaking, but it shows that our gulp task successfully built our TypeScript.

I know we set our package's entry point to `build/index.js` earlier. We'll use that in the project we build out in the next post. For now, I just wanted an example script to show that it works.

## Compiling Automatically

Commit: [b95a793](https://github.com/devbanana/typescript-example/commit/b95a793597b4ea33b6f231ee5ffc1f3c7451067d)

What if we could detect when a TypeScript file changed, and so automatically recompile them as needed? That'd be cool.

Well if you're familiar with gulp, you certainly know about gulp's `watch` API.

So let's build another task:

```typescript
export function monitor(): NodeJS.EventEmitter {
  return watch('src/**/*.ts', compile);
}
```

First, we're using a new function from `gulp`, so we have to add it to our import statement:

```typescript
import { dest, watch } from 'gulp';
```

Otherwise, this is pretty straightforward. It takes a glob pattern as the first argument, and a callback as the second. For the callback, we're calling our existing `compile` function.

Now run:

```bash
gulp monitor
```

And you'll see something like this:

```bash
[22:08:02] Requiring external module ts-node/register
[22:08:04] Using gulpfile ~/code/typescript-example/gulpfile.ts
[22:08:04] Starting 'monitor'...
```

Now the script will remain open, since it's actively monitoring the `src` directory for any changes.

To see how it works, let's make a change to `src/greeting.ts`:

```typescript
console.log('Bonjour!');
```

If you save that file, then quickly jump over to the open terminal window, you'll see:

```bash
[22:09:53] Starting 'compile'...
[22:09:55] Finished 'compile' after 2.17 s
```

In a separate window (since the script is still watching), if you go and run:

```bash
node build/greeting.js
```

You'll now see:

```
Bonjour!
```

Now you don't have to run anything at all to compile TypeScript!

Once you no longer want to monitor the `src` directory for changes, just press `⌃C` and it'll exit.

## Clean the Build

Commit: [b79a0a2](https://github.com/devbanana/typescript-example/commit/b79a0a2e188e133199d12b5b0b79304ea5afd0cb)

Next we want a way to clean our build files.

Let's install [`del`](https://www.npmjs.com/package/del):

```bash
npm install --save-dev del
```

`del` is one of those libraries that nicely includes declaration files, so we don't need to install anything else.

Import it with:

```typescript
import * as del from 'del';
```

And now the task:

```typescript
export function clean(): Promise<void> {
  return del(['build/**/*.js', 'build/**/*.d.ts']).then(paths => {
    paths.forEach(path => console.log(`Deleted ${path}`));
  });
}
```

`del()` accepts an array of globs. We set it to delete all `.js` and `.d.ts` files within the `build/` directory.

`del()` returns a promise that resolves to an array of paths that have been deleted. We use that to output each path, with the word `Deleted` before each one.

If you're unfamiliar with promises, I'll be discussing them a bit more in the next post — and if there's interest, perhaps I'll do a post or two dedicated to promises in the future.

Now if you run:

```bash
gulp clean
```

You'll see something like this:

```bash
[06:45:27] Requiring external module ts-node/register
[06:45:29] Using gulpfile ~/code/typescript-example/gulpfile.ts
[06:45:29] Starting 'clean'...
Deleted /Users/brandonolivares/code/typescript-example/build/greeting.d.ts
Deleted /Users/brandonolivares/code/typescript-example/build/greeting.js
[06:45:29] Finished 'clean' after 32 ms
```

## Building the Package

Commit: [dbe29c7](https://github.com/devbanana/typescript-example/commit/dbe29c759e080c28811b343f7a2fd9788b7c7b64)

Let's wrap up by creating one more task to create a `.zip` file of our project.

For this we'll need [`gulp-zip`](https://www.npmjs.com/package/gulp-zip), which you can get like this:

```bash
npm install --save-dev gulp-zip @types/gulp-zip
```

Next, we have to import it:

```typescript
import * as zip from 'gulp-zip';
```

We're also going to need a few other functions from gulp, specifically `series()` and `src()`. So let's modify our `gulp` import line:

```typescript
import { series, src, dest, watch } from 'gulp';
```

Next, let's create a function to create the zip file:

```typescript
function createZip(): NodeJS.ReadWriteStream {
  return src(['package*.json', 'build/**/*.js', 'build/**/*.d.ts'], {
    base: '.',
  })
    .pipe(zip('typescript-example.zip'))
    .pipe(dest('dist'));
}
```

This should be pretty familiar by now, as it's not too different from what we've already done.

The first thing to notice is that we didn't include `export` before the function. That's because I don't necessarily want this to be its own task: I'm just creating the function to be used in a later task, which you'll see below.

If you tried calling:

```bash
gulp createZip
```

You'd get this:

```bash
[19:12:58] Requiring external module ts-node/register
[19:13:00] Using gulpfile ~/code/typescript-example/gulpfile.ts
[19:13:00] Task never defined: createZip
[19:13:00] To list available tasks, try running: gulp --tasks
```

Of course, if you want it to be its own task, just put `export` in front of the function and it'll work.

In `src()`, we include all our JavaScript files, plus our `package.json` and `package-lock.json`. Right now we only have dev dependencies, but once we flesh out the project in the next post, we'll have a dependency that will need to be included at runtime, so it's important to include these files.

I'm not including `gulpfile.ts`, or anything in `src/`, because these are our dev files and not needed for a production package.

You'll notice I passed the `base` option to `src()`. This will make sure all included files are relative to this base directory. In other words, it maintains our directory structure. Without this option, our zip file will only contain our actual files all in one directory.

Next we specify the `zip()` function, passing it the name of the zip file we want to create.

And finally, we call `dest()`, specifying that we want our zip file written to the `dist/` directory.

Now for the actual task. I'd like to be able to type `gulp` and have it do everything: compile our TypeScript files **and** create the zip. Doing that is quite simple:

```typescript
export default series(compile, createZip);
```

This creates a default export, which calls `series()`. `series()` will call the included tasks in order, not starting the next task until the previous one has completed.

`series()` returns a function reference, so we don't need to include it in a function itself.

And now when we run:

```bash
gulp
```

We'll see something like this:

```bash
[19:26:33] Requiring external module ts-node/register
[19:26:35] Using gulpfile ~/code/typescript-example/gulpfile.ts
[19:26:35] Starting 'default'...
[19:26:35] Starting 'compile'...
[19:26:37] Finished 'compile' after 1.75 s
[19:26:37] Starting 'createZip'...
[19:26:37] Finished 'createZip' after 30 ms
[19:26:37] Finished 'default' after 1.78 s
```

I'd like to make one final change. When we run `gulp clean`, it'd be nice to also delete the compiled package. So let's change the `clean` task to this:

```typescript
export function clean(): Promise<void> {
  return del(['build/**/*.js', 'build/**/*.d.ts', 'dist/*.zip']).then(paths => {
    paths.forEach(path => console.log(`Deleted ${path}`));
  });
}
```

Now when we run:

```bash
gulp clean
```

We'll see this:

```bash
[19:57:03] Requiring external module ts-node/register
[19:57:05] Using gulpfile ~/code/typescript-example/gulpfile.ts
[19:57:05] Starting 'clean'...
Deleted /Users/brandonolivares/code/typescript-example/build/greeting.d.ts
Deleted /Users/brandonolivares/code/typescript-example/build/greeting.js
Deleted /Users/brandonolivares/code/typescript-example/dist/typescript-example.zip
[19:57:05] Finished 'clean' after 24 ms
```

Just so it's all in one place, here is our complete [`gulpfile.ts`](https://github.com/devbanana/typescript-example/blob/dbe29c759e080c28811b343f7a2fd9788b7c7b64/gulpfile.ts):

```typescript
// noinspection JSUnusedGlobalSymbols

import { dest, series, src, watch } from 'gulp';
import * as ts from 'gulp-typescript';
import * as del from 'del';
import * as zip from 'gulp-zip';

const project = ts.createProject('tsconfig.json');

export function compile(): NodeJS.ReadWriteStream {
  return project.src().pipe(project()).pipe(dest('build'));
}

export function monitor(): NodeJS.EventEmitter {
  return watch('src/**/*.ts', compile);
}

export function clean(): Promise<void> {
  return del(['build/**/*.js', 'build/**/*.d.ts', 'dist/*.zip']).then(paths => {
    paths.forEach(path => console.log(`Deleted ${path}`));
  });
}

function createZip(): NodeJS.ReadWriteStream {
  return src(['package*.json', 'build/**/*.js', 'build/**/*.d.ts'], {
    base: '.',
  })
    .pipe(zip('typescript-example.zip'))
    .pipe(dest('dist'));
}

export default series(compile, createZip);
```

## Up Next

You can see [the full project as it is so far](https://github.com/devbanana/typescript-example/tree/dbe29c759e080c28811b343f7a2fd9788b7c7b64).

So far we've created an extremely simple TypeScript project that just prints out a greeting. However, we created some gulp tasks to compile that project, build a zip from the compiled files, and clean the build as necessary.

Our `gulpfile` was written entirely in TypeScript without needing to be compiled into JavaScript first, thanks to the use of `ts-node`.

In the next post, we'll flesh out the project to demonstrate some other features of TypeScript.

Perhaps building these gulp tasks wasn't the most exciting thing ever, but hopefully you got a good idea of how it works, and now we have a solid foundation to build upon for next time.
