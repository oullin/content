# Engineering Management Mistakes We Need to Stop Making

There's a lot of advice out there about what makes a great engineering leader. But sometimes the most valuable lessons aren't about what to do — they're about what to stop doing.

After years of working with engineering teams, I've compiled a list of 12 of the most common and costly mistakes we see over and over again. I've grouped them into four categories: leadership issues, team issues, technical planning issues, and the wrong responses to problems. Let's get into it.


## Leadership Issues

### 1. Putting a Non-Technical Person in Charge of a Technical Team

This one keeps coming up because it keeps happening. The person leading our engineering organisation — whether that's a Director of Engineering, VP of Tech, or CTO — needs to be technically competent. Not a manager who dabbles in technology as a hobby. An actual technologist.

If it's a programming team, their leader should ideally be a programmer.

It's not impossible to run a healthy team under a non-technical leader, but it creates unnecessary pain. When our engineering lead has never had to balance speed vs stability, take on technical debt to hit a deadline, or resist the urge to write beautiful code in favour of shipping, they won't know when to advocate for the team and when to push back. Those are hard calls that require lived experience.

The minimum acceptable version of this? A non-technical leader who deeply respects and trusts the senior engineers on the team, and who will go to battle for them when they say something is a problem.

### 2. Hiring a CTO Who Immediately Wants to Rewrite Everything

This one is especially common in startups and scale-ups. The company has grown on one tech stack, they bring in a new CTO, and within weeks that person is saying: *"This is all terrible. We need to throw it away and rebuild it in [favourite stack] — I've got people from my last job who can help."*

What they're overlooking is massive.

That existing codebase isn't just prototypes and POCs. It contains years of encoded business logic, hard-won decisions, and institutional knowledge. The team that built it? They're probably gone now, along with everything they knew. The result is almost always a 6- to 12-month regression — not the "couple of months" that was promised.

Here's a truth that holds without exception: no CTO who has promised to rewrite everything faster and better has ever actually delivered on that timeline.

If our tech stack and application are working and have gotten us this far, we should find a technical leader who knows that stack and can help us move forward. Don't throw it away.

### 3. The Superman Developer Bottleneck

Most of us have worked with one — the 10x developer who can do anything, works in any language, and seems to handle every problem single-handedly. Everyone loves them, right?

Here's what a closer look usually reveals: while this developer is incredibly productive on their own, they're often unable to build systems that other people can maintain. They slowly take over every critical piece of the codebase until everything flows through them. They become the single point of failure.

They'll often say things like, *"I wish I wasn't the only one who could do this"* or *"I haven't been able to take a vacation in years."* And we probably feel grateful — they are working hard. But the organisation can't scale or stay healthy when so much depends on one person.

No individual contributor, no matter how talented, should be a choke point for the entire operation.


## Team Issues

### 4. Building a Team Almost Entirely of Junior Developers

It's tempting — junior developers are cheaper, there are more of them available, and there's even a good intention behind it (giving people their first shot). But a team stacked with juniors is a liability.

They pull down our capable developers. They burn through time and money. They make the whole team look bad, which then drives away the experienced engineers we actually need.

Hiring junior developers is genuinely valuable — for the fresh perspectives they bring, for the pipeline they create, and for the industry as a whole. We won't have senior engineers tomorrow if no one hires juniors today. But we need to keep the ratio sensible: **no more than one junior for every two experienced developers.**

### 5. Going Cheap on Contractors

The logic sounds appealing at first: if we hire offshore developers at a quarter of the cost and they produce even half the output, we come out ahead.

The problem is that's rarely how it plays out.

Even when the output *looks* fine on the surface, cheap contractors often cut corners in ways that compound over time. Spaghetti code. Brittle architectures. Security gaps. These are expensive problems to fix later — often more expensive than just hiring capable developers at a fair market rate in the first place.

Cheap inputs produce cheap outputs. We should hire well and pay accordingly.


## Technical Planning Issues

### 6. Adding More Developers to a Late Project

[Fred Brooks](https://en.wikipedia.org/wiki/Brooks%27s_law) wrote about this in 1975, and it's still being ignored today. The instinct when a project is running late is to throw more people at it. But that almost never works.

More developers mean more onboarding time, more communication overhead, and more friction as people step on each other's work. We're adding all that burden right when the team is already under the most pressure.

***Brooks said it well:*** one woman can make one baby in nine months, but nine women can't make a baby in one month. Some things just don't parallelise like that.

And this isn't just about late projects. Every team has an ideal size for the work at hand. Beyond that point, adding people costs more — in process and coordination — than it returns in speed.

### 7. Copying Architecture from Companies 100x Our Size

AWS uses microservices, so we should too, right?

Wrong.

AWS has hundreds of times as many developers as most of our teams. Their problems are not our problems. Their solutions are not our solutions.

Microservices make sense when 50 engineers are working on a single service. When one developer is responsible for 22 concerns, splitting each one into its own service doesn't simplify anything — it makes every deployment harder, every feature branch more complicated, every bug fix a multi-repo coordination effort.

We should build for the team we have today, not the company we hope to be in ten years.

### 8. Ignoring Technical Debt Because Product Keeps Winning

When there's a strong divide between product and engineering, requests for technical debt are often dismissed as "the nerds wanting to play with their toys" rather than as real business priorities.

But unmanaged technical debt is a real business risk. We need to treat it as such.

The good news is that this is usually a communication problem, not a values problem. When we frame technical debt work in terms of risk reduction and long-term stability, most reasonable leaders will engage with it. We just need to learn to speak that language.

### 9. Chasing New Technology for Its Own Sake

There's a pattern in engineering teams where newer always feels like better. *Sure, we could use server-rendered HTML, but Vue.js is the cool thing now, so obviously that's what we should use.*

This thinking has real costs. New tools mean our team is less experienced with them. There's less community knowledge, less tooling, and fewer answers on blogs/the internet. The shiny object is shiny, which means our judgment about whether it's actually the right choice is probably compromised.

Before adopting something new, we should ask: what does our team already know? What does the rest of the application use? The most important factor in choosing a tech stack is often fit — with our team, our codebase, and our current problems.

### 10. Reaching for Complexity When Our Simple System Has Problems

This is the cousin of shiny object syndrome. The logic goes: *our simple system has a problem, so obviously the answer is a more complex system.*

It almost never is.

If our models are disorganised, we should organise them — not migrate to a repository pattern. If our directories are getting crowded, we should restructure our folders — not go full domain-driven design. If our monolith has issues, we should fix them within the monolith before breaking it apart.

The problems we have now almost certainly stem from how the system was used, not from its simplicity. Adding complexity brings the same problems into a harder-to-maintain, harder-to-learn, higher-friction environment. We should fix the actual problem first.

### 11. Adding Process Complexity Instead of Actually Leading

The same instinct that makes teams reach for architectural complexity also shows up in how we handle process problems.

Is the team struggling to stay on top of tasks? Let's hire a Scrum Master and spend nine months re-training everyone on a new framework.

Before jumping to that, we just need to lead. Talk to the team. Find out what's actually going wrong. Ask whether the problem can be solved inside the system we already have.

Is a client complaining that they don't get enough updates? We could roll out a complex ticket-tracking system that requires everyone to rethink how they work — or we could ask the team to write a few bullet points in Slack at the end of each day.

Not all processes are bad. But all processes have a cost. Just because something is expensive, complex, or has a logo doesn't mean it'll fix the problems we actually have.


## Wrong Responses to Problems

### 12. Promising Things We Won't Follow Through On

This one applies to all kinds of leaders, not just engineering managers.

It's easy to talk about how the team is family, how we offer flexibility, and how we'll go to bat for people. It feels good. It makes us look like great leaders at no cost to us.

But if we say those things and then back down when it becomes inconvenient or expensive, we've done more damage than if we'd said nothing. Trust, once broken like that, is very hard to rebuild.

The rule is simple: don't commit to something unless we're ready to honour it when it's hardest to do so. Only make promises we're prepared to keep.


## Final Thoughts

Good engineering leadership isn't just about having great ideas — it's about avoiding the patterns that quietly undermine everything we're trying to build. Most of these mistakes aren't born out of bad intentions. They come from cost pressure, inexperience, or just following what looks like a reasonable path.

But knowing the patterns is the first step to breaking them.

If any of this resonates — or if you're working through one of these challenges right now — share this article with someone who needs to hear it.
