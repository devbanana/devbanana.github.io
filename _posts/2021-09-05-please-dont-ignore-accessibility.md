---
layout: post
title: Please Don’t Ignore Accessibility
date: 2021-09-05 20:54:01 -0400
categories: accessibility
---

I've been using a service called [IEX Cloud](https://iexcloud.io) for going on two years now. It's an API that allows me to fetch stock information programmatically.

I just got an email the other day that they failed to charge my card. I realized I had recently changed my credit card and forgot to update it, so I went to log in to update my payment information.

Now as I mentioned [on the about page][about], I'm visually impaired and so use a screen reader. Specifically I use VoiceOver on the Mac.

And when I logged in to the IEX console, it was totally unusable.

This was on Safari, and so luckily I was able to get it to work on Chrome. It was still mostly inaccessible, but I was able to work around the issue.

Unfortunately, accessibility is something most businesses rarely think about. I just had to notify another application I use of accessibility issues they had. It's just not something developers test very often.

Now the thing is, this issue has quite an easy fix. But the question is, will I be able to convince them that the issue is worth fixing?

It's such a simple issue, it's worth discussing it in this post, because it really demonstrates how easily a page can be made inaccessible.

## Aria Roles

Because the thing is, but for this one issue, the page would actually be perfectly accessible.

I fixed the issue myself using JavaScript injection, and was then able to navigate perfectly well.

But I mean, asking me to manually modify the DOM any time I want to log into their system, is probably asking a bit much, even if I **can** do it.

I'm insanely grateful sometimes that I **am** a programmer and can use these unconventional ways of fixing problems.

So what was the issue?

They literally wrapped their entire page in a `<section>` element with `role="button"`.

Here's what it looked like:

```html
<section role="button" id="account-home" class="col-12 min-height-100vh relative overflow-x-hidden">
    <!--...-->
</section>
```

And that wrapped the entire page.

Now that little `role="button"`, if you aren't aware, tells my screen reader to treat everything inside that element as a button.

It's often used for things like this, if you don't want to use an actual button element:

```html
<p role="button" onclick="doSomething();">I'm a Button</p>
```

This is what's called an [aria role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/ARIA_Techniques), and there are lots of other roles besides "button".

But, because they applied `role="button"` to a container element that wrapped the entire page, now it tells my screen reader to treat the whole page as a button.

And if the whole page is a button, that means I can't do anything with it, besides click the whole element of course. But I can't find links, can't read text, etc. It's just all one big button that does nothing.

And of course that's not the only place they did this. They also applied the button role to the container that wrapped the main content of the page, like this:

```html
<div role="button" class="account-container account-min-height">
    <!--...-->
</div>
```

So even after I had removed the role from that `<section>` tag, I had other instances to contend with.

## Getting Businesses to Listen

This is literally just one attribute, applied to a couple of elements, that is making the whole thing inaccessible to screen readers.

So I emailed them with this bug, and with how to fix it.

The question is, will they fix it?

Of course, if they don't, I definitely won't be renewing my subscription with them.

But I've had bad experiences with other applications or services that didn't take accessibility seriously.

I recently just finally got another application I use to make their UI more accessible, after bugging them regularly for 3 or 4 years about it. It took me posting in their Facebook group to finally get their attention, because so many other users were outraged at their lack of response.

## Accessibility Isn't Hard

The thing is, it's not all that difficult. Especially on the web, most things you could do with HTML, or even JavaScript, are perfectly accessible.

Even if you want to do something dynamic using JavaScript, most popular JavaScript libraries are perfectly accessible.

So, it's really hard these days to make something **not** accessible. Of course, people still manage to do it all the time.

Here are a few easy but important accessibility tips:

* Put alt text in your images, especially ones that are meant to communicate something to the user.
* Put an `<h1>` heading before your main content. That's the first thing I look for when navigating a page.
* Use tags for what they're meant to be used for. If you want a link, use an `<a>` element. If you want a button, use `<button>` or `<input type="button">`.
* If you use another tag than the default, then find an equivalent aria role and apply it to the element. Example, `<span role="link">I'm a link</span>`.
* If you want features like autocomplete or other dynamic widgets, use an existing, popular JavaScript library that already has accessibility built-in.

There's more to accessibility than that, of course, but these will go a very long way to making any website accessible.

Of course, I'm always happy to answer questions about accessibility, so just let me know in the comments below, or feel free to contact me using my contact information at the bottom of the page.

[about]: /about/
