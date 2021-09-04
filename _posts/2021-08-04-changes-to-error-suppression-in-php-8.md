---
layout: post
title:  Changes to Error Suppression in PHP 8
date:   2021-08-04 10:32:00 -0400
categories: [php8, error-handling]
---

In my project at work I've been working on upgrading the code to a minimum of PHP 7.2, but also to work with PHP 8.

In running my unit tests under PHP 8, an interesting error presented itself which took a bit of time to resolve.

Essentially, we have a custom error handler registered. But, we need to detect if the error was suppressed with `@`, so as to prevent certain actions from occurring in the handler.

Here's how we used to do it:

```php
<?php

set_error_handler(function (int $errno, string $errstr): bool {
    if (error_reporting() === 0) {
        // Do nothing
        return false;
    }

    echo "Received error with message: $errstr\n";
    return true;
});

@trigger_error('Oops', E_USER_WARNING);
```

Since the error is suppressed, there should be no output.

And that's how it worked, before PHP 8.

After PHP 8, the above code still falls through to the echo statement, because [`error_reporting()`][error reporting] no longer returns 0 when error suppression is turned on.

I read through the [PHP 8 migration page][PHP 8 migration] and didn't find any indication of this, except perhaps of this small, indirect mention way down in the "Type system and error handling improvements" section:

> The @ operator no longer silences fatal errors.

Finally I found this reference on the page for [error control operators]:

> Prior to PHP 8.0.0, the value of the severity passed to the custom error handler was always `0` if the diagnostic was suppressed. This is no longer the case as of PHP 8.0.0.

So what you have to do now is to use the `&` bitwise operator to compare the return of `error_reporting()` against the `$errno` passed to the error handler.

Let me demonstrate first, then I'll explain:

```php
<?php

set_error_handler(function (int $errno, string $errstr): bool {
    if (!(error_reporting() & $errno)) {
        // Do nothing
        return false;
    }

    echo "Received error with message: $errstr\n";
    return true;
});

@trigger_error('Oops', E_USER_WARNING);
```

So what on earth is going on? If you've dealt with binary flags a lot then you already know, but some developers won't know exactly why this works.

Basically, the various error reporting flags are enabled bits (1 or 0) as part of a 2-byte bit mask ([see error constants for more][error constants]). For example, just looking at the first 4 bits, `E_ERROR` is 0001, `E_WARNING` is 0010, `E_PARSE` is 0100, and `E_NOTICE` is 1000.

We're going to pretend an `E_WARNING` was triggered here, because `E_USER_WARNING` is 0x0200 and showing that binary representation would get old. But the same idea applies.

Basically before PHP 8, no bit flags were enabled if error suppression was on (0000). So you could just check for a 0 integer and know that error suppression was enabled.

But now as the note said above, error suppression no longer suppresses fatal errors.

So what does PHP consider a fatal error? As it happens, the following are no longer suppressed:

* `E_ERROR`
* `E_PARSE`
* `E_CORE_ERROR`
* `E_COMPILE_ERROR`
* `E_USER_ERROR`
* `E_RECOVERABLE_ERROR`

This is probably for the best, but we can no longer check that `error_reporting()` returns 0 to see if error suppression is on, since now it returns a bit mask of all of the above fatal errors (which turns out to 0x1155, by the way).

So now we have to do a bitwise operation to check if our particular error is suppressed. If it is, the bit that represents our error type will be 0. If it is not, then it will be 1.

Lucky for us, the error type is passed to our error handler as `$errno`.

How do you check if a flag is enabled? You use the `&` bitwise operator. `&` returns all the bits that are present in both operands. for example, `$a & $b` returns all the bits in both `$a` and `$b`.

So if you have the bitmask 0101 and want to check if `E_ERROR` (0001) is enabled, you can do:

```php
0b0101 & E_ERROR;
```

Since the final bit was enabled in both operands, that bit will also be enabled in the output. Therefore, it will return binary `0001`.

And of course, if the two operands have no bits in common, then it will return `0`.

```php
0b0101 & E_WARNING;
```

Since `E_WARNING` is 0010, and the enabled bit is not present in 0101, this returns 0. In other words, the two operands have no bits in common.

Our original code triggers `E_USER_WARNING` (though we're pretending it's `E_WARNING` for our purposes here to make things simpler).

So if we want to check if `E_WARNING` (0010) is enabled, then we use bitwise and to compare that with the output of `error_reporting()`.

If the output is 0, we know our error is suppressed and can act accordingly. Hence we have:
```php
    // ...
    if (!(error_reporting() & $errno)) {
        // Do nothing
        return false;
    }
    // ...
```

And the great thing is, this is backwards compatible. since before PHP 8, `error_reporting()` returns `0` when error suppression is enabled, this will work exactly the same.

Anything you `and` with `0` will never have any bits in common, so the result will always be `0`.

I know this is a bit of a lengthy explanation, but hopefully I've demonstrated the new behavior of `error_reporting()` in PHP 8 and how you can use it to check if error suppression is enabled.

[error reporting]: https://www.php.net/manual/en/function.error-reporting.php
[PHP 8 migration]: https://www.php.net/releases/8.0/en.php
[error control operators]: https://www.php.net/manual/en/language.operators.errorcontrol.php
[error constants]: https://www.php.net/manual/en/errorfunc.constants.php
