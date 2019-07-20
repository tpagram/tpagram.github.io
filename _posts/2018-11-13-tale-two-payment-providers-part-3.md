---
layout: post
title: Stripe Down Under
subtitle: How we earned our Stripes — Part 3
description: Third article in a series of blog posts about the migrating to Stripe.
category: tech
excerpt_separator: <!--more-->
comments: true
---

![](/assets/two-payment-providers-header-3.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee)

*This is the third article in a series of blog posts about our transition towards Stripe as Airtasker’s main payment provider.*

February 2018, we launched in the UK. We’d be lying if we said there weren’t hiccups — we certainly had a few bugs to clean up and a lot of work getting to feature parity with our old provider. But, most importantly, it worked. It worked great.

Our MVP was proven. Stripe’s architecture was meshing well with our own system and everyone was happy. Developers loved working with the [pristine Stripe API](https://stripe.com/docs/api). Support agents loved the cleaner interface. DevOps loved the zero downtime. Our UK team loved that we could actually accept payments in London. And finance loved how much money we were saving.

Given how well-received it was, there was only one option: get it on Australia.
<!--more-->

One problem — payments in the UK and payments in Australia are two completely different playing fields.

In the UK, we were starting from scratch. No users had existing credit cards or bank accounts, or worse — tasks mid-flight. We had a small cohort of users receiving a lot of attention from a specialised support team.

In Australia, we go through in excess of 30,000 tasks a week. We’ve got close to a million combined credit cards and bank accounts. And most importantly, there’s a lot of dollarydoos floating through our payment system from poster to tasker. There’s no margin for error.

### Planning iteration

![Could it be this simple?](/assets/simple-plan.png){:width="100%"}
*Could it be this simple?*

We needed a plan. So we sat down and made one.

As it should be, our first mental pitstop was the simplest: a hard cutover. One day, all tasks are in our old provider. The next, Stripe.

It may sound simple, but the implementation is complex. This plan would involve taking all of the money for in-flight tasks currently escrowed in our old provider and transferring it into Stripe with the appropriate payout structure in place. Yeah, that’s not happening.

This did surface a definitive point — we weren’t going to transfer escrowed money. And the logical conclusion of that is that in-flight tasks must be left to run their course in the old provider. This led us to what we called a semi-hard cutover plan: on day one, all *new* tasks are in Stripe.

This plan was a lot more palatable, but there were a lot of edge cases to consider. For a user to post a task in either Stripe or our old provider, they need a credit card record created in the respective service. As a parallel, taskers also need a bank account record in the correct service. And herein lies the source of a million headaches.

On launch day, we are running under the assumption that all users have their existing credit cards and bank accounts migrated to Stripe. But if there are still tasks from the old provider active in the system from before the cutover, how are new taskers without bank accounts in the old provider going to accept payment from them? How are old taskers that want to update their bank account details going to get paid from these old tasks? How do we even select the right bank account in our codebase in the first place?

Grass was greener on the credit card side, since once a task is assigned the money is always escrowed into Stripe — not a problem. But there was still one nasty edge case — what if a poster wants to give additional funds to their tasker on a pre-cutover task? We’ve got to select the right service.

There was the question of whether this was an acceptable loss. If for one measly day of the year, taskers can’t bid on day-old tasks, is it the end of the world? Can we accept these edge cases? Is that too much of a bad user experience?

Yes, it is. At Airtasker we strive for quality, and we weren’t going to accept compromise on the experience of our users.

With that decided, another clear conclusion fell out of our line of reasoning: we had to support both types of tasks running concurrently on the platform. Without them realising it, users would be posting or bidding on tasks either in Stripe or our old provider. The work to support that was arduous but unavoidable.

There was a silver lining to this logic. If the work to support both types of tasks to run in parallel needed to be done no matter what, then we could broaden the assumptions for our plan. We could go even softer. More … graduated.

### Our graduated migration plan

![A graduated plan.](/assets/graduated-plan.png){:width="100%"}

Our third plan was similar to the second: after a given date, we would introduce Stripe tasks onto the platform and leave existing tasks in the old provider. However, we would also let new tasks be created in the old provider — essentially having both kinds of tasks running concurrently on Airtasker.

The decision point that determined whether a task’s funds would be escrowed on Stripe was whether the user had added or updated their credit card. This was a master stroke for several reasons.

Firstly, it let us run tasks in both providers without worrying about syncing credit cards between the two. Both providers tokenise cards directly and hand the token to us afterwards. In order to pass it to both providers, we’d have to have the sensitive card information running through our backend — a [PCI](https://en.wikipedia.org/wiki/Payment_Card_Industry_Data_Security_Standard) compliance nightmare.

Secondly, it solved another issue we’d been puzzling over — how to migrate our credit cards without losing any along the way. We’ll speak about this more when we get to [Part 5]({% post_url 2018-11-13-tale-two-payment-providers-part-5 %}), but here’s the gist.

We got our old and new providers talking to each other (awkward), and they graciously agreed to do a token migration. Essentially, the old provider would supply the info for half a million credit cards to Stripe, and Stripe would tokenise each one. Fantastic.

Problem is, they only wanted to do it once since it’s an arduous process. Further more, even if they did do it a second time, there was no way for them to only migrate the cards that been added or updated since the last migration. They’d have to do all of them again, possibly creating duplicates.

This created a dilemma. If our providers migrated tokens before we cut over, we’d have a period between the migration and cutover where card information would be lost. And if we did it after the cutover, then there would be a gap period where a cohort of users selected to create Stripe tasks wouldn’t have their card details in Stripe in order to do so.

Trying to plan so they both happened at the same time was a pipe dream. Hoping to deploy or run two complicated pieces of software at the same time is just asking for trouble. Let alone the logistical nightmare of having to organise this feat between three different companies. Forget it. It wasn’t going to happen.

Our graduated plan averted this issue. By selecting the cohort of users to experience Stripe to be exactly those users that add or update Stripe credit cards, we ensured that every Stripe user had access to a card and that, once the credit migration took place, there were no updates to cards in the old system to worry about.

We called this period Phase 1, because, well, it was the first phase and we were being uncreative. It began after we organised the bank account sync to Stripe and lasted until our providers migrated tokens and we were confident our new payments engine was working well.

This lead into, you guessed it, Phase 2. Now we had every single user’s card on Stripe — if you were an old user it was migrated by our providers, while if you were a new user you were creating it in Stripe directly.

This meant we could finally turn off the bank account sync to our old provider and make every new task a Stripe task. The semi-hard cutover finally had its chance to shine. Old tasks would continue to remain on our old provider, but their numbers would slowly dwindle as they were completed by our hard-working taskers.

Finally, we have Phase 3. One day, months in the future, an unwitting tasker will pack up their equipment, dust off their hands after a job well done and press the request payment button. Their poster, delighted with their handiwork, will release the funds and give a enthused review. And we’ll be standing in front of our computers, champagne in hand, watching their task page progress as these two unintentionally mark the end of an era. The final task in our old provider, and a complete migration to Stripe.

That was the plan, at least. Before we could dream of that, we first had to tackle two glaring problems: syncing our bank accounts, which we’ll address in [Part 4]({% post_url 2018-11-13-tale-two-payment-providers-part-4 %}), and migrating our credit cards, which we’ll close up in [Part 5]({% post_url 2018-11-13-tale-two-payment-providers-part-5 %}).
