---
layout: post
title: A Tale of Two Payment Providers
subtitle: How we earned our Stripes — Part 1
description: This is the first article in a series of blog posts about the transition towards Stripe as Airtasker’s main payment provider.
category: Technology
excerpt_separator: <!--more-->
comments: true
---

![](/assets/two-payment-providers.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/airtribe/a-tale-of-two-payment-providers-8788e4401b0c)

*This is the first article in a series of blog posts about our transition towards Stripe as Airtasker’s main payment provider.*


It’s 2017. You’re a rapidly scaling start-up trying to launch globally, but your payment provider (face blurred to protect their identity) isn’t available outside Australia. What do you do?

Switch to [Stripe](https://stripe.com), of course. And that’s what [Airtasker](https://www.airtasker.com) did.

<!--more-->

“Ok, sounds great!” we said, after googling Stripe for 5 minutes. “Let’s do it! How long should we book in to switch over? A couple of days? A few weeks?”

Try ten months.

In the dark ages (or the good old days depending on your perspective), long before we hit our inflection point of growth and knew what a sensation we were, we made a naive assumption.

We were fresh off of signing a deal with our old payment provider. “Wow,” we said. “These guys are great. We’ll never need to switch.”

Wrong, wrong, wrong.

Our old payment engine had tentacles stretching into every part of our codebase. We had close to a million credit cards and bank accounts tokenised in our old provider. It took a coordinated engineering effort of four backend engineers, four frontend engineers and one long-suffering PM to pry the beast free and deliver us into the promised land of our new payment engine, Airtasker Pay.

This is their story.

It was a long journey, and so we’ve broken this article into 5 parts:

In [Part 2](https://medium.com/@tpagram/stripe-in-the-uk-dec797a7585b), we talk about our MVP: Stripe in the UK.

In [Part 3](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee), we talk about our graduated migration plan for Australia.

In [Part 4](https://medium.com/@tpagram/putting-our-accounts-in-order-3366d17ce549), we talk about the trials and tribulations of getting 250,000 bank accounts into Stripe.

In [Part 5](https://medium.com/@tpagram/to-our-credit-1cea3c41dfbb), we talk about the joys of a JSON file with half-a-million credit cards.
