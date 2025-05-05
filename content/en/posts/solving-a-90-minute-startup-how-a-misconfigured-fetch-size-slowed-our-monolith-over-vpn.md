---
title: 'Solving a 90-Minute Startup: How a Misconfigured Fetch Size Slowed Our Monolith Over VPN'
description: 'When our monolith app took over 90 minutes to start over VPN, I traced the problem to an overlooked database fetch size setting. This post walks through the investigation, the unexpected culprit, and how a small config change cut startup time by over an hour.'
author: Rabah M
toc: false
date: 2021-11-13
tags: ['software-engineering']
---
I landed my first remote job just after COVID, at a time when companies were still hiring a lot and pretty relaxed about remote work. I was the first remote employee in the company, and that status mattered more than I expected. My contract took a long time to finalize—because, like in many big companies, everything had to follow a process, every box checked, every legal requirement fulfilled.

Being the first remote employee wasn’t just a bureaucratic milestone; it turned out to be key to a technical problem I ran into. See, we were working on a big monolith app that connects to a shared Oracle database. When running the app in the office, startup took about 10 minutes. But running it from home, over VPN? It ballooned to 1 hour and 30 minutes.

This is the story of how I investigated, traced, and fixed that problem.

# The 10-Minute Startup (In-Office)
I started by spending a week onsite for onboarding. I got my hardware, met my colleagues, and made sure I had access to everything I’d need. They walked me through the configuration and how to start the app. There were a lot of manual tasks and installs—annoying, but manageable.

That process told me a lot about their engineering culture. Good companies tend to automate everything: you’d bootstrap a machine with one command. Bad ones? They give you outdated documentation and expect you to figure it out. This company was somewhere in between.

Their main product was a big monolith that every team still depended on, even though each team also owned some microservices. You couldn’t avoid running the monolith—it was the entry point for almost everything. In the office, with everything ready, it would start in about 8 to 10 minutes, maybe 15 including the build. That seemed reasonable at first.

# Discovering the Slow Startup at Home

I started working from home on my second week, ready to tackle my first tickets. The changes I had to make were in the monolith, so I went ahead and tried to start it locally, just like I had done at the office. At first, it failed right away—it turned out I needed to connect to the corporate VPN for it to work. Once connected, I tried again.

The monolith uses an Oracle database, shared by everyone. I already knew the database was getting pretty big with test data and old entries, so I figured the startup might be a bit slower than at the office. But after 10 minutes of waiting, I noticed something odd: the logs had stopped, but I couldn’t access any page in the app. The server still wasn’t responding.

I restarted it, checked configurations, tried different things—nothing changed. Then, by pure chance, I left it running while I went for lunch. When I came back over an hour later, I saw new logs: the startup was finally complete. I was stunned—it had taken 1 hour and 30 minutes to start, just because I was working remotely over VPN.

I asked my colleagues about it. Their answer? “Yeah, it’s a known thing. We don’t know exactly why, it’s just slow from home.” I asked how they worked around it, since many had 2 days a week of work-from-home or worked from their agency’s offices.

The answers varied. Some avoided touching the monolith on WFH days, focusing on meetings or tasks that didn’t require running it locally. Others wrote code guided by unit tests and only tested the full app once back in the office. The consultants had a special setup: they each got a remote development machine where they could deploy the app like a personal integration environment. But that wasn’t provided to full-time employees like me.

I quickly realized that unless I got one of those remote machines—which would likely take a long time to approve—I was stuck with a terrible developer experience. And waiting 1h30 every time I wanted to restart the app wasn’t going to cut it.

# Investigating the Root Cause

Like I said, the developer experience wasn’t great, and I didn’t want to accept it as “just the way it is.” I really wanted to understand why it was so slow from home. So I decided to invest some time—even my personal time—because it felt like a fun technical challenge, and I couldn’t resist it.

## First Hypothesis: Network Timeouts

The first thing that came to mind was that the app might be trying to connect to some internal services that weren’t reachable from home, and maybe there was a big timeout kicking in.

That hypothesis turned out to be partially true. I discovered that the app was trying to resolve the IP address of localhost, but instead of using localhost, it was using the machine’s hostname. And with the VPN connected, DNS resolution for the hostname was timing out.

Easy fix: I just added an entry to /etc/hosts.

Let’s say my machine was called CORP12345, I added this line:

```
127.0.0.1    CORP12345
```

This solved the DNS resolution issue and saved me… about one minute. Useful, but nowhere near enough.

## Cache Loading Bottleneck

Next, I went through the logs carefully and noticed that the app was always getting stuck at the cache loading step. It was fetching over 100,000 records at startup from the database.

The first obvious idea was: “Why not use a local database instead of the shared one?” But here’s the thing—we’re talking about Oracle Database. Yeah, good luck spinning that up locally! Because nothing says developer-friendly like a 20GB install and licensing nightmare.

There were some Docker images floating around for Oracle, but in the company’s case, it wasn’t that simple. We didn’t even have the full schema available internally. A lot of database migrations were either lost or stored in some ancient FTP server (yes, this was really happening… in 2021).

## Trying Lazy Loading

Another solution was to make the cache lazy-loaded—so it wouldn’t load everything eagerly at startup. Luckily, the app had a config option for that, and I could even keep it as a local override just for me.

It worked… but only partially. Some parts of the app relied on the data being preloaded at startup. Disabling eager loading caused errors and missing data downstream. So I couldn’t fully use this solution without breaking functionality.

## Disk Caching: Saving the Loaded Data

The next obvious trick was to load the caches once, save them to disk, and load them from the local file system on subsequent startups.

That approach worked surprisingly well—but it required me to add a lot of custom code. I had to make several classes serializable, and I had to be very careful not to accidentally commit these changes. Still, it reduced startup time significantly and saved me from waiting 1 hour 30 minutes every time.

But… I wasn’t satisfied. I didn’t feel like I had truly found the root cause. I had a workaround, not a solution.


## Digging Deeper: Wireshark to the Rescue

Since software-level debugging wasn’t giving me a full answer, I decided to check things at the network level.

I had some basic networking knowledge (thanks to a few issues I had solved on my personal server in the past), so I fired up Wireshark and started capturing traffic. I filtered for Oracle traffic and watched what happened when the app was loading the caches.

That’s when I saw it:

- Each database request was taking about 500ms over the VPN.
- The round-trip latency to the database was around 300ms.
- But the real surprise? The app was fetching just 10 records per request.

I realized what was happening: the app was fetching tiny batches of data at a time—10 records per round trip. Over a slow VPN connection, with 300ms latency, that added up fast.

This parameter, known as fetch size, controls how many rows the driver fetches in a single database round trip. A value of 10 might have been fine in 1999, when our database driver was originally configured (and maybe optimized for lower memory usage)… but in 2021, over a VPN? Definitely not fine.

# Applying the Solution

Once I realized it was a fetch size issue, I thought, “Alright, this should be an easy fix—just bump up the fetch size in the config.” After all, the fetch size was already exposed in our configuration properties.

Well… not so fast.

Turns out, those configuration properties only applied to Hibernate, which handled most of the app’s ORM. But the caches? They weren’t loaded using Hibernate. They were loaded using… Castor JDO. Yes, Castor JDO 0.7. From 1996. Wrapped in our own internal library.

So, if I wanted to change the fetch size, I had to configure it inside Castor JDO, not Hibernate. And to do that, I first needed to understand where and how the wrapper library was setting up the Castor configuration.

## The Hunt for the Source Code

Here’s where it got fun.

We had the wrapper library as a dependency in the app—but no source code was attached. I asked around my team, but no one knew where the source was. I searched through all the company’s Git repos. Nothing.

I started to wonder if the source code was even still around.

Then I reached out to one colleague—let’s call him The Oracle—who had been at the company for 15 years. I explained what I was looking for, and after a pause, he said,

> “Oh… I think I know where that is.”

A few minutes later, he sent me a repo URL. A repo I had never seen before. It wasn’t in our main Git server. It wasn’t in the list of company projects. It wasn’t even documented anywhere.

Apparently, it lived in an old, dusty SVN-to-Git migration server that only a few people remembered existed. Maybe only him.

But inside that repo? The source code I needed.

It felt like uncovering a secret map in a forgotten archive.

## Exploring and Modifying the Wrapper

With the source code finally in hand, I dove into it. The codebase was old, but relatively small and easy to follow once I got past the initial unfamiliarity. I traced where the database connections were configured and where queries were being executed using Castor JDO. Eventually, I located the part of the code where I could set the fetch size.

Now came the next hurdle: building the wrapper library. It wasn’t using Maven or Gradle—this was Ant, with its familiar but archaic build.xml, essentially Java’s take on Makefiles. It felt like time-traveling back to the early 2000s.

After experimenting a bit with the Ant build scripts, I was able to compile the wrapper with my updated fetch size configuration (I bumped it from 10 to 10,000). I replaced the old dependency with my freshly built jar file.

Finally, I restarted the monolith. This time, the caches loaded in under a minute. Victory.

# The Aftermath

I presented my findings and solution to the team, complete with documentation on how to apply the modified library while we waited for the tech team to eventually handle it properly—by releasing a new version and updating the monolith’s dependencies.

But to my surprise, the reaction was… lukewarm. There wasn’t much enthusiasm. Some teammates were hesitant to adopt the fix, worried it might introduce unforeseen issues, given how deep and technical the root cause was. Others had simply adapted to the slow startup; they worked around it, so for them, it wasn’t really a problem anymore.

For me, though, it was still a personal victory. I learned a lot through this investigation, gained hands-on experience debugging across layers, and, most importantly, I could now start the app in 10 minutes—even from home.

Oh, and for the record: I finally got my remote development machine… two months later.

# Conclusion: Lessons from the Journey

This wasn’t just about fixing a slow startup—it was a deep dive into a legacy system, a lesson in patience, curiosity, and persistence. I learned how layers of abstraction can hide simple configuration issues, how old tools like Ant and Castor-JDO still quietly power critical systems, and how technical debt sometimes becomes invisible because teams grow used to it.

But most of all, I learned that not every solution will be embraced, no matter how effective. And that’s okay. Sometimes, solving a problem is valuable for the knowledge you gain and the doors it opens for yourself—even if it doesn’t change how others work.

In the end, I walked away with a faster app, a better understanding of the system, and a great story to tell.