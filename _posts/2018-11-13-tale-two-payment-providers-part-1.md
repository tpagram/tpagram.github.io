---
layout: post
title: A Tale of Two Payment Providers
subtitle: How we earned our Stripes — Part 1
description: First article in a series of blog posts about the migrating to Stripe.
category: tech
excerpt_separator: <!--more-->
comments: true
---

![](/assets/two-payment-providers-header-1.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/@tpagram/a-tale-of-two-payment-providers-8788e4401b0c)

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

In [Part 2]({% post_url 2018-11-13-tale-two-payment-providers-part-2 %}), we talk about our MVP: Stripe in the UK.

In [Part 3]({% post_url 2018-11-13-tale-two-payment-providers-part-3 %}), we talk about our graduated migration plan for Australia.

In [Part 4]({% post_url 2018-11-13-tale-two-payment-providers-part-4 %}), we talk about the trials and tribulations of getting 250,000 bank accounts into Stripe.

In [Part 5]({% post_url 2018-11-13-tale-two-payment-providers-part-5 %}), we talk about the joys of a JSON file with half-a-million credit cards.
