---
layout: post
title:  "From Slack to Snack: Replacing Critical Dependencies"
description: How Wilco tackled the challenge of replacing a widely-used third-party dependency
date: 2022-09-05
minutes: 8
image:  '/images/posts/slack-to-snack/slack-to-snack.jpeg'
tags:   [architecture, refactoring, wilco]
---

It’s a story as old as time: your team is responsible for a major system feature that’s widely used, but it’s not just your code that makes it work. There’s a third-party dependency: an external platform, a database, a library — something that’s there seemingly for the long run. But one day, your team can no longer rely on that dependency. There can be many reasons for this, from performance to missing features to pricing. And so, a search for a replacement is launched. 

If you’re lucky, there’s a perfect replacement with similar integration. If you’re even luckier, the business logic is well separated from the third-party integration code. Don’t count on it, though—few people are **that** lucky. 

And so, today, we will look at tackling a project of replacing a critical dependency. In our case, it was replacing Slack with Snack. 

#### Wait, what? Slack? Snack?
Corporate messengers are an integral part of software engineers’ work. Most often, it’s our most basic communication tool at work. And since at Wilco we aim to simulate a real working environment, we’ve decided very early on to integrate with Slack. 

Up until a few months ago, connecting Slack to Wilco was part of every new user’s onboarding. This is how they received assignments and communicated with co-workers like their R&D manager, Ness, and Navi, the DevOps guy. Both are bots, by the way, or so we have been told — my attempts to look at their code were rebuffed for some reason, and they sometimes send me messages asking if I can hear the voices too. But never mind that. 

![ness](/images/posts/slack-to-snack/ness.png)

It did not take long for us to realize that Slack wasn’t the best fit for Wilco:

-- When users connect to Slack, they get Slack’s onboarding experience. For us, it meant added friction in the form of additional popovers and clicks before our users could start their Wilco journey.

-- We had issues with users' privacy and message history. All of our users were added to the same Slack workspace so they could see all the other users, and Slack workspaces are limited to viewing only the last 10K messages, unless you use paid plans, impacting scalability.

-- We used Slack in a non-traditional way and found ourselves struggling with ״hacking״ Slack’s API for our needs.

-- As Slack is a closed platform, our options to customize the chat UI was limited, and we had almost no control over the look and feel.

So we decided it was time to replace Slack with our very own messaging app, Snack!

The project's main goal was to find and integrate a new service that would manage all the conversation data between our users and the bots while allowing us much more control over the experience.

#### Research, Discussion; POC, Discussion
This kind of task typically starts with a research phase. An engineer will spend a few days researching the available options. At this point, there are two alternatives:

1. The engineer will choose the best option, develop a quick POC, and continue with the implementation.

2. The engineer will create a document summarizing the various options. The team will discuss and choose the best option, and only then the POC phase will begin.

Too often, the discussion phase is overlooked. Indeed, adding a discussion phase means the project will take longer to accomplish, but it is worth the extra time. This is an excellent opportunity to identify missing features and ensure the selected option matches the company’s roadmap. A single engineer, even a very experienced one, is more likely to miss essential details.

Moreover, it is important to accompany the POC phase with an additional discussion. As the project progresses, going back and selecting another service gets increasingly more expensive. While working on the POC and gathering more information, it’s very common to discover additional limitations. This is the best time to decide whether to continue with the full implementation or go back to the drawing board.

In our Slack to Snack project, we used Google Slides and Notion to summarize and share all information gathered before and after the POC, and to have a meaningful discussion. Eventually, we decided on CometChat to host and manage our chat solution data.

![notion](/images/posts/slack-to-snack/notion.png)

#### Refactor when it makes sense. Rewrite otherwise.
Business logic code should be separated from third-party service specifics. We all know that, right? But in many real-world scenarios, this is not the case. Slack was integrated into Wilco at a very early stage of the startup, when the requirements changed rapidly and code was modified repeatedly. We ended up with business logic code tightly coupled to the Slack integration code.

Integrating CometChat into the existing code would have been difficult and would result in even more coupling and complexity. Our intuition in such cases is to start with refactoring:

1. Cover existing code with unit tests to allow us to modify it without the fear of introducing bugs.

2. Refactor existing code to decouple the third-party service code from our business logic.

3. Integrate with the new service while keeping the business logic generic and independent from service specifics.

However, ours was one of those cases in which switching services comes with adopting a very different integration method. While CometChat exposes a Rest API and uses webhooks that require custom implementation, Slack is integrated with [Bolt](https://api.slack.com/tools/bolt), encapsulating all HTTP requests and using a listener functions-based API. Business logic that supports both of them would have been much more complex and significantly harder to test and maintain. We decided to go in another direction: refactor and reuse only a tiny part of the code, and rewrite most of it.

Just like with refactoring, rewriting logic starts with unit tests. However, in this case, unit tests have a slightly different purpose. They are used to document the existing logic, helping us ensure that every current scenario is successfully replicated. And the best part is that we end up with code already covered with unit tests. We also learned from our experience and completely decoupled our business logic from CometChat specifics.

Now, if we will decide to switch to another service, this will be a much simpler task!

#### Break into small production-ready chunks
Projects like this take a few weeks, and sometimes more, to be fully ready for production. But does it mean we must wait so long until the new service is functional? Not necessarily!

Usually, we can break a project into smaller pieces. Each can be deployed to production as soon as it is ready. Ideally, we can easily replace the old functionality with the new one. When this is impossible, we should try to parallelize the two services. This was also our strategy when replacing Slack with Snack.

First, we implemented the registration functionality, adding all new registering Wilco users to CometChat. They still used Slack just as before, but we could start building our user base in CometChat. Then we parallelized sending all messages to both Slack and CometChat. We sent messages from our bots to both services, and every message sent by the users in Slack was intercepted by our server and sent to CometChat as well. Not only did this allow us to test the message-sending functionality thoroughly, but when we were ready to switch to Snack, all users’ conversations were already in CometChat. Finally, when the majority of the features were ready, we could start testing the entire flow in production. We used [split](https://www.split.io/) to first allow access to Snack only for Wilco employees and then gradually rolled it out to the users. Every few days, we switched part of our users to using Snack, and the transition process was complete after a few weeks.

#### What about the old code?
Hooray! The replacement process is complete, and all users are now using the new service. All that’s left is to decide what to do with the old code.

The worst thing we can decide is to just keep it. We might tell ourselves that a major bug could be discovered in the new service, and we will have to switch back quickly. This is true, and I believe it is worth keeping the old service functional for a short period of time. But code starts rotting the moment it stops being in use. Our codebase is changing rapidly, and code that’s not in use will slowly but surely cease to be functional. Not only that, but we will also have to consider this code when adding new features or modifying existing APIs.

Once we feel comfortable with the new code, we can have fun with the absolute best part of the process: deleting old code.

#### What a journey
We started with research and POC, followed by a team discussion. Next, we identified parts of the code that should be refactored and parts of the code that are better to be rewritten. Unit tests were used to document existing behaviors and give us confidence when replicating them to use the new service. Splitting the project into small production-ready tasks and parallelizing Slack and CometChat allowed us to deploy to production early and start testing while keeping Slack fully functional. We first opened Snack to internal users and slowly rolled it out to all users. Finally, we celebrated by deleting old code. Win!


###### *This post was originally published on the [Wilco](https://www.trywilco.com/blog) blog*
