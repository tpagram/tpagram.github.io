---
layout: post
title: Putting Our Accounts in Order
subtitle: How we earned our Stripes — Part 4
description: This is the fourth article in a series of blog posts about the transition towards Stripe as Airtasker’s main payment provider.
category: Technology
excerpt_separator: <!--more-->
comments: true
---

![](https://cdn-images-1.medium.com/max/6002/1*ikeGJuLrXRqkn7wiq2ChDA.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/@tpagram/putting-our-accounts-in-order-3366d17ce549)

*This is the fourth article in a series of blog posts about our transition towards Stripe as Airtasker’s main payment provider.*

Despite all the excellent work getting our own system ready to accept Stripe tasks, we had one glaring omission. Our taskers are paid their hard-earned funds into bank accounts, which were stored and tokenised in the ecosystem of our old payment provider. We had about a quarter of a million of these. If we wanted our taskers to continue to get their money, we needed to add these bank accounts to Stripe.
<!--more-->

What’s more —we had to time this with our [Phase 1 launch](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee). We obviously needed the bank accounts synced before launch day, but what about the gap between those two events? If any users updated their bank account between the sync and the Stripe launch, we still needed to capture them.

Solution? Write a big, gnarly migration script.

### A dummies guide to designing a big, gnarly migration script

Like all projects, migrating a quarter-of-a-million bank accounts sounds a lot less complicated when you break it into components. From a high level, a migration script only needs do two things.

First, it needs to find users that require being migrated. And then, for each user, it migrates them.

When you put it like that, it sounds easier already.

Let’s start with finding our users. It’s a given that if you want to write a big, gnarly migration script, you’re probably going to write big, gnarly SQL query. But what conditions do we want to impose? What defines a user in need of being migrated? Well, they must:

1. Be in Australia. Everyone outside of Australia was already on Stripe.

1. Have an associated bank account record for our old provider, but not with Stripe.

1. **OR** have updated their old provider’s bank account more recently than their Stripe equivalent.

1. Fit within our constraints, for use in testing. We passed in a minimum and maximum ID, as well a total limit on the number of records, in order to give ourselves finer control.

Conditions 1 and 2 are straightforward. The third was our coup de grâce against our gap problem. Our query would locate not only users without migrated Stripe bank accounts, but also users that updated their bank account through the Airtasker platform since the migration script was last run.

That way, all we had to do was run the script once to catch the initial quarter of a million, intermittently before Phase 1 to catch updates, and then immediately afterwards for any edge case users updating during the launch. No taskers left behind.

### Striped taskers

Now we had our users, there was the matter of actually migrating them.

For each individual, we made the following three following API calls to Stripe:

* Find or create a connected account for that user.

* Add legally required (KYC) data to that account, e.g. billing address.

* Add their external bank account details.

We also needed to create our own records of the connected accounts and bank accounts in our own platform — without any financial information, of course.

For this second part of the script, we went out of our way to make an important design choice — the work performed on each user needed to be *idempotent*. That is to say, it didn’t matter how many times the script was run on a user or whether the user had been migrated already. It would repeat the same routine, consequence free.

We’ve taken a little liberty with calling the process idempotent — it’s more of a principle than a technical property. For instance, if we migrated a user twice we performed the exact same operations and create the same records. Sounds pretty idempotent. However, we also kept the old records created the first time for historical purposes. Less idempotent, but not really the point.

This idempotency, born of practical cynicism, was a blessing. After the thorough code review and stringent testing in our development environments, the big day came: running the script in production. And when we pressed go, we were met with something we anticipated: setbacks.

### Setback number one

Our plan was to test production in increments of powers of 10: 1 user, 10 users, 100 users, etc. So who was going to be the first user? Our patient zero? The first person to have their bank account in our Australian Stripe account? The author of the script, of course. It’s only fair.

Our canary was a success, and so too were 10 users and 100. During 1000 users we sat back and watched the successful migrations roll in, feeling very pleased with ourselves. Until we noticed a peculiarity. How long had this been running for now? We’d already gone through an entire cup of coffee and it still wasn’t finished.

We hurried back to our testing notes (oh yes, you bet we took precise testing notes) and looked back at the time it took for 100 users to complete. It was about 8 minutes.

After doing some quick maths, we figured the script was taking 4.6 seconds per user. Meaning that syncing the whole batch of 250,000 bank accounts was going to take 320 hours, or about 13 days. Ouch.

Ok, we had a problem. So what is the typical software engineer response when something takes a long time? Break it into little pieces and do it concurrently.

After a heated debate about the pros and cons of threads vs workers, we settled on putting the logic into a [Resque worker](https://github.com/resque/resque). After a second heated debate, we decided to make use of our existing workers in production rather than spinning up a new instance. And after yet another heated debate, we decided on a batch size of one — a single user per Resque job.

Feeling pleased with ourselves, we scurried back to our computers and refactored the script, peer-reviewed it and then tried our test again. This time it was going to work. We knew it.

### **Setback number two**

1 user. 10 users. Looking good! 100 users. Wow, it’s so fast! We’re going to be done by lunchtime.

… but wait a second.

Upon examining the logs, we discovered the reason why it was blazing fast. Stripe was returning 429s. We’d hit their rate limit for that endpoint.

Stripe doesn’t like to talk specifics about their rate limits for security reasons, but now we’d hit it they felt free to open up. Although most of their endpoints can easily support hundred of hits a second, creating a connected account is a particularly expensive operation. And so they limit it to 6 accounts a second.

If you’re a masochist and you have a pressing desire to feel bad, try rate limiting your own code. It’s a depressing situation to look at code and think to yourself ‘Oh no, much too fast. Let’s try and reduce the performance.’

Unfortunately, it’s what we had to do. After investigating the options, we settled on an existing gem: [Resque Restriction,](https://github.com/flyerhzm/resque-restriction) which was pretty much plug and play. It’s almost a shame, because if we’d chosen to write the migration using threads we would have gotten the opportunity to squander our precious time and resources making an amazing thread rate limiter, which sounds like a lot of a fun. Our PM was grateful though, at least.

On the plus side, the rate limiting gem was pretty nifty. It made us create an additional queue for our workers called the restriction queue, where jobs would be relegated if a worker attempted to pick it up when we’d already gone over our allotted jobs per second. The workers would then come round for a second pass on the restriction queue later. Pretty much like being picked last on a high school sports team. Not that I have any experience with that.

Once implemented, we settled on a limit of 5 jobs a second, safely under the Stripe limit of 6 a second, and once more tried our incremental sample size test.

### Success!

Third time was the charm. 10,000 users synced successfully, and then 100,000. We ran it on the rest over a working day and headed home with the full set migrated. The final stats were: 241764 successfully migrated users, 234,010 of which had their payouts enabled.

The migration of the these accounts was a cause for celebration. Yet there still remained the matter of the 7754 accounts that didn’t make it to a payout enabled state — about 3% of our total.

Determined to leave no bank account behind, we started a post-analysis of the migration. We found four different reasons why a user would be stuck with their payouts disabled:

1. Their DOB was invalid. Embarrassingly, we were not correctly validating date of births in our system until a few years ago, leaving a few user records with unformatted dates.

1. Their postal code was invalid.

1. Their account number or bank code were invalid. Unsurprisingly, Stripe was a lot more stringent with the numbers they accepted than our old provider, which happily accepted typos and fakes without complaint.

1. The user’s IP was in a country Stripe was obligated to not serve under United States regulations.

These were not issues we could solve by technological kung fu. The users themselves had put in invalid data and would need to update it. We ran a report on the number of tasks these 7754 unmigrated users had completed since March 2018.

728 users, about 9.4% of the taskers left behind (or 0.3% of all taskers migrated) had been active on a task in the last few months. We reached out to these users personally via email where appropriate and asked them to update their invalid details.

The rest we decided were okay to leave alone — with a caveat.

Once we migrated Stripe in Australia, a user with an unmigrated bank account would see their old, invalid bank account as empty on the platform and would not be allowed to bid on tasks. Adding a new one with invalid user details would trigger an error — all we had to do was go back and make sure these errors were explicit and user-friendly for each of the four cases.

### Our learnings

Overall, we were pleased with the way the bank account migration played out. We had three pain points: the need for concurrency, running into the Stripe rate limit and the 3% of users that couldn’t be migrated. However, with data migrations of this scale, especially when reconciling three different payment ecosystems, you expect to run into these kinds of devil-in-the-details issues. We anticipated them from the start.

This pessimism shows in our implementation of the migration script and our approach to testing it.

We had two saving graces. The first was the idempotence of the core script. This was pivotal, since it would have been easy to write the code riddled with side-effects. This decision let us run the script on production users without fear or consequence.

The second was our batching. Our rake task was implemented taking in four arguments: a minimum ID, maximum ID, limit of total records and batch size for passing into workers. This let us have fine control over the users we tested on and follow the result directly into Stripe.

It’s easy to imagine a world in which we didn’t do these two things. Where the task would need an explicit reversion if it failed on a user, or if it were written to run on the full 230,000 accounts from the get-go. It would almost be a logical decision — the task was unit tested to death and worked in development environment like a charm. Reading the code in the PR, it was hard to imagine what could go wrong.

It’s only experience that allows you to think: well, this is beautiful code. Probably won’t work anyway.

Follow us to [Part 5](https://medium.com/@tpagram/to-our-credit-1cea3c41dfbb), where we tackle the final piece of the puzzle by migrating half-a-million credit cards.
