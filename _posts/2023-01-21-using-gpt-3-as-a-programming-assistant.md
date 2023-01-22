---
layout: post
title: Using GPT-3 as a Programming Assistant
categories: artificial-intelligence
tags: symfony, console
---

Well, I'm back from a hiatus of over a year, hopefully to add some more useful content to this blog.

Lately I've been having a lot of fun with GPT-3.

I've heard a lot of claims that it will one day be able to replace programmers, so I wanted to see how far I could take it.

## Simple Usage

I find it does particularly well with simple requests. The other day I wanted to see how to programmatically get the meta description from a WordPress site that uses the Yoast plugin. Too lazy to Google it, I thought I'd see what GPT-3 could offer, so I asked:

> How can I export Yoast SEO meta descriptions from WordPress posts using PHP?

Here was its response:

> You can use the WordPress function get_post_meta() to get the SEO meta descriptions from a post. The code might look something like this:
> 
> ```php
> $seo_descr = get_post_meta(get_the_ID(), '_yoast_wpseo_metadesc', true);
> ```

Now I admit, I'm useless when it comes to WordPress programming, mostly because I hate WordPress so much as a programming environment. so, wanting to know what the `true` was about, i asked:

> What does the third argument of the WordPress `get_post_meta()` function do?

And GPT-3 answered:

> The third argument of the WordPress `get_post_meta()` function is a boolean value indicating whether to return an array or a single value. If this argument is set to `true`, an array of the post meta values will be returned; if it is set to `false`, only the first value matching the given meta key will be returned.

I'm pretty impressed by that answer. And yes, it included the markdown formatting in the response, since I had included it in my question.

Anyway, the task I really wanted was to output a list of all posts, ordered chronologically, in the format `title: meta description`. So I asked it this:

> In PHP, output a list of all blog post titles ordered chronologically. Also include the value of the `_yoast_wpseo_metadesc` meta key.
> 
> For example:
> 
> Title 1: description 1  
> Title 2: description 2  
> Title 3: description 3

Here was the answer:

> ```php
> <?php 
> 
> $args = array('post_type'=> 'post', 'posts_per_page' => -1, 'orderby'=> 'date', 'order'=> 'DESC');
> $blogPosts = get_posts( $args );
> 
> foreach( $blogPosts as $blogPost ):
>     $title = get_the_title($blogPost->ID);
>     $description = get_post_meta( $blogPost->ID, '_yoast_wpseo_metadesc', true);
>     echo $title . ': ' . $description . '<br/>';
> endforeach;
> 
> ?>
> ```

OK, pretty good overall.

I'm a bit annoyed at the `foreach`/`endforeach` format, but it seems to assume this is placed in a template, seeing as it ended each line in `<br/>`.

I also think the code style leaves a bit to be desired, and wish it would have used short array syntax, but all that is easily fixed.

It's funny to see the assumption it made though, that I wanted it ordered chronologically from most recent to oldest. To be fair, I hadn't specified that in my request.

But overall, that's a pretty good job, and I verified the code did indeed work.

## Building a Symfony Console Application with GPT-3

But the question is, how would it handle something with greater complexity?

I've been working on building a console application in Symfony that helps solve the NYTimes [Letter Boxed](https://www.nytimes.com/puzzles/letter-boxed) game.

If you're not familiar with Letter Boxed, it's a word game where you create words from letters in a square grid. Words must be at least 3 letters in length, letters can be reused, and consecutive letters must not come from the same side. The last letter of a word becomes the first letter of the next word. The aim is to use all letters to solve the puzzle in as few words as possible.

The console application would accept a list of one or more words, validate that they follow the rules of the game, then return any letters that haven't yet been used by the given words.

For example, the January 19, 2023 puzzle had these letters:

* XBM
* INA
* OYL
* TEC

If we feed the word “exclamation” into the application, it should return the letters “BY”, since those are the only letters left that haven't been used in the provided word.

Let's see how far we can get with only GPT-3.

## Create the Entry Point

Request:

> Create a console script in PHP which will be the entry point into a Symphony console application. It must include `vendor/autoload.php` first. It should create a new `Application` object and run it.

```php
<?php 
require 'vendor/autoload.php';

use Symfony\Component\Console\Application;

$application = new Application();
$application->run();
```

Pretty good, but I want this to be an executable script, and I also want to enable strict types. While we're at it, we'll set the title of the application to `Letter Boxed Solver`.

> Create a console script in PHP which will be the entry point into a Symphony console application. First enable strict types. It must include `vendor/autoload.php` first. It should create a new `Application` object and run it.
> 
> This is an executable script so it should begin with a call to the PHP executable.
> 
> The title of the application is `Letter Boxed Solver`

```php
#!/usr/bin/env php
<?php
declare(strict_types=1);

require_once __DIR__.'/vendor/autoload.php';

use Symfony\Component\Console\Application;

$app = new Application('Letter Boxed Solver');

$app->run();
```

Yep that's pretty good. And it works, too, albeit with no current functionality.

## Creating the Command

> Create a Symphony command called `Devbanana\LetterBoxed\Command\RemainingCommand`.
> 
> * It should declare strict types.
> * It should use the `AsCommand` PHP attribute to declare it is a command. The command's name is `remaining` and the description is `Determine remaining letters after word(s)`.
> * The help text is, `Determine which letters are left after the given words`.
> * It should accept an argument called `letters`, which is required and is an array. The description of the argument is: `The letters of the puzzle (each side separated by a space)`
> * It should accept an option called `word` (short form `w`) which requires a value and is an array. The description of the option is: `The word(s) in the puzzle so far`.
> * For outputting text, it should use `SymfonyStyle` assigned to `$io`.
> * With `$io`, it should output the title `Remaining`.
> * It should check that the `letters` argument is an array with 4 elements. If not, it should output the error, `There must be exactly 4 sets of letters.`. This should be written to `stderr` using the `getErrorStyle()` method of `$io`. It should return `Command::INVALID`.
> * Otherwise, it should return `Command::SUCCESS`.

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

/**
 * @AsCommand(name="remaining", description="Determine remaining letters after word(s)")
 */
class RemainingCommand extends Command
{
    protected function configure()
    {
        $this
            ->setDescription('Determine which letters are left after the given words')
            ->addArgument('letters', InputArgument::REQUIRED, 'The letters of the puzzle (each side separated by a space)')
            ->addOption('word', 'w', InputOption::VALUE_REQUIRED | InputOption::VALUE_IS_ARRAY, 'The word(s) in the puzzle so far');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        // create a style helper for outputting text
        $io = new SymfonyStyle($input, $output);

        // output the title
        $io->title('Remaining');
       
        // check that the argument is an array with 4 elements
        $letters = $input->getArgument('letters');
        if (!is_array($letters) || count($letters) !== 4) {
            $io->getErrorStyle()->error('There must be exactly 4 sets of letters.');
            return Command::INVALID;
        }

        // TODO: add logic

        return Command::SUCCESS;
    }
}
```

Funnily enough, it actually output this code in markdown format, too.

Anyway, this is pretty good overall.

Unfortunately it seems to not know about PHP attributes. I tried several times to get it to output a proper `#[AsCommand()]` attribute but it wouldn't do it.

I had to be very specific about using `SymfonyStyle::getErrorStyle()`, but it got the point in the end.

It didn't actually define my `letters` argument as an array for some reason.

And finally, it didn't declare the return types of the methods.

But again, pretty good overall — it just requires some manual oversight.

Here's my modified version after some styling using my [preferred code style](https://github.com/devbanana/php-cs-fixer-config):

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'remaining', description: 'Determine remaining letters after word(s)')]
final class RemainingCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->setHelp('Determine which letters are left after the given words')
            ->addArgument('letters', InputArgument::REQUIRED | InputArgument::IS_ARRAY, 'The letters of the puzzle (each side separated by a space)')
            ->addOption('word', 'w', InputOption::VALUE_REQUIRED | InputOption::VALUE_IS_ARRAY, 'The word(s) in the puzzle so far')
        ;
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $io->title('Remaining');

        $letters = $input->getArgument('letters');
        if (!\is_array($letters) || \count($letters) !== 4) {
            $io->getErrorStyle()->error('There must be exactly 4 sets of letters.');

            return Command::INVALID;
        }

        // TODO: add logic

        return Command::SUCCESS;
    }
}

```

I'm also realizing there's no validation that at least one word is passed. A value is required if the option is given, but we should validate a word is actually given.

> Given the following command, ensure the `word` option is an array that is not empty. If it is empty, output the error `At least 1 word is required.`
> 
> `<script excluded for brevity>`

```php
<?php

// ...

$words = $input->getOption('word');

if (!\is_array($words) || empty($words)) {
    $io->getErrorStyle()->error('At least 1 word is required.');

    return Command::INVALID;
}

// ...

```

OK that's cool: no joke, it only output the part of the script that had changed. It even included the `// ...` comments. And I no longer have to be so specific about how to output errors, because it sees how it was already done before.

And now let's add this command to the entry point:

> Given the following PHP script, add the `Devbanana\LetterBoxed\Command\RemainingCommand` command.
> 
> `<script excluded for brevity>`

```php
#!/usr/bin/env php
<?php
declare(strict_types=1);

require_once __DIR__.'/vendor/autoload.php';

use Symfony\Component\Console\Application;
use Devbanana\LetterBoxed\Command\RemainingCommand;

$app = new Application('Letter Boxed Solver');
$app->add(new RemainingCommand());
$app->run();
```

It did a great job adding the command to the script. Of course it'd likely be faster if I just added it myself, but the point in this post is to have the AI do as much as possible.

## Validating Input

OK, so we want to take the given words and validate that they are valid according to the rules of Letter Boxed.

These are complicated rules but let's see what we can do.

> Create a PHP class `Devbanana\LetterBoxed\Validator\WordValidator`. The class should be final and have one method called `validate()`.
> 
> The `validate()` method accepts an array of words, which cannot be empty, and a second array of sides, which are strings of 3 letters each. `$sides` is in uppercase and should be compared case insensitively. There must be exactly 4 sides.
> 
> Each word must pass the following rules:
> 
> * Each word must be at least 3 letters in length.
> * The first letter of each word must equal the last letter of the previous word, unless it is the first word.
> * Two consecutive letters of the word must come from different elements of `$sides`. For example, the first letter could come from `$sides[1]`, and the second letter from `$sides[3]` but cannot come from `$sides[1]`. There are always 4 elements of `$sides`.
> 
> Here is the stub class;
> 
> ```php
> <?php
> 
> declare(strict_types=1);
> 
> namespace Devbanana\LetterBoxed\Validator;
> 
> final class WordValidator
> {
>     /**
>      * @param string[] $words
>      * @param string[] $sides
>      */
>     public function validate(array $words, array $sides): bool
>     {
>         return true;
>     }
> }
> ```
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Validator;

final class WordValidator
{
    /**
     * @param string[] $words
     * @param string[] $sides
     */
    public function validate(array $words, array $sides): bool
    {
        // Ensure there are 4 sides
        if (count($sides) !== 4) {
            return false;
        }

        // Ensure there are words
        if (empty($words)) {
            return false;
        }

        // Go through each word and ensure that it meets the rules
        $lastLetter = null;
        foreach ($words as $word) {
            // Word must be at least 3 letters
            if (strlen($word) < 3) {
                return false;
            }

            // First letter must match last letter of previous word
            if ($lastLetter !== null && $word[0] !== $lastLetter) {
                return false;
            }

            // Two consecutive letters must come from different sides
            for ($i = 0; $i < strlen($word) - 1; $i++) {
                if (strtoupper($word[$i]) === strtoupper($word[$i + 1])) {
                    $side1 = array_search(strtoupper($word[$i]), array_map('strtoupper', $sides));
                    $side2 = array_search(strtoupper($word[$i + 1]), array_map('strtoupper', $sides));
                    if ($side1 === false || $side1 === $side2) {
                        return false;
                    }
                }
            }

            // Update last letter
            $lastLetter = $word[strlen($word) - 1];
        }

        // All tests have passed
        return true;
    }
}

```

This time I included an outline for the class so GPT-3 would get a better feel for my stylistic preferences, which seems to have helped.

I can already see from the start that this code won't work how I want it to.

It got most of the rules right, like the first letter of one word needing to match the last letter of the previous word.

But it totally botched the requirement that consecutive letters have to come from different sides, probably because it's so hard to describe it clearly and concisely.

But just to check, I wrote some [unit tests](https://github.com/devbanana/letter-boxed/blob/ed06dd31b46ca8e94d5a4579657a3a601df51160/tests/Validator/WordValidatorTest.php) that this needs to pass. I'm linking to the tests to save space here.

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.F...                                                               5 / 5 (100%)

Time: 00:00.007, Memory: 4.00 MB

There was 1 failure:

1) Devbanana\LetterBoxed\Tests\Validator\WordValidatorTest::testConsecutiveLettersFromSameSideAreInvalid
Failed asserting that true is false.

/path/to/letter-boxed/tests/Validator/WordValidatorTest.php:35

FAILURES!
Tests: 5, Assertions: 5, Failures: 1.
```

Failing just 1 out of 5 isn't horrible, but it still definitely misunderstood that rule.

I tried different ways of explaining it, but none resulted in much better code.

So I'm wondering if breaking down the process would yield better results. Let's try it.

### Find the Side by Letter

> Add a private method to the following class called `findSideByLetter`.
> 
> It accepts a string, `$letter`, and an array of 4 sides, `$sides`. Each element of `$sides` is a string with three letters.
> 
> The method should return the index of `$sides` that contains `$letter`, with a case insensitive search, or null if it cannot be found.
> 
> Stub class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

namespace Devbanana\LetterBoxed\Validator;

final class WordValidator
{
    /**
     * @param string[] $words
     * @param string[] $sides
     */
    public function validate(array $words, array $sides): bool
    {
        return true;
    }
    
    /**
     * @param string $letter
     * @param string[] $sides
     * @return int|null
     */
    private function findSideByLetter(string $letter, array $sides): ?int
    {
        $letter = strtolower($letter);
        foreach ($sides as $index => $side) {
            if (strpos(strtolower($side), $letter) !== false) {
                return $index;
            }
        }
        
        return null;
    }
}
```

This is good. I would have personally used `stripos()` instead of making both strings lowercase, but technically there's nothing wrong with it. Still, I'll modify the method to use `stripos()` instead.

Also, I'm happy it's learned and is now using return type declarations.

I'm also removing some of the redundant phpdoc information.

### Validate Each Word

Now let's create a method to validate each individual word.

> Add a private method to the following class called `validateWord`.
> 
> It accepts a string `$word`, and an array of 4 sides.
> 
> For each letter in `$word`, it should call `findSideByLetter()` to get the index of the side that letter is in.
> 
> If two consecutive letters have the same side, it is invalid and should return false.
> 
> Otherwise it is valid and should return true.
> 
> Current class:
> 
> `<code excluded for brevity>`

Updated class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Validator;

final class WordValidator
{
    /**
     * Validate that the given words are valid according to the provided sides.
     *
     * @param string[] $words The words to validate
     * @param string[] $sides The sides containing the 4 sets of letters
     */
    public function validate(array $words, array $sides): bool
    {
        return true;
    }

    /**
     * Validates that the given word is valid according to the provided sides.
     *
     * @param string   $word  The word to validate
     * @param string[] $sides The 4 sides of the puzzle
     */
    private function validateWord(string $word, array $sides): bool
    {
        $previousLetterSide = null;
        foreach (str_split($word) as $letter) {
            $letterSide = $this->findSideByLetter($letter, $sides);
            if ($previousLetterSide === $letterSide) {
                return false;
            }
            $previousLetterSide = $letterSide;
        }
        return true;
    }

    // ...
}

```

Can I just say how impressed I am that it generated accurate, properly aligned comments for the `validateWord()` method? The comments you see for `validate()` were added by me, but GPT-3 copied the style of my comments and applied the same style for `validateWord()`.

This code looks pretty accurate, but I can't test it until we complete the class.

### Validate the Words Match

Now let's create a method to validate the first letter of one word is equal to the last letter of the previous word.

> Create a private method called `validateWordsMatch()` to the following class.
> 
> It accepts two words, `$word1` and `$word2`.
> 
> It should validate that the first letter of `$word2` is equal to the last letter of `$word1`, with a case insensitive match. If not, it should return false.
> 
> Current class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Validator;

final class WordValidator
{
    /**
     * Validate that the given words are valid according to the provided sides.
     *
     * @param string[] $words The words to validate
     * @param string[] $sides The sides containing the 4 sets of letters
     */
    public function validate(array $words, array $sides): bool
    {
        if (\count($words) < 1) {
            return false;
        }

        foreach ($words as $word) {
            if (\strlen($word) < 3) {
                return false;
            }
        }
        if (\count($sides) !== 4) {
            return false;
        }

        for ($i = 0; $i < count($words) - 1; $i += 1) {
            if (!$this->validateWordsMatch($words[$i], $words[$i+1])) {
                return false;
            }
        }

        foreach ($words as $i => $word) {
            if (!$this->validateWord($word, $sides)) {
                return false;
            }
        }

        return true;
    }

    /**
     * Validates that the given two words are valid according to the provided matches.
     *
     * @param string $word1 The first word to validate
     * @param string $word2 The second word to validate
     */
    private function validateWordsMatch(string $word1, string $word2): bool
    {
        $lastLetter = strtolower($word1[strlen($word1) - 1]);
        $firstLetter = strtolower($word2[0]);
        return $lastLetter === $firstLetter;
    }

    /**
     * Validates that the given word is valid according to the provided sides.
     *
     * @param string   $word  The word to validate
     * @param string[] $sides The 4 sides of the puzzle
     */
    private function validateWord(string $word, array $sides): bool
    {
        $previousLetterSide = null;

        foreach (str_split($word) as $letter) {
            $letterSide = $this->findSideByLetter($letter, $sides);
            if ($previousLetterSide === $letterSide) {
                return false;
            }
            $previousLetterSide = $letterSide;
        }

        return true;
    }

    /**
     * Find the side that contains the given letter.
     *
     * @param string   $letter The letter to search for
     * @param string[] $sides  The 4 sides of the puzzle
     */
    private function findSideByLetter(string $letter, array $sides): ?int
    {
        foreach ($sides as $i => $side) {
            if (stripos($side, $letter) !== false) {
                return $i;
            }
        }

        return null;
    }
}

```

Straightforward enough.

### Try to Modify `validate()` Again

Now let's put all these steps together.

> Modify the `validate()` method of this class so it does the following:
> 
> * Validates `$words` contains at least one word, or returns false otherwise
> * Validates each element of `$words` contains at least 3 letters, and returns false otherwise
> * Validates `$sides` contains exactly 4 elements, or returns false otherwise
> * Uses `validateWordsMatch()` to validate the first letter of each word matches the last letter of the prior word
> * Uses `validateWord()` to validate that each word is valid
> 
> Current class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Validator;

final class WordValidator
{
    /**
     * Validate that the given words are valid according to the provided sides.
     *
     * @param string[] $words The words to validate
     * @param string[] $sides The sides containing the 4 sets of letters
     */
    public function validate(array $words, array $sides): bool
    {
        // Validate that $words contains at least 1 word
        if (\count($words) < 1) {
            return false;
        }

        // Validate that each element of $words contains at least 3 letters
        foreach ($words as $word) {
            if (\strlen($word) < 3) {
                return false;
            }
        }

        // Validate that $sides contains exactly 4 elements
        if (\count($sides) !== 4) {
            return false;
        }

        // Validate that the first letter of each word matches the last letter of the prior word
        for ($i = 0; $i < count($words) - 1; $i += 1) {
            if (!$this->validateWordsMatch($words[$i], $words[$i+1])) {
                return false;
            }
        }

        // Validate that each word is valid
        foreach ($words as $i => $word) {
            if (!$this->validateWord($word, $sides)) {
                return false;
            }
        }

        return true;
    }

    // ...
}
```

WOW is that inefficient! It loops through the words several times, when it could clearly be optimized.

But, I am impressed it is learning my code style and sticking to it. That's very cool!

But does it work? Let's see:

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.....                                                               5 / 5 (100%)

Time: 00:00.005, Memory: 4.00 MB

OK (5 tests, 6 assertions)
```

Nice! Everything works as it should!

## Optimize the Method

Now let's see if we can clean it up.

> Rewrite the `validate()` method so `$words` is only looped through once.
> 
> Current method:
> 
> `<code excluded for brevity>`
> 
> Modified method:

```php
    /**
     * Validate that the given words are valid according to the provided sides.
     *
     * @param string[] $words The words to validate
     * @param string[] $sides The sides containing the 4 sets of letters
     */
    public function validate(array $words, array $sides): bool
    {
        // Validate that $words contains at least 1 word
        if (\count($words) < 1) {
            return false;
        }

        // Validate that $sides contains exactly 4 elements
        if (\count($sides) !== 4) {
            return false;
        }

        // Validate that each element of $words contains at least 3 letters and is valid
        foreach ($words as $i => $word) {
            if (\strlen($word) < 3 || !$this->validateWord($word, $sides)) {
                return false;
            }

            // Validate that the first letter of each word matches the last letter of the prior word
            if ($i > 0 && !$this->validateWordsMatch($words[$i-1], $words[$i])) {
                return false;
            }
        }

        return true;
    }
```

Much better, and it still passes the tests.

One caveat though. You'll notice I wrote “current method” and “modified method” instead of including the entire class.

This is because if I included the entire class, it deleted some of my private helper methods and moved them into `validate()`, no matter how clearly I said not to touch any private methods, or only to modify the `validate()` method.

But when I just included the method I wanted modified, that's of course all it optimized.

But otherwise, the result is pretty much exactly what I'd want. I'm realizing I should have moved the check for the word being 3 or more letters into the `validateWord()` method, but I can do that myself.

So I think the takeaway from this is that if you have a complex task, it's better to break it down into manageable pieces, and the AI will handle it a lot better.

### Additional Validations

I'm realizing I forgot a criterion though: we need to make sure every letter in `$words` is contained in the puzzle.

Let's write a test to see if it passes:

```php
    public function testEveryLetterMustBeFoundInSides(): void
    {
        self::assertFalse($this->wordValidator->validate(['atoms'], $this->sides));
    }
```

Running this test:

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

...F..                                                              6 / 6 (100%)

Time: 00:00.005, Memory: 4.00 MB

There was 1 failure:

1) Devbanana\LetterBoxed\Tests\Validator\WordValidatorTest::testEveryLetterMustBeFoundInSides
Failed asserting that true is false.

/path/to/letter-boxed/tests/Validator/WordValidatorTest.php:46

FAILURES!
Tests: 6, Assertions: 7, Failures: 1.
```

Nope, it doesn't pass, so let's modify the `validateWord()` method.

> Modify this method so that if `findSideByLetter()` returns null for any letter, it is invalid and returns false.
> 
> Current method:
> 
> `<code excluded for brevity>`
> 
> Modified method:

```php
        /**
     * Validates that the given word is valid according to the provided sides.
     *
     * @param string   $word  The word to validate
     * @param string[] $sides The 4 sides of the puzzle
     */
    private function validateWord(string $word, array $sides): bool
    {
        if (\strlen($word) < 3) {
            return false;
        }

        $previousLetterSide = null;
        $letterSide = null;

        foreach (str_split($word) as $letter) {
            $letterSide = $this->findSideByLetter($letter, $sides);
            if ($previousLetterSide === $letterSide || $letterSide === null) {
                return false;
            }
            $previousLetterSide = $letterSide;
        }

        return true;
    }
```

Do the tests pass?

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

......                                                              6 / 6 (100%)

Time: 00:00.006, Memory: 4.00 MB

OK (6 tests, 7 assertions)
```

Yep!

Why it initializes `$letterSide` I have no idea. It's definitely not needed.

I want to add one more validation, to not accept any non-alpha characters in `$sides`.

First the test:

```php
    public function testNonAlphaCharactersInSidesAreInvalid(): void
    {
        self::assertFalse($this->wordValidator->validate(['atom'], ['XBM', 'INA', 'OYL', 'TE-']));
    }
```

Does it fail?

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.F.....                                                             7 / 7 (100%)

Time: 00:00.006, Memory: 4.00 MB

There was 1 failure:

1) Devbanana\LetterBoxed\Tests\Validator\WordValidatorTest::testNonAlphaCharactersInSidesAreInvalid
Failed asserting that true is false.

/path/to/letter-boxed/tests/Validator/WordValidatorTest.php:35

FAILURES!
Tests: 7, Assertions: 8, Failures: 1.
```

And now let's request the modification:

> Modify this method to disallow any non-alpha characters in `$sides`. Use `array_filter()` to perform the validation.
> 
> Current method:
> 
> `<code excluded for brevity>`
> 
> Modified method:

```php
    /**
     * Validate that the given words are valid according to the provided sides.
     *
     * @param string[] $words The words to validate
     * @param string[] $sides The sides containing the 4 sets of letters
     */
    public function validate(array $words, array $sides): bool
    {
        // Validate that $words contains at least 1 word
        if (\count($words) < 1) {
            return false;
        }

        // Validate that $sides contains exactly 4 elements that are all alphabetic
        if (\count($sides) !== 4 || array_filter($sides, "ctype_alpha") !== $sides) {
            return false;
        }

        // Validate that each element of $words contains at least 3 letters and is valid
        foreach ($words as $i => $word) {
            if (!$this->validateWord($word, $sides)) {
                return false;
            }

            // Validate that the first letter of each word matches the last letter of the prior word
            if ($i > 0 && !$this->validateWordsMatch($words[$i-1], $words[$i])) {
                return false;
            }
        }

        return true;
    }
```

Does it pass?

```bash
$ vendor/bin/phpunit tests/Validator/WordValidatorTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.......                                                             7 / 7 (100%)

Time: 00:00.004, Memory: 4.00 MB

OK (7 tests, 8 assertions)
```

Great!

You'll notice I was pretty specific, telling it to use `array_filter()` for the validation. This is because otherwise it was looping through the sides and I just thought there was a nicer way of doing things.

Also I ran this twice, because the first time it used `preg_match()` which I thought was far less efficient than `type_alpha()`.

You can see the final version of [`WordValidator`](https://github.com/devbanana/letter-boxed/blob/ef3c8e43f61c76e196b65971e6b3225606caeb6a/src/Validator/WordValidator.php), which I did modify a bit to be more to my liking.

### Add the Validator to the Command

Let's add `WordValidator` to `RemainingCommand`.

> Modify this class to add `Devbanana\LetterBoxed\Validator\WordValidator` as a dependency. In  ``execute()`, it should pass `$words` and `$letters` to theWordValidator::validate()` method. If it is false, it should output the error, `The provided words are invalid.`
> 
> Current class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Command;

use Devbanana\LetterBoxed\Validator\WordValidator;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'remaining', description: 'Determine remaining letters after word(s)')]
final class RemainingCommand extends Command
{
    private WordValidator $wordValidator; 

    public function __construct(WordValidator $wordValidator)
    {
        $this->wordValidator = $wordValidator;
        parent::__construct();
    }

    // ...

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $io->title('Remaining');

        $letters = $input->getArgument('letters');
        if (!\is_array($letters) || \count($letters) !== 4) {
            $io->getErrorStyle()->error('There must be exactly 4 sets of letters.');

            return Command::INVALID;
        }

        $words = $input->getOption('word');
        if (!\is_array($words) || empty($words)) {
            $io->getErrorStyle()->error('At least 1 word is required.');

            return Command::INVALID;
        }

        if (!$this->wordValidator->validate($words, $letters)) {
            $io->getErrorStyle()->error('The provided words are invalid.');

            return Command::INVALID;
        }

        // TODO: add logic

        return Command::SUCCESS;
    }
}

```

Nice!

All we have to do is to pass `WordValidator` to `RemainingCommand` in our entry point script. We could ask the AI to do this but it'd take longer to form the request and copy the result as to just do it ourselves.

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

require_once __DIR__ . '/vendor/autoload.php';

use Devbanana\LetterBoxed\Command\RemainingCommand;
use Devbanana\LetterBoxed\Validator\WordValidator;
use Symfony\Component\Console\Application;

$app = new Application('Letter Boxed Solver');
$app->add(new RemainingCommand(new WordValidator()));
$app->run();
```

Now let's try passing an invalid word:

```bash
$ ./letterboxed remaining XBM INA OYL TEC -w tea
 
Remaining
=========


                                                                                
 [ERROR] The provided words are invalid.                                        
                                                                                
```

## Strip Words From Letters

Now we want to create a utility that strips the letters of the provided words from the letters in the puzzle, and only returns the letters that are left.

Just as before, let's write some [unit tests](https://github.com/devbanana/letter-boxed/blob/9123e3817abd517049aa5525cfe50de1627be238/tests/Util/StripWordsTest.php) for this.

And now let's see if we can get it in one shot.

> Modify this class to strip the letters from `$words` from `$sides` and return the remaining letters as an alphabetically-ordered string.
> 
> Current class:
> 
> ```php
> <?php
> 
> declare(strict_types=1);
> 
> namespace Devbanana\LetterBoxed\Util;
> 
> final class StripWords
> {
>     /**
>      * Strip the letters of the given words from the letters in the given sides.
>      *
>      * @param string[] $words The words to strip
>      * @param string[] $sides The sides of the puzzle
>      *
>      * @return string The remaining letters
>      */
>     public function strip(array $words, array $sides): string
>     {
>         return '';
>     }
> }
> 
> ```
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Util;

final class StripWords
{
    /**
     * Strip the letters of the given words from the letters in the given sides.
     *
     * @param string[] $words The words to strip
     * @param string[] $sides The sides of the puzzle
     *
     * @return string The remaining letters
     */
    public function strip(array $words, array $sides): string
    {
        $remainingLetters = [];
        
        foreach ($sides as $side) {
            foreach (str_split($side) as $letter) {
                $remainingLetters[$letter] = 1;
            }
        }
        
        foreach ($words as $word) {
            foreach (str_split($word) as $letter) {
                unset($remainingLetters[$letter]);
            }
        }
        
        ksort($remainingLetters);
        
        return implode('', array_keys($remainingLetters));
    }
}

```

Cool. Does it pass?

```bash
$ vendor/bin/phpunit tests/Util/StripWordsTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.FFFF                                                               5 / 5 (100%)

Time: 00:00.007, Memory: 6.00 MB

There were 4 failures:

1) Devbanana\LetterBoxed\Tests\Util\StripWordsTest::testOnlyStripPresentLetters
Failed asserting that two strings are identical.
--- Expected
+++ Actual
@@ @@
-'BCEILNXY'
+'ABCEILMNOTXY'

/path/to/letter-boxed/tests/Util/StripWordsTest.php:35

2) Devbanana\LetterBoxed\Tests\Util\StripWordsTest::testStripMultipleWords
Failed asserting that two strings are identical.
--- Expected
+++ Actual
@@ @@
-'CILNX'
+'ABCEILMNOTXY'

/path/to/letter-boxed/tests/Util/StripWordsTest.php:40

3) Devbanana\LetterBoxed\Tests\Util\StripWordsTest::testReturnEmptyStringIfAllLettersStripped
Failed asserting that two strings are identical.
--- Expected
+++ Actual
@@ @@
-''
+'ABCEILMNOTXY'

/path/to/letter-boxed/tests/Util/StripWordsTest.php:45

4) Devbanana\LetterBoxed\Tests\Util\StripWordsTest::testDuplicateLettersShouldBeRemoved
Failed asserting that two strings are identical.
--- Expected
+++ Actual
@@ @@
-'BEILNXY'
+'ABEILMNOTXY'

/path/to/letter-boxed/tests/Util/StripWordsTest.php:52

FAILURES!
Tests: 5, Assertions: 5, Failures: 4.
```

Not even a little.

Looking at the code, I see that it's doing a case sensitive search, because I neglected to mention that. So let's see if we can modify it to be case insensitive.

> Modify the following class to make the match case insensitive but still return the string in uppercase.
> 
> Current class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Util;

final class StripWords
{
    /**
     * Strip the letters of the given words from the letters in the given sides.
     *
     * @param string[] $words The words to strip
     * @param string[] $sides The sides of the puzzle
     *
     * @return string The remaining letters
     */
    public function strip(array $words, array $sides): string
    {
        $remainingLetters = [];

        foreach ($sides as $side) {
            foreach (str_split(strtoupper($side)) as $letter) {
                $remainingLetters[$letter] = 1;
            }
        }

        foreach ($words as $word) {
            foreach (str_split(strtoupper($word)) as $letter) {
                unset($remainingLetters[$letter]);
            }
        }

        ksort($remainingLetters);

        return implode('', array_keys($remainingLetters));
    }
}
```

Does it pass?

```bash
$ vendor/bin/phpunit tests/Util/StripWordsTest.php 
PHPUnit 9.6-dev by Sebastian Bergmann and contributors.

.....                                                               5 / 5 (100%)

Time: 00:00.003, Memory: 6.00 MB

OK (5 tests, 5 assertions)
```

Perfect!

Note how I said it still had to be returned as an uppercase string. That's because when I ran this without that qualifier, it made all the strings lowercase, and so the tests failed.

I like this implementation. I'm not sure it's exactly what I would have chosen, but it's creative and it works. It might actually be more compact than what I would have chosen.

### Add it to the Command

> Modify the following class to add a dependency to the constructor for `Devbanana\LetterBoxed\Util\StripWords`. After all the validation, it should pass `$words` and `$letters` to `StripWords::strip()` in order to strip the words from the puzzle, and output the result like this:
> 
> If the string is not empty, then output:
> 
> ```
> The remaining letters are:
> {$remaining}
> ```
> 
> If the string is empty, then output the following:
> 
> `<info>You've solved the puzzle! All the letters were used.</info>`used.`
> 
> Current class:
> 
> `<code excluded for brevity>`
> 
> Modified class:

```php
<?php

declare(strict_types=1);

namespace Devbanana\LetterBoxed\Command;

use Devbanana\LetterBoxed\Util\StripWords;
use Devbanana\LetterBoxed\Validator\WordValidator;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'remaining', description: 'Determine remaining letters after word(s)')]
final class RemainingCommand extends Command
{
    private WordValidator $wordValidator;
    private StripWords $stripWords;

    public function __construct(WordValidator $wordValidator, StripWords $stripWords)
    {
        $this->wordValidator = $wordValidator;
        $this->stripWords = $stripWords;
        parent::__construct();
    }

    protected function configure(): void
    {
        $this
            ->setHelp('Determine which letters are left after the given words')
            ->addArgument('letters', InputArgument::REQUIRED | InputArgument::IS_ARRAY, 'The letters of the puzzle (each side separated by a space)')
            ->addOption('word', 'w', InputOption::VALUE_REQUIRED | InputOption::VALUE_IS_ARRAY, 'The word(s) in the puzzle so far')
        ;
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $io->title('Remaining');

        $letters = $input->getArgument('letters');
        if (!\is_array($letters) || \count($letters) !== 4) {
            $io->getErrorStyle()->error('There must be exactly 4 sets of letters.');

            return Command::INVALID;
        }

        $words = $input->getOption('word');
        if (!\is_array($words) || empty($words)) {
            $io->getErrorStyle()->error('At least 1 word is required.');

            return Command::INVALID;
        }

        if (!$this->wordValidator->validate($words, $letters)) {
            $io->getErrorStyle()->error('The provided words are invalid.');

            return Command::INVALID;
        }

        $remaining = $this->stripWords->strip($words, $letters);
        
        if (!empty($remaining)) {
            $io->writeln("The remaining letters are:");
            $io->writeln($remaining);
        } else {
            $io->writeln("<info>You've solved the puzzle! All the letters were used.</info>");
        }
       

        return Command::SUCCESS;
    }
}

```

And let's add it to the entry point:

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

require_once __DIR__ . '/vendor/autoload.php';

use Devbanana\LetterBoxed\Command\RemainingCommand;
use Devbanana\LetterBoxed\Util\StripWords;
use Devbanana\LetterBoxed\Validator\WordValidator;
use Symfony\Component\Console\Application;

$app = new Application('Letter Boxed Solver');
$app->add(new RemainingCommand(new WordValidator(), new StripWords()));
$app->run();

```

Now let's see if it works:

```bash
$ ./letterboxed remaining XBM INA OYL TEC -w exclamation

Remaining
=========

The remaining letters are:
BY
```

And let's try solving the puzzle:

```bash
$ ./letterboxed remaining XBM INA OYL TEC -w bye -w exclamation

Remaining
=========

You've solved the puzzle! All the letters were used.
```

I can't show the green text but it does come out green if your terminal supports color.

## Final Thoughts

You can view the final state of this application as of this writing by [visiting the repository](https://github.com/devbanana/letter-boxed/tree/fc9de5aec8597ccfeaee213c589c719f6fc8754f).

As you can see, GPT-3 has some really powerful capabilities when it comes to programming.

However, in my opinion you have to know how to guide it. You either have to know how to explain it clearly enough, or to break it down into small enough steps that it doesn't get confused.

I also think having unit tests to guide the process really helped, since I had a clear idea of success.

Do I think it can replace programmers? Not really. Could it perhaps in the distant future? Maybe, but I don't think that's its strength.

Someone with no programming knowledge clearly couldn't sit down and generate this console application purely with GPT-3. It required plenty of technical oversight and correction along the way.

But still, it is extremely helpful for the parts it is good at.