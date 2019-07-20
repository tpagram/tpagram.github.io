---
layout: post
title: Stripe in the UK
subtitle: How we earned our Stripes — Part 2
description: This is the second article in a series of blog posts about the transition towards Stripe as Airtasker’s main payment provider.
category: Technology
excerpt_separator: <!--more-->
comments: true
---

![](https://cdn-images-1.medium.com/max/6002/1*Ot9Gb-fcfM4_JFhgIPcY5A.png){:width="100%"}

[*Originally posted on Medium.*](https://medium.com/@tpagram/stripe-in-the-uk-dec797a7585b)

*This is the second article in a series of blog posts about our transition towards Stripe as Airtasker’s main payment provider.*

When you start from ground zero, switching payment providers is an intimidating project. Airtasker hosts around 4000 tasks a day, which adds up to a fair amount of dollarydoos moving through our payment system from poster to tasker. That’s not something you want to mess with. It’s as critical to Airtasker as you can get.
<!--more-->

When we sat there at the beginning, trying to create a list of all the things that needed to be done was overwhelming. We had to:

* Learn about the way Stripe organises its internal account structure and its API.

* Figure out what parts of the codebase needed to be rewritten.

* Design the architecture of our new implementation.

* Figure out the differences between Stripe and our old provider.

* Migrate all of the existing credit cards and bank accounts.

* Develop a list of cutover edge cases to consider.

* Think about how to mitigate risk.

* …the list just grew and grew.

That’s too much to fit in any person’s head. We needed to chunk up the project and take it one step at a time. Fortunately, we were blessed with just what we needed. Airtasker had another exciting project being developed along side — **we were launching in London!**

This was the perfect recipe for testing out Stripe. We could start fresh on a smaller subset of users without the risk or intricacies of migrating our Australian users. This would essentially divide the project into three parts:

1. First, develop a functional version of the Stripe payments engine ready for UK launch.
 2. Test our implementation on the UK users, ironing out any issues and continuing to build the engine to feature parity with our old Australian implementation.
 3. Finally, migrate our two million Australian users to Stripe.

### Stripe MVP

We decided that a functional version of our Stripe implementation would be ready for the UK launch in February. While this goal was more manageable than migrating Australia, it was still a meaty piece of work.

From a high level, the product requirements were simple. A UK user, while going through the usual Airtasker flow of posting tasks, bidding, assigning bids, escrowing funds and releasing payments, would need to perform actions in Stripe rather than through our old provider.

We identified four main tasks to accomplish before launch:

1. **Prune our codebase**. Identify every point of our codebase that dealt with our old provider. For each of these, insert code that conditionally works with either Stripe or our old provider, depending on whether the user is from the UK or Australia.
 2. **Plan our architecture.** Decide on an internal Stripe architecture to support our flow. A marketplace with escrow-like behaviour is not a typical Stripe customer — we needed to build something bespoke.
 3. **Develop our payments engine.** Architect and develop a new Rails payments engine to implement the core payment flows required for launch.
 4. **Analyse the impact.** Figure out how to adequately report on the financials so our Financial Controller wouldn’t strangle us.

We needed to prune, plan, develop and analyse. We’ll go through each in detail.

## **Pruning our codebase**

According to [CLOC](https://github.com/AlDanial/cloc), the Airtasker API alone has over 800,000 lines of code. Our existing payments engine was tightly coupled to this codebase like vines wrapped around the trunk of a tree. After years of growth, it was difficult to see what was providing more structure — the trunk or the vines.

Like real vines, it was easier to hack them off at the base than to pry free every little branch. Go too far down, though, and you end up removing large chunks of the tree itself.

Let’s abandon metaphor and talk plainly: no matter what, we were going to end up rewriting parts of our core flow. There were too few sections of our codebase you could point to and say ‘everything encapsulated here is purely payments’. Sometimes a section would be mostly payments with a few important side effects on the main code base. Other times a non-payments flow would decide to touch a payment record.

These records in our database were a particular pain point — it was wishful thinking that we could sever the parts of the codebase with API calls to our provider, replace it with Stripe code that produced the same output and call it a day.

The database structure for our payments tables were too coupled to our old provider — the difference in Stripe’s internal architecture, API structure and naming conventions meant the tables were not fit for purpose.

This spread the impact of the payments engine on our codebase further — a hypothetical past developer may have the wisdom to abstract around a payments API call, but not for simply checking a field on a payment record in the database. And now those records were bunk.

There was no getting around it. We had to sit down, go through the codebase, and pick points to sever the code. To do this, we wrote a simple module called a PaymentProviderSwitcher. It took a user, and returned whether they should use the old payments engine or the new. The majority of the time it was as simple as whether the user was in Australia or the UK.

We identified 46 points in our code to stick these conditional switch statements. Everything around those statements would stay. Anything inside would be rewritten. Simple as that.

With this done, the project became a lot less daunting. We had a finite list of features to implement and code to write. All we had to do was figure out how to do them.

## **Planning our architecture**

Before we talk about Stripe’s architecture, let’s briefly talk about our own needs.

Since you’re reading this, I assume you’re familiar with Airtasker. But just in case you stumbled across this blog while blind drunk, let me recap for you. Airtasker is a marketplace for services, both online and in the local community. Posters put up job descriptions called tasks and taskers bid to complete them.

When a poster assigns a tasker, we escrow the money agreed to for that task and hold it until both parties conclude the work is complete. Once this happens, we release the funds to the tasker’s bank account. Minus, of course, a service fee — a deduction off the task price we take to maintain the website, manage insurance cover and pay our salaries and taxes. To complicate matters, we also have both a coupon and credit system — where we partially fund the value of some tasks out of our own pockets.

### Stripe’s API

Most of Stripe’s clients do not have bespoke needs. Their goal is to accept money from their customers and transfer it to themselves. They might be an e-commerce business needing a [checkout cart](https://stripe.com/checkout) or a subscription service needing [recurring billing](https://stripe.com/docs/billing/quickstart). Something more complex like a marketplace, that needs to accept money and pay out to multiple parties, is catered for by [Stripe Connect](https://stripe.com/docs/connect).

![Stripe Connect. Source: [https://stripe.com/docs/connect](https://stripe.com/docs/connect)](https://cdn-images-1.medium.com/max/3600/1*oIHty9n6BMqO8fSwluUI5A.png)*Stripe Connect. Source: [https://stripe.com/docs/connect](https://stripe.com/docs/connect)*

The high-level architecture of Stripe Connect is essentially a directed graph flowing from payer to recipient — a structure very familiar to software engineers. In the middle is a big fat node called the **platform account**, which all money is routed through.

The platform account represents the account of the business itself — in this case, Airtasker. Money flows in and out of the platform account in two atomic actions. The customer’s money is taken from their card into the platform account in a **charge. **It is then broken up and moved into **connected accounts* ***using*** *transfers*.***

Connected accounts are at the heart of Stripe Connect. They represent each of the parties you wish to pay out to. Say, as a hypothetical example, you had a business drop-shipping bouquets of flowers. A customer pays you $50 for a bunch of white lilies, which is charged into your platform account. You then transfer $30 to the wholesaler’s connect account for the flowers, $15 to the shipping company’s connect account and then leave $5 in the platform account as your take.

The nursery and shipping company have registered a Stripe connect account and can log in to their own Stripe dashboards. In there, they can set a payout schedule to get their dosh.

You can begin to see how this might work as an Airtasker architecture. On a task, the poster’s card is charged, moved into our platform account, then transferred into the tasker’s connected account.

One problem: we can’t make our workers all sign up to Stripe connected accounts to manage their earnings. Imagine a valued user wanting to get paid for doing a $60 pruning job on a Sunday. Do they really want to go through another sign-up flow, unbranded with Airtasker, and navigate the choices of when and how to pay that money out? Of course not. Our customers should be completely oblivious to what payment provider we’re using — the experience should be seamless and joyful, with little mental effort.

So yet again we had to forgo the average. We needed to move away from what Stripe call **standard** or **express** connect accounts, and strike out into the unknown territory of [**custom** connect accounts](https://stripe.com/docs/connect/custom-accounts).

Unlike standard accounts, custom accounts are completely invisible to users. Even though somewhere they actually do have a little dashboard with information about their money (think of the UI for a bank account), they never access it. Instead, they interact with it purely through the Airtasker platform, and we handle their actions with the appropriate API calls.

Perfect. With this newfound knowledge, we sat down with Stripe and devised our first architectural plan for the Airtasker task flow.

### The Airtasker Flow

Richard, a 63-year-old pensioner, needs the wattle tree in his backyard cut down. He loves it but the council won’t budge, so he’s finally acquiesced. He can’t do it himself, what with his knee replacement surgery, so he decides to try out that company that keeps handing out $10 coupons at the train station.

He creates an account and posts a task for $80, which quickly collects bids. Sara, a part-time gardener, gets his pick. Someone called Brady said he’d do it for $40, but he chose Sara because she has a warm smile and reminds him of his granddaughter.

When he accepts her bid, he’s prompted for his credit card. When he enters it, we have our first contact with the payments engine — his details are entered as a **customer** on our platform account.

His card is then charged and the funds, minus the $10 dollars from the coupon, move to our platform account. This is a core part of of the Airtasker flow and a bit of a headache in terms of payments infrastructure. In the interim before Sara completes the task and her funds are released, Richard’s money is held in escrow. Where that money sat was a point of contention.

### Plan A

![Our first plan.](https://cdn-images-1.medium.com/max/6000/1*TKPMElqRxcTDfqDmGsQczw.png)*Our first plan.*

Our first plan was to keep the escrowed money in our platform account. When Sara cuts down the tree and Richard releases the funds, we would do the following:

1. Move the escrowed $80, minus the Airtasker Fee, from our platform account to Sara’s connected account.

1. Wait for Sara’s connected account to payout on an automatic daily payout schedule.

The advantage of holding the money in the platform account was the simplicity. We could easily distinguish between credit card, coupon and Airtasker Credit amounts. Refunding was a breeze and payouts were handled for us.

There were exactly two transactions per task: charge the card upon assignment, then transfer the money to the connected account upon completion. Simple.

Unfortunately, we quickly learned that escrow has a precise legal definition for Stripe, and it supports only escrow-like behaviour with certain specifications. So we had to pivot our implementation.

### **Plan B**

![Our final plan.](https://cdn-images-1.medium.com/max/6000/1*VaWQLY2SL63O3v76mf5p3w.png)*Our final plan.*

To meet legal requirements, the money held secure in Airtasker Pay had to be kept in the tasker’s connected account. This had a few consequences in terms of the complexity.

Instead of two transactions per task, we now had at worst-case five: one charge, three transfers and a payout.

For Richard, this meant:

1. On task assignment, charge Richard’s card for $70 and move the funds into our platform account.

1. Immediately transfer those funds, minus the Airtasker fee, to Sara’s connected account.

1. Transfer the $10 for the coupon, again minus the Airtasker fee, to Sara’s connected account. If Richard has used Airtasker Credit as well, this would be another transfer.

1. When Richard releases the funds, schedule a manual payout on Sara’s connected account equal to the task’s value.

Plan B was ultimately more complicated, but it’s what we went with. You can’t argue the law on the basis of technical hardship.

## **Developing our payments engine**

We’d figured out the 46 parts of our codebase we had to rewrite and the internal Stripe transactions that would make our system work. Now we had to go and actually build it.

There were a couple of big decision points to consider:

### **Encapsulation**

The big problem with our old payments engine was how tightly coupled it was to the rest of the codebase. Given that it was 2017, the obvious question to ask was whether the code responsible for payments should be a microservice.

There’s a couple of things to unpack here. First of all, to address the elephant in the room: the Airtasker API is a Rails monolith. We haven’t really embraced microservices — partly through intention, but mostly through a lack of need.

We were scaling like a beast (and still are) which meant the microservice question had finally entered into the forefront of our minds. Was this an opportunity to dip our toes in the water?

Ultimately, the answer was no. The payments infrastructure was absolutely too pivotal to us to risk an experiment. We had two hypotheses: 1) that we had the resources to create a solid microservice and 2) that a microservice architecture was a good choice in the first place. This was better validated with a smaller-scale project.

However, the decision to go against microservices was no excuse to repeat the mistakes of the past. We couldn’t let the new payments engine and our codebase morph together once again to form twisted vines and branches, stunting our growth.

Fortunately, a microservice architecture is not a necessary condition for decoupling a codebase — it just conveniently forces you to. There was no reason we couldn’t encapsulate the logic of our payments engine as far from our codebase as possible without intending to make it a service. Encapsulation is essential for good design regardless of the choice of a microservice architecture. Plus, if we ever did want to go down that route? It’s as simple as pulling it out.

And that’s what we did. We created an **engine **—a self-contained section of our Rails repo that shared the same database, but was written with absolutely no context of the rest of the codebase. The engine would have a bunch of payments-related Rails models, like Payment, StripeAccount, PaymentMethod and so forth, which we would try as hard as possible to use purely within the engine. Models from the rest of the codebase, like User or Task, were unwelcome within the engine.

Code that dealt directly with the Airtasker flow, like escrowing a task, went in the main codebase, where it would call the payments engine as a service. The engine itself was nothing more complicated than an interface over atomic Stripe actions — charging, transferring, refunding, adding credit cards and bank accounts, and so forth.

**Non-polymorphic providers**

We started off [part 1](http://link) of this series by saying we never anticipated moving away from our old payment provider. And we’ve all been in enough relationships to know you may start off by thinking ‘this time I’ve found the one’, but there’s a good chance two years later you’ll wake up and look at them and wonder what you even saw in them in the first place.

Post-implementation, we’re certain Stripe really is the one. We’re deeply in love, getting engaged and there’s probably kids on the horizon.

Pre-implementation, there was an open-ended question of whether we should generalise over the concept of a provider in our codebase. In fact, there’s still remnants of this line of thinking in our engine— we have a model called Provider, from which StripeProvider is the sole inheritor.

This desire came from a technical side rather than the business — engineers love to generalise. We can sit and dream about an alternative reality where, years ago with our first provider, we had engineered this brilliant interface called Provider, which stubbed a few abstract methods like charge_card and payout_money that were core to the payments workflow. Then, moving to Stripe would have been as simple as creating a subclass called StripeProvider to join the OldProvider and implementing those core methods.

This is a fever dream. Trying to orchestrate it was a wild goose chase and a classic example of over-engineering that we quickly abandoned. Payment providers have internal architectures and APIs that are wildly different — trying to fit a layer of abstraction over a singular payment provider was going to couple that abstraction layer to the provider without fail, making it redundant if we ever actually tried to add a new one. It would also add excruciating structure to an already complicated implementation.

Trying to create unnecessary layers of abstraction is the cliché example of over-engineering. It happens to all of us, from the beginner programmer who just learned about object-oriented design and wants to make everything they touch as generic as possible, to a senior engineer trying to architect a complex piece of software.

The result of this is that the structure of our payments engine is tightly coupled to our Stripe implementation. If you want to change it, you’ve got to write more code. Get over it.

## **Launch**

Come late February, it was time to roll out in London. At this point the Stripe implementation was functional, yet far from feature parity with Australia. Refunds were being handled manually by support agents in the support dashboard, we didn’t have payment history or receipts, and our plan B for our account infrastructure was still work in progress.

We pressed deploy with bated breath. As anyone producing software will tell you, you can architect and test something to death, but you won’t truly know if it works until it’s out in the wild.

It was successful. More than successful, in fact — we’d thoroughly proven our MVP and were willing to commit more resources to bringing Stripe into Australia, as we will discuss in [Part 3](https://medium.com/@tpagram/stripe-down-under-9fe3ca7aa8ee) of our series
