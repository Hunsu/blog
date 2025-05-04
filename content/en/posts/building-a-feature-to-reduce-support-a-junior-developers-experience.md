---
title: 'Building a Feature to Reduce Support: A Junior Developer’s Experience'
description: 'A look back at one of my first big projects at the PSP company: building a feature solo (with the guidance of an architect) to fix a pain point in card registration. How it reduced support tickets, improved the product, and surprisingly boosted revenue.'
author: Rabah M
toc: false
date: 2019-05-21
css: "/css/styles.css"
tags: ['software-engineering']
---

# A Special Project Outside the Sprint

When I was working at the PSP company, one day my manager asked me if I wanted to work on a feature independently. That meant the feature wouldn’t follow the usual team process—you know, the whole routine of adding it to the sprint, estimating it, breaking it down into subtasks, and so on.

I understood pretty quickly that it was a big feature, but they didn’t want to disrupt the team’s goals. I think I got picked because I’d already shown interest in handling support issues (which this feature aimed to solve), and I was always up for taking initiative and tackling challenges.

The feature itself was pushed by the company president, so it got prioritized outside the normal roadmap. The expected impact was huge: cutting support tickets by 30% and generating an estimated €54k per month in revenue. Naturally, I was excited to take it on and saw it as a great chance to learn and grow.

# What’s Card Registration Anyway?

Let me explain what the feature was all about. You’ve probably seen it when paying online—a little checkbox saying something like “save this card for future payments.” Or maybe you’re just adding a payment method to your account without buying anything, or starting a subscription (because let’s face it, everything’s a subscription now, with that classic first month free deal).

When a merchant wants to save your card, they ask the PSP (payment service provider) to generate a token that can be reused for future payments. But the PSP doesn’t just store the card blindly; it needs to check that the card’s valid. Back then, the way to do that was to try charging €1 and let the authorization expire (some of the better PSPs would even cancel it immediately to avoid holding funds unnecessarily).

Now, if the merchant was doing a payment and registration together, and something went wrong, they’d get a declined transaction with full details explaining why it failed. Our product was really solid in that area, and merchants could easily figure things out on their own.

<div class="image-gallery">
  <h3>Examples of how transaction details are shown—Not real screen</h3>
<div class="image-wrapper">
<a href="/images/transaction-details.png">
  <img src="/images/transaction-details.png" style="width:300px">
</a>

<a href="/images/error-details.png">
  <img src="/images/error-details.png" style="width:300px">
</a>
</div>
</div>

But when it was just registering the card—without an actual payment—things got trickier. Since no real transaction was created (even though we technically tried a €1 authorization), no transaction record is shown in our dashboard. If the operation failed for any reason—invalid card, expired, wrong CVV, fraud rules, authorization failure, you name it—the merchant only got a notification with an error code. They didn’t have a nice dashboard view in their side to see what happened. And since we didn’t provide one for card registration failures, their only option was… you guessed it: contacting support.

I had been handling support for a few months by then, and honestly, I hadn’t noticed that many tickets about it. So at first, I didn’t get why solving this would reduce tickets by 30%. But it turned out that Level 1 and Level 2 support were handling most of them without escalating, usually replying with a generic email listing possible reasons—leading to a lot of back-and-forth with merchants. Worse, sometimes these generic replies could miss actual issues, like when a failure was caused by a bug on our end.

That’s when I learned something surprising: the president of the company read every support email. He even personally replied to some of them—sometimes without a signature, and at odd hours like 2am (he worked from another timezone and put in really long hours). So he was deeply aware of the pain this caused and really pushed for tackling it.

# Building the Solution
## Why Not Just Save the €1 Transaction?

As you might’ve guessed, the simplest solution would’ve been to just save that €1 transaction and handle card registration like a regular €1 payment that we cancel right after. In fact, from a technical perspective, it almost looked like a one-line change—maybe just tweaking the shouldSaveTransaction method to return true in that case. Easy, right?

But of course, things are never that simple. You’ve got to think about the side effects. [I learned that the hard way](my-journey-breaking-production-at-a-psp-company.md).

Merchants weren’t expecting to see a €1 transaction in their dashboard. They didn’t ask to charge the card—they asked to register it. If we started showing these dummy €1 transactions, it could easily confuse them. Why are they seeing payments for €1 when nobody actually paid €1? Some merchants might even call their own support teams, thinking it’s an error. We’d basically be pushing our internal implementation detail onto them, and that’s never a good idea.

That’s why the direction we chose was to introduce a new transaction type for card registration. By having it as its own type, we could store and display these operations separately from regular payments, keeping things clear for merchants. It also opened the door to other possibilities—like distinguishing it from other transactions for reporting purposes or even charging different fees for it (something we’ll get into later).

## Building the Solution

I worked closely with the architect, who walked me through the solution. At that point, I was already pretty familiar with our payment engine—it was built with a modular architecture, where different services each handled one task, passing and enriching the context along the way. On top of it all was an orchestrator deciding which services to trigger and in what order.

The solution was pretty elegant: we’d add a new service whose job was simply to adjust the transaction type in the context so that it gets saved as a card registration, rather than as a regular debit. Then we’d update the orchestrator to call this new service, followed by the existing service that saves transactions. Of course, everything was protected by a feature flag—so with the flag off, the system would behave exactly as before. And since this change was done at a central point in the flow, it meant every integration (whether web, API, or even batch file uploads) would benefit from it automatically.

Technically, these were the only changes needed. But like every dev knows, once you touch something, you uncover hidden surprises. Some tests started failing, and I had to fix them—sometimes paying off technical debt that had been lingering. There were also side effects to think through. For example, now that we were saving a transaction, the notification service (which is triggered next in the chain) would include more details in the notifications sent to merchants. Even though we had always told merchants not to hardcode fields in notifications, some inevitably had, and this change could break their flows. Since this was such an important feature, we knew any breakage would be blamed on us, so we had to be really careful and account for these edge cases.

I made sure everything worked end to end, duplicating all our E2E tests to run with the feature flag enabled and making the necessary fixes. And after about a month, the feature was ready to be rolled out.

## Rolling It Out

All the dev work was done, everything deployed, but the feature flag was still off—I was just waiting for the green light to enable it. But if you’ve worked in medium or big companies, you know how it goes: sometimes they ask for an urgent feature to be done in a month, you deliver in a month, and then… it sits there for another few months before anyone approves switching it on (or worse, it gets forgotten or deprioritized).

In our case, we had to finalize the documentation, which was handled by another team. Luckily, the person in charge had solid knowledge of how everything worked, so that part went smoothly. But we also had to communicate about it internally and externally, especially since we sold our product as a white label solution to banks. That’s where things got tricky: getting approvals from banks meant going through multiple levels of decision-makers, many of whom didn’t really understand the technical side of what we were doing.

Thankfully, we didn’t have to wait for their approval to proceed for our direct clients. So we started enabling the feature for them first, leaving the white-label clients for later. And honestly? It went super smoothly. No major bugs, no regressions. From the start of the project to full rollout, it took about three months, with around two months to have it fully enabled everywhere.

We didn’t really get solid reports on the impacts at first—our company wasn’t the best at tracking post-release outcomes, something I think is really important. You shouldn’t just ship and move on; you need to follow up and see what changed.

But I did get clues. A few months later, I started getting calls from colleagues—people from sales, the billing team—telling me some clients had noticed big increases in their invoices, like 20–30% more than usual. We’d expected the feature to generate extra revenue since we charge fees per transaction, but that kind of jump wasn’t typical… unless a client was only using us for card registration and not for actual payments. And that’s exactly what happened: some clients were using our platform just to register cards and then processing payments elsewhere. Registering a card still has value—it confirms the customer’s card is valid, that they’re an adult, and that they’re interested in the product.

Apart from those billing surprises, I never really heard of other issues.

# Conclusion

Looking back, this feature was one of the best learning experiences I had early in my career. I got to work more independently than usual, but still with guidance from the architect, and I felt trusted to deliver something important. Even if it wasn’t following the usual team process, I realized that sometimes priorities shift fast in a company, and you need to adapt and move outside the standard roadmap to get things done.

One thing that really stood out to me: if we hadn’t looked closely at what was happening in support, we wouldn’t have seen either the pain or the opportunity behind it. At first it looked like just a way to reduce tickets, but it turned out to be a chance to actually generate more revenue. Sometimes cutting costs—like reducing support requests—can lead to new ways to earn money too. Support can be a hidden source of opportunities if you listen closely.

The rollout went smoothly, but I also saw how even a technically simple change could have unexpected impacts—on invoices, on clients’ behavior, on how we communicate the product. It wasn’t just about coding and deploying; it involved collaboration with other teams, navigating approvals, and thinking ahead about how clients would react and use the feature.