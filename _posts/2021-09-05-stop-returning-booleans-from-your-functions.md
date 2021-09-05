---
layout: post
title: Stop Returning Booleans From Your Functions
date: 2021-09-05 01:58:00 -0400
author: Brandon Olivares
categories: [best-practices]
---

[As I said recently][error suppression], I've been working on upgrading a project at work.

As such, there's plenty of code that hasn't been updated for several years, and definitely does not follow best practices.

One thing I've noticed that has become quite a pet peeve is returning booleans from functions.

In my opinion, there is **almost** never a reason to return a boolean. And a boolean should never be returned as an indication of success or failure.

Here is a valid case for returning a boolean:

```php
<?php

function isEven(int $number): bool
{
    return $number % 2 === 0;
}

```

And another valid case:

```php
<?php

final class User
{
    // ...

    public function isBirthdayOn(\DateTimeInterface $date): bool
    {
        return $this->birthday->format('m-d') === $date->format('m-d');
    }
}

```

Here is **not** a valid case:

```php
<?php

use Symfony\Component\Mailer\Exception\TransportExceptionInterface;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

final class UserNotifier
{
    public function __construct(private MailerInterface $mailer)
    {
    }

    public function notify(string $email): bool
    {
        $email = filter_var($email, FILTER_VALIDATE_EMAIL);
        if ($email === false) {
            return false;
        }

        $message = (new Email())
            ->from('foo@example.com')
            ->to($email)
            ->subject('Test')
            ->text('You have registered.')
        ;

        try {
            $this->mailer->send($message);
        } catch (TransportExceptionInterface) {
            return false;
        }

        return true;
    }
}

```

So what's wrong with this `notify()` method?

There's no reason it should return `true` or `false`. As I said above, `true` or `false` should never be returned as indicators of success or failure.
I'll explain more about that later.

## Inconsistencies in PHP

Unfortunately PHP doesn't do much to discourage this bad practice, because many native PHP functions return false on failure.
I've been made especially aware of this as I've been working more with [PHPStan], which of course is rather picky about return values.

For example, take a look at this code:

```php
<?php

$json = file_get_contents('somefile.json');
$data = json_decode($json);
```

That seems pretty straightforward, but if you run that through phpstan, it'll report this error:

      4      Parameter #1 $json of function json_decode expects string,
             string|false given.

[Check it out for yourself here.](https://phpstan.org/r/7d880332-5b17-41d0-960d-6c7e024864f9)

Why is there an error?

Because `file_get_contents()` returns false on failure.
That means if we want to prevent accidentally passing false to `json_decode()`, we have to check our return value like this:

```php
<?php

$json = file_get_contents('somefile.json');
if ($json !== false) {
    $data = json_decode($json);
}

```

Now there are no errors. [Check it out on phpstan.](https://phpstan.org/r/b098a1ea-3b79-45dd-bdc5-3221055cbf60)

But the really messed up part is the value of `$data` after calling `json_decode()`.
`json_decode()` returns `mixed`, which means it could return literally anything.
But most confusingly of all, if it returns `null`, that could mean the passed string contained `null`, but it could also mean it couldn't parse the JSON.

```php
<?php
var_dump(json_decode('null'));
var_dump(json_decode('[1, 2, 3'));
```

[Both of these return null.](https://3v4l.org/ujAfS)

So long story short, PHP doesn't have a great track-record here, and so it's easy to follow in its footsteps.

## Know What You Want

I'm a proponent of design by contract.
A function or method has a contract with its client.
It expects its inputs to follow certain specifications (preconditions), and if those are valid, promises some particular output or result (postconditions).

As the client of a function, I ask myself, "What do I need from this function?"
Then as the developer of that function, I need to ensure it fulfills that requirement.

The `isEven()` function I showed first has a very simple contract. Given an integer (precondition), it will tell me whether that number is even (postcondition).

The `User::isBirthdayOn()` method also has a simple contract.
Given a user with a birthdate (precondition), and provided a date to compare (precondition),
it will return whether the month and day of those two dates are the same (postcondition).

But what's the contract of the `UserNotifier::notify()` method?

You could say, given a valid email address (precondition), an email is sent to the user (postcondition).

But notice we're not looking for any information from the method.
We're just looking for it to perform some action. If that action is performed, the postcondition has been met.

As such, returning `true` on success doesn't really make sense.
If the email is sent, the contract is fulfilled and we can exit.

## Fail Fast

But even worse, if a precondition is not met (like an invalid email is given),
or the postcondition fails to be met (like the email can't be sent),
returning `false` seems like a really bad way of making the client aware of that.

What if the client doesn't check for a `false` return value?
Then it'd happily continue on without ever knowing there was an issue.

Imagine something like this:

```php
$notifier = new UserNotifier(/* ... */);
$notifier->notify(''); // Oops, this is invalid

// Do other stuff without ever checking the result
```

If that call to `notify()` fails, which it will in this case, no one is any the wiser.

That's not desirable of course. If a pre or post condition fails, we want all alarms blaring. We don't want the client to be able to ignore a failed contract.
Therefore, returning `false` isn't strong enough.

Instead, we should be throwing an exception. Martin Fowler calls this [failing fast].

Let's revise the code like this:

```php
<?php

use Symfony\Component\Mailer\Exception\TransportExceptionInterface;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

final class UserNotifier
{
    public function __construct(private MailerInterface $mailer)
    {
    }

    public function notify(string $email): void
    {
        $email = filter_var($email, FILTER_VALIDATE_EMAIL);
        if ($email === false) {
            throw new \InvalidArgumentException('Email is invalid');
        }

        $message = (new Email())
            ->from('foo@example.com')
            ->to($email)
            ->subject('Test')
            ->text('You have registered.')
        ;

        try {
            $this->mailer->send($message);
        } catch (TransportExceptionInterface $exception) {
            throw CouldNotSendEmail::withError($exception);
        }
    }
}

```

This is a lot better.
If our precondition fails, we know immediately, because an `InvalidArgumentException` is thrown.
It literally cannot be ignored.
And if our postcondition fails, we also know right away, because a different exception is thrown.

But even better is that if everything goes well, we don't need to know about it.
It silently exits, having successfully fulfilled its contract.

Now our calling code can look like this:

```php
$notifier = new UserNotifier(/* ... */);
$notifier->notify(''); // Oops, this is invalid

// This will never be reached because an exception will be thrown.
```

When that call to `notify()` fails, code execution halts right there.
Either the exception is handled directly in a try/catch block, or it bubbles up and is handled elsewhere.
But, the client can never be ignorant of an error.

Yet, if the call is correct and the email is successfully sent, there's no point of the client checking the result of the call.
It can just continue on with whatever else it has to do.

This results in cleaner code.
We don't have to do things like this all the time:

```php
<?php

$user = User::create(/* ... */);
if ($user) {
    $notifier = new UserNotifier(/* ... */);
    if ($notifier->notify($user->email())) {
        // etc...
    }
}

```

These constant return value checks result in countless unneeded conditionals.
Then pretty soon we have code nested 5 levels deep and it's hard to make sense of everything.

Whereas if we just do:

```php
<?php

$user = User::create(/* ... */);
    $notifier = new UserNotifier(/* ... */);
$notifier->notify($user->email());

```

Our code is far simpler.
If code execution continues, we can be sure that everything is as it should be,
whereas if something fails, it'll fail fast, and loudly.

## How to Know If You Should Return a Boolean

In my opinion there's one case where booleans should be returned:

When you ask yourself, "What do I need from this function?" and the needed information can be formulated as a yes or no answer.

What do I need from the `isEven()` function?<br>
To know whether a number is even (yes) or not (no).

What do I need from the `User::isBirthdayOn()` method?<br>
To know whether the given date is on the user's birthday (yes) or not (no).

If you can formulate the answer as yes or no, then you can return a boolean.

Otherwise, you should return something more specialized.

Or, in the example of a function that performs some effect, like the `notify()` method that sends an email,
it should return void, because you don't need any information from it at all.

This is the core of CQS, but that's a topic for another day.

[error suppression]: /php8/error-handling/2021/08/04/changes-to-error-suppression-in-php-8.html
[ynab-tools]: https://github.com/devbanana/ynab-tools
[PHPStan]: https://github.com/phpstan/phpstan
[failing fast]: https://www.martinfowler.com/ieeeSoftware/failFast.pdf

*[CQS]: Command Query Separation
