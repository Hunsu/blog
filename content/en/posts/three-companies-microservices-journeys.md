---
title: 'Three Companies, Three Microservices Journeys'
description: "What happens when three different companies decide to go the microservices route? In this post, I share my personal experience with each—what worked, what didn't, and the tradeoffs that came with every decision."
author: Rabah M
toc: false
date: 2023-01-11
tags: ['software-engineering']
---

# Introduction: Three Journeys, One Theme

If you’ve worked as a software engineer in the past ten years, chances are you’ve spent time on a monolithic application—and maybe witnessed that pivotal moment when the team decided to embrace the magic of microservices and start migrating features.

In this post, I’ll share my experience across three different companies, each going through that journey with very different outcomes. One experience was chaotic and painful, another was functional but flawed, and the last one was smooth and empowering. Rather than diving into technical details of “how to migrate a monolith” (there are already dozens of great articles for that), this post focuses on the real-life organizational and developer experience behind those migrations.

Each section will give a snapshot of the company context, why the decision to go microservices was made, and how it actually played out—both the good and the bad. Hopefully, it will give you a grounded view of what to expect (or avoid) if you find yourself on a similar path.

# The First Experience: A Payment Service Provider Embracing Microservices
## Why We Migrated to Microservices

I worked at a payment service provider (PSP) company where the entire product—frontend and backend—was built as a monolith. For performance reasons, some parts could be deployed separately and accessed via remote EJBs, but scaling horizontally wasn’t exactly straightforward.

Then came a shift: new regulations required strong customer authentication (SCA) on all online payments, along with the rollout of 3D Secure 2 (3DS2).

> 3D Secure (3DS) is an authentication protocol designed to reduce fraud in online credit and debit card transactions. The new version, 3DS2, introduced a smoother user experience, better integration with mobile apps, and more flexibility for merchants, while keeping fraud prevention strong.

Previously, few merchants even enabled 3DS—they preferred to take the fraud risk rather than lose customers at checkout. But with these regulations, 3DS wasn’t optional anymore.

Our PSP had a built-in 3DS solution and even licensed it to other providers who hadn’t developed their own. Suddenly, this product was poised to see a massive increase in traffic. More transactions meant more revenue—but also more load, more compliance requirements, and much tighter performance guarantees.

At the same time, development inside the monolith was painfully slow. It could take 10 to 15 minutes just to start the app locally, and the developer experience was poor overall. The tech stack was outdated, upgrades were rare and painful, and since many teams were working in the same codebase, coordination overhead was huge. Starting fresh with a microservice offered a clean break: a faster dev loop, a modern tech stack, fewer dependencies on unrelated teams, and the ability to move at our own pace. 

We had less than two years to implement 3DS2 and certify with major authentication networks like Visa, Mastercard, and others. In the world of banking, two years for such a major shift is tight.

Under these constraints, making the authentication service its own microservice wasn’t just a trendy decision—it was practical.

## Building the Service: Successes and Frictions

I joined the team just as this microservice project was getting off the ground. But before I arrived, the team had already spent weeks in meetings trying to convince other teams that going with a microservice was key to success.

And they had a point—it wasn’t just about writing code. Introducing a microservice affected several other teams: the deployment team, the infrastructure team, the release management team. Every new application meant extra work for these groups: another system to deploy, monitor, troubleshoot, and keep up-to-date.

But the benefits were obvious, at least from a technical standpoint. This new service let us use modern tech. We built it with Spring Boot and Reactor, with up-to-date dependencies, and even integrated tools to automate upgrades (compared to the monolith, where upgrading something like Hibernate could take months). We packaged it as a Docker image, set up Kibana for monitoring and alerting, and had an isolated, easily testable HTTP API. We were finally working with the latest stack instead of dragging technical debt along.

But it wasn’t all great news. The monolith depended on this new service for every transaction—it became a critical point of failure. If the service was down or misconfigured, every payment failed with an internal error. I remember nights wasted because the service hadn’t been properly deployed to the integration environment, killing end-to-end tests.

Developer experience in the monolith also took a hit. Now you had to start up another service just to run things locally. 

The biggest issue, though, was ownership. In this company, infrastructure, deployment, and release processes were managed by separate teams. Every configuration change—like adding a new endpoint or updating a proxy—had to go through them. That created a lot of back-and-forth, sometimes over small mistakes like pointing Visa requests to the Mastercard endpoint by accident.

In this kind of structure, spinning up multiple microservices wasn’t ideal. 

# The Second Experience: When Microservices Multiply Without Control
## Why Microservices: A Seemingly Logical Step
Fast forward to 2021, I was working at a major e-commerce company. Like many big players at that time, they already had some microservices in place—nothing unusual.

My team was responsible for two microservices. The first one handled tax calculations based on ordered items. Except… it didn’t really calculate anything. It just forwarded requests to an external tax service that did all the business logic. The second microservice was in charge of compliance, but again, it simply passed information through to another external compliance provider. Both services were essentially thin adapters.

A new project came up: integrate a new external compliance partner. The proposed solution? Build two more microservices:

- One adapter microservice to talk to the new provider.
- One router microservice to decide which provider to call.

It felt like we were adding services just for the sake of adding services. We weren’t solving new problems; we were creating layers of indirection. But as developers, we weren’t in a position to challenge the decision—it was already made at a higher level.

## The Reality: Complexity Without Control

To develop a new microservice, we started from a company-provided repo template—but it required a lot of manual tweaking to get something usable. All new services were created on a year-old Spring Boot version, since no one maintained or updated the template or dependencies. There wasn’t even a shared library across microservices; each had its own duplicated boilerplate.

Each microservice lived in its own Git repo, so as a developer, you had to juggle multiple IntelliJ projects just to work across them. We didn’t have any feature flagging solution in place, but frankly, it wouldn’t have made a difference: the services were only pass-through adapters, adding little value.

Like in my previous experience, there were dedicated teams for releases, deployment, and infrastructure. But this time, we were responsible for setting up the CI ourselves. I was stunned when I saw the Jenkins setup—over 500 jobs for all kinds of build, deploy, and integration tasks, managing 100+ apps.

Each microservice had its own dedicated database, but most were used only for logging requests and responses. Accessing logs meant SSH’ing into one of a dozen servers and grepping around, hoping you’d picked the right machine and log file. It was already painful for the two microservices we maintained—adding two more would multiply the chaos.

We used asynchronous messaging in some parts, but creating RabbitMQ queues and exchanges was a manual task for the infrastructure team. Database configuration? Also handled by another team. Even choosing which port our service should use meant sending an email and waiting for approval.

I couldn’t believe it when I learned our adapter microservice had 2GB of RAM allocated, with 4 instances running—despite being non-critical and spending most of their time idle. Monitoring and alerting were practically nonexistent; if something went wrong, we found out the hard way.

In the end, we were trapped in a situation where the supposed benefits of microservices—autonomy, agility, scalability—were never realized. We inherited the complexity without the ownership, and each new service felt like adding weight to a machine we couldn’t control.

# The Third Experience: Microservices With Purpose and Discipline
## Why Microservices: Growth That Drives Structure

At Malt, a freelancer marketplace and scale-up, microservices—or “apps,” as we call them—emerged naturally. Each app corresponds to a specific business domain and shares the same PostgreSQL and MongoDB instances, with clear schema separation. The motivation for splitting apps is usually team growth or clearer domain boundaries, not just architectural dogma. Apps are lightweight, start in under a minute, and aren't resource-intensive. 

Since I joined, several apps were created from a single one as the team scaled: we started with the project app, handling full project lifecycles, and later broke out project-payment and then invoicing as dedicated teams formed.
Every split happened for a clear, practical reason—avoiding the trap of microservices for microservices’ sake.

## The Setup: Automation, Ownership, and a Smoother Dev Experience

At Malt, spinning up a new app is quick and painless—because it has to be. Scaling a company without automating the repetitive parts just doesn’t work. We have internal tooling that lets you bootstrap a Spring Boot-based app in minutes. It sets up everything: database access, RabbitMQ, monitoring integrations, and shared starter modules.

CI/CD pipelines are created just by adding a config file. Ownership is clear—each team is responsible for a set of apps, and getting added to one means you’ll receive all monitoring alerts in your Slack channel.

You can deploy a production-ready app within minutes.

For observability and cost control, we rely on DataDog dashboards to monitor performance and usage. Teams are responsible for keeping their apps efficient—but if something slips, internal tooling will let you know when you’re burning too much.

The developer experience is also thoughtfully managed. There’s minimal friction to working on a single app. You don’t need to start every service locally—remote calls fall back to mock responses if the target app is unavailable. All apps live in a monorepo, so we work in a shared codebase.

It’s not perfect. Because all apps rely on shared starter modules, smaller or atypical use cases can inherit unnecessary dependencies. I worked on a simple Slack bot that didn’t even expose HTTP endpoints, but I still had to include the web starter, Mongo, Redis, and other unused pieces. These edge cases aside, the setup works extremely well for the majority of services.

Having many apps also comes with its own challenges—most notably the increase in web service calls. At Malt, we use HTTP calls when an app needs data from another business domain handled by a different app. That means you need to be thoughtful about service boundaries and avoid creating call chains that slow everything down.

For instance, at one point, we discovered that one of the apps my team was managing was receiving 40 million calls per month for a single endpoint—and, in turn, making another 40 million calls to another app. This happened because the frontend was polling frequently, and the backend app handling those requests had been migrated to use HTTP calls to another app instead of handling things locally. That change had a cascading effect. We fixed it by introducing a local cache and using async messaging to invalidate the cache when needed, which brought the call volume down to just 50,000 per month.

In theory, we could let each app manage its own database. But to avoid duplication and maintain consistency, we use shared databases with separate schemas. For large or shared domain models, we sometimes rely on a shared code repository—but even that is scrutinized carefully before being adopted.

# Conclusion: Same Buzzword, Very Different Outcomes

After going through these three experiences, one thing is clear—just saying “we’re doing microservices” doesn’t mean much on its own. The difference between a good setup and a painful one comes down to ownership, tooling, and having the right mindset.

In the first company, microservices felt like the right move, but we were stuck depending on too many other teams, which made the whole thing fragile. The second company took it to the extreme—new services everywhere, no real reason, and a lot of overhead without clear benefits. At Malt, it’s been the most balanced: apps are created when needed, automation helps a lot, and teams actually own what they build.

At the end of the day, microservices can be great—but only if they make your life easier, not harder. If you’re adding complexity without the tooling or the team structure to handle it, you’re probably better off sticking to something simpler.