---
layout: post
title: To Our Credit
subtitle: How we earned our Stripes — Part 5
description: This is the fifth article in a series of blog posts about the transition towards Stripe as Airtasker’s main payment provider.
category: Technology
excerpt_separator: <!--more-->
comments: true
---

![](https://cdn-images-1.medium.com/max/6002/1*PreUrIMpeE_v2dRm-qSYBA.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/@tpagram/to-our-credit-1cea3c41dfbb)

*This is the fifth article in a series of blog posts about our transition towards Stripe as Airtasker’s main payment provider.*

In [Part 3](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee), we made a decision. We unleashed Stripe onto our system before our poster’s credit cards were migrated, creating two concurrent streams of payment. I shouldn’t say it was a damn fine call, since the impact was more due to serendipity than insight, but damn was it a fine call.
<!--more-->

Why? Well, while we intended the token migration to be a fast follow, it took close to three months to see through to the end. There’s no way we could have waited that long to switch to Stripe.

What was the cause of this hold-up? Was it due to unforeseen implementation problems, followed by a heroic effort of stringent, impressive engineering? Nope. Competing priorities? Under-resourcing? Total global collapse? Not even close.

Emails. It took three months because of emails. Let me spin you a yarn about the pleasures of organising a migration between four separate companies.

## Simple enough, isn’t it?

Stripe, bless their hearts, have an optimistic view about migrating credit cards. Their documentation essentially says the following: politely ask your old payment provider to migrate their data. Then, behind the scenes, without you worrying your pretty head about it, all of the super secure payment data will be migrated over to Stripe. Afterwards, Stripe will provide you with a terrifyingly large, Atom-crashing JSON file, which will map your old payment methods to the new Stripe ones.

We got stuck on the first step. But let’s start from the beginning.

In the Airtasker platform, when a user enters a credit card, all of the private, sensitive information is sent off to our payment provider to store. In return, they give us some harmless form of identifying that credit card: a token. For each credit card, we store a record of that token and some other bits and pieces of info. So instead of storing that Hugh Jass has a credit card number of 4111 1111 1111 1111 and an expiry of 11/23, we store some gibberish like 3f2@314dfH8sad-234and the last four digits 1111.

What that means is our database had half-a-million tokens corresponding to credit card information stored in our old provider. Each time we wanted to use the credit card, we sent off the token and our old provider would use the secure credit card info.

Yet if we wanted to process our payments through Stripe, those tokens now became completely useless. Not only does Stripe not recognise the tokens, it doesn’t even have a record of the credit card in the first place.

This is what Stripe were suggesting. Get your old provider to send us two important things: the sensitive credit card information and some identifier that Airtasker recognises. That way we can create our own secure records for each card and then provide a file that maps the old token to the new one. All Airtasker have to do is run through their payment records and create a new record with the new token for each. Simple. Right?

## It’s never simple

After some discussion, it turned out there was an extra step we were missing. Our old provider dealt with **performing** the payments. When it came to **storing** credit cards, it used an external service that specialises in handling secure payment information.

![The circle of life.](https://cdn-images-1.medium.com/max/6000/1*XmEOhI0dw-XI40il3S0Fuw.png)*The circle of life.*

Ah, a new challenger. Let’s recap what had to go down now. Airtasker had to provide our old provider with a list of tokens for the credit cards that needed to be migrated. Our old provider needed to turn that list of tokens into their **own** tokens provided by their token-storing service, and then ask them to pass the corresponding credit card info to Stripe. Stripe would receive that info and create internal records with new tokens, which they would pass to us. And then we would map our old tokens to the new ones.

But hang on. Didn’t Stripe need to be passed two things? Not only the credit card information, but also a piece of identifying information that we would recognise. Yet Stripe is being passed the data by the token storing service, which we have no relationship with. Only our old provider has information that we would recognise in our own database.

So not only did we need to coordinate this migration effort between four companies, we also needed to ensure that the correct piece of identifying data was being passed from our old provider into the token securing service, and that it would emerge intact for consumption by Stripe.

How did we achieve this? Communication. It was a menagerie of personalities and interests. We had Stripe the excitable puppy, our old provider the apathetic cat, the token securer the confused hamster, and Airtasker: crying in the corner.

![A few days of email communication.](https://cdn-images-1.medium.com/max/6000/1*eZo6Wjiv-mdLQysX3EnnYg.png)*A few days of email communication.*

As an illustration, above is a snapshot of a few days of emails between the four of us. I was going to include the full set, but was advised to exclude it on the basis of ‘well, that just makes us all look incompetent.’

## A new day

Eventually, one fateful morning, we walked into the office, cracked open up our email clients with our faded, broken keyboards, and found a JSON file waiting for us from Stripe. Hallelujah!

    "999999": {
        "id": "cus_abcde",
        "cards": {
          "123456789": {
            "id": "card_123456",
            "fingerprint": "1234",
            "last4": "1111",
            "brand": "Visa"
          },
        }
      }

Above is an anonymised example of the contents of that JSON file. Here, a poster with an identifying record of 999999 is now a Stripe customer that we can access with a customer id of cus_abcde. Their single active card, which was some meaningless token from the secure token provider, has a new unique id card_123456 and a bunch of non-sensitive info we can use for sanity checks.

Now came the part actually involving engineering: another terrifying migration script to run on production. Luckily at this point, having already written a [terrifying migration rake task for bank accounts](https://medium.com/@tpagram/putting-our-accounts-in-order-3366d17ce549), we were pros.

Log the task to death. Make it reversible. Give fine grain control over the who and how many users the task runs on. Test locally, then on staging. Test on 1 users, then 10, then 100, 1000, the full batch. Sacrifice an intern to grant the favour of the gods.

In the end the task was a lot more straightforward than the bank account equivalent, since there weren’t any external calls involved. Still, it’s worth celebrating this fact: it went without a hitch.

One morning, most of our users were creating tasks in our old provider. A few hours and half-a-million tokens later, everyone was creating tasks in Stripe.

## The Promised Land

We were on Stripe. After close to a year of effort, every single new task on the platform was a Stripe task. The number of tasks on our old provider was dwindling as taskers completed their active tasks. In a few months time, we could purge the old payments code from our codebase. Paradise.

A little like a dog that finally catches its tail, now that the Stripe migration was over we didn’t really know what to do with ourselves.

Actually, that’s completely untrue. We knew exactly what to do: go on a celebratory team outing and drink a lot of beer.

Looking back, there was indeed a lot to celebrate. The highest compliment I give Stripe is that we quickly forgot it existed after the migration was over. In the old days, our payment provider was never far from our mind. Now it churns in the background, letting us focus on what is most important: delivering features for our customers.

If you’ve made it through this saga of articles, well done. It’s been a lengthy read. Let’s recap.

In [Part 1](https://medium.com/@tpagram/a-tale-of-two-payment-providers-8788e4401b0c), we set the scene: a small team of engineers determined to guide Airtasker through the murky depths of our old provider and into the green pastures of Stripe.

In [Part 2](https://medium.com/@tpagram/stripe-in-the-uk-dec797a7585b), we used our launch of Airtasker in the UK to test an MVP of Stripe on a small number of users.

In [Part 3](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee), we took our successful MVP across to the main Australian platform and formed our plan for the migration.

In [Part 4](https://medium.com/@tpagram/putting-our-accounts-in-order-3366d17ce549), we migrated across all of our tasker’s bank accounts and switched on Stripe in Australia.

And finally, in Part 5, we migrated our credit cards — completing the migration.

If you’re considering changing payment providers, whether to Stripe or otherwise, or if you were simply curious how such a feat could be done, I hope these series of articles was illustrative. Thank you.
