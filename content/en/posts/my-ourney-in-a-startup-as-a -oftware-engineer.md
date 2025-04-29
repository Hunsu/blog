---
title: My Journey in a Startup as a Software Engineer
description: 'I joined a startup early on, helped build it from the ground up, and learned a lot before seeing it eventually fail.'
author: Rabah M
toc: false
date: 2018-07-04
---

# Joining the Adventure

It all started back in 2014, during my first year of my master’s degree. Part of the curriculum was a group project that ran three days a week for four months. Honestly, my friends and I weren’t too excited about the projects proposed by the professors—most of them were just trying to get students to help with their dry research topics.

Then we came across something different: a project submitted by someone from outside the university. It immediately caught our interest. We met the guy—a seasoned electronic engineer working at a major car manufacturer—who was trying to launch his own startup. He had a clear vision: to build a tool that would speed up the design and development of electronic circuits.

In his experience, engineers were wasting five to six months designing circuits, juggling outdated software tools and doing endless calculations manually in Excel. His product idea was simple but powerful: let engineers design circuits using a huge component library, focus on a smooth and intuitive UX, and automatically handle all the calculations. It could even simulate whether a circuit would hold up in different countries' climates and temperatures.
We really believed in the idea. 

# The First Four Months

We decided to go for his project and officially picked it. He had just joined the Startup incubator, so we worked alongside him there. It was a cool environment—lots of people trying to build their own startups—but honestly, some of the ideas we heard around us didn’t seem very promising. (Things like "Uber but for umbrellas" or "a social network for pets.") Still, I later realized that even the craziest ideas can sometimes succeed... and the ones that sound great can crash just as easily.

The founder didn’t come from a software engineering background, but he had taught himself a lot. He was really into Clean Code, the SOLID principles—stuff we were just starting to hear about in school. He also believed strongly in agile methods. We had a kanban board full of post-its, and we tried to stick to Scrum, delivering something every couple of weeks.

One thing I appreciated was that he wasn’t just using us as free labor—he actually wanted us to learn and build something real. We spent about two weeks just working on a signup/login system, learning to use the Vaadin framework, applying the MVP (Model-View-Presenter) pattern, keeping separation of concerns between controllers, services, DAOs, and repositories, and even writing unit tests and integration tests.
It felt amazing to build something that wasn’t just going to end up in the trash once the semester was over.

# Becoming an Employee

After we wrapped up the school project, the founder asked if we wanted to keep going—with real jobs this time. Both my classmate and I still had one more year to finish our master’s degree, and we needed a contrat de professionnalisation anyway, where you work three days a week as a real employee with real responsibilities.
The timing was perfect, and the startup was starting to get serious attention—there was even talk of raising a million euros from investors.

We jumped right in, working on real features like the UI for designing electronic circuits and learning how the calculation engine worked, along with the science behind it.
Not long after, we moved out of the incubator and got our own office. As the project grew, we started hiring more people. In just a few months, we went from a small team to about ten people.
It felt surreal hearing that even big companies like Airbus were showing interest in the tool.

The founder really tried to build a workplace where people enjoyed coming to work. We started doing the typical startup things you always hear about: team offsites, working from cafés once in a while, having beanbags at the office, even having a "happiness manager."
We also landed our first customer, and it felt amazing to actually talk to real users, see them using our product, and get direct feedback.

# Troubling Times

As the startup grew, things started to get more complicated—especially on the human side. The founder was really attached to the idea of a flat structure where everyone could speak up freely. It sounded great in theory, but in practice, it led to chaos. Everyone, myself included, started acting like they were in charge, pushing their ideas and not compromising. Looking back, I’m still a bit embarrassed about it—I was definitely part of the problem.

We’d spend endless hours arguing over everything: whether something followed SOLID principles, if a merge request was too big to review, whether functional programming was better in that case... even how we organized work. Should we use story points? Should tickets always include what-who-why (or is it why-who-what)? Should we switch from sticky notes to a digital board? No one stepped in to make decisions, so disagreements just dragged on. Despite having more people, we ended up delivering less and spending way too much time in meetings—which is a red flag in any startup, and we totally missed it.

On the business side, we had plenty of interest from companies, but I learned the hard way that “interest” doesn’t mean “deal.” The people excited about the tool weren’t the ones making budget decisions, and procurement processes in big companies are painfully slow. We were also claiming we could reduce design time by 70%—a very attractive promise—but building a product that could actually do that was a different story.

Eventually, a major company gave us a challenge: reproduce the calculation for one of their existing circuits using our tool. It had taken them months to do it themselves, and they gave us a tight deadline. If we could match it, it would’ve been huge.

But the reality hit fast. The circuit was sent in a proprietary format, and although we claimed to support importing from popular tools, that feature didn’t exist yet. The components used weren’t in our library, and adding them manually took forever. Worse, one component wasn’t even supported by our calculation engine. And even if we had everything ready, the circuit was huge—running the simulation would’ve taken weeks. Our algorithms weren’t optimized, couldn’t run in parallel, and we were using BigDecimal for everything. It was like trying to train ChatGPT on a single-core i3 with no GPU.

Seeing that things weren’t working out and realizing I was contributing to the issues rather than helping solve them, I decided it was best to step away from the startup. I stayed in touch with my former teammates and kept an eye on the project, right up until it quietly came to an end.

# Why It Failed
## The Product Was Too Complex

The product tried to solve a real and painful problem, but the solution itself turned out to be extremely complex. Just allowing engineers to import their existing circuits meant reverse-engineering proprietary formats, which is difficult and time-consuming. Even the circuit editor—a basic feature—was a serious technical challenge to implement properly.

On top of that, the tool needed a massive library of electronic components to be useful. These components vary between manufacturers, and their technical specs are hidden in long PDFs or reserved for paying customers. We had to extract this information manually and turn it into formulas for simulation.

The simulation engine, while conceptually simple, didn’t scale. Simulating large circuits took far too long, and our algorithms weren’t efficient or parallelizable. We relied on BigDecimal operations, which made everything worse. And since big clients needed the tool to work with their internal systems, missing key integrations also added to the complexity.

## Lack of Experienced Leadership in Engineering

In the beginning, we were just three people—two of us still in school and the founder, who had learned software engineering on his own. While we were all motivated and eager to learn, we didn’t have anyone with real, hands-on experience building and scaling complex software products. That kind of guidance could have helped us avoid a lot of mistakes. We did try to hire someone with that background, but at the time, it was really hard to find experienced engineers willing to take the risk of joining a small, early-stage startup.

## When Craftsmanship and Agile Become Dogma

We genuinely believed in software craftsmanship and agile methods—I personally loved the idea of writing clean, maintainable code and delivering in short, focused sprints. But over time, these principles became more like dogmas than tools. We spent too much energy debating story points, ceremonies, or whether code was “clean enough,” instead of focusing on delivering real value. I eventually started pushing to drop Scrum altogether, since we were a small team and didn’t need all the overhead. My teammates didn’t share that view, and the disconnect grew. That clash of perspectives played a big part in my decision to leave.

# What I Learned

Working in a startup from its early days taught me lessons I couldn't have learned anywhere else. I discovered the excitement of building something from scratch, owning real features, and working closely with users. I also realized how hard it is to make a product that truly delivers value, especially when the problem is complex and the expectations are high.

I learned that believing in clean code and agile methods is great, but dogma can hurt more than help. You need pragmatism to ship and adapt. I saw how lack of leadership and structure—even with the best intentions—can slow a team down, and that strong opinions don’t replace experience. I learned that good intentions aren't enough to build a sustainable company, and that even great ideas need sharp execution, clear focus, and the ability to say no.

Most importantly, I saw how essential it is to listen, compromise, and stay humble—something I wasn't always good at, but definitely took with me after leaving.