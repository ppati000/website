---
title: "Professionalism in Software Engineering"
author: "Patrick Petrovic"
date: 2022-09-10T17:55:35+02:00
draft: true
---

Having worked as a software engineer for a few years, it turns out software engineering is a profession like many others:
you are part of a team with fellow humans and working to achieve something in a collaborative fashion.

I find it important that others enjoy working with me.
I would like to be a reliable and cooperative coworker.
In short, being a professional (but never boring) version of myself.
In this blog post, I would like to share some habits and ideas that I have learned to embrace over the years.
Not everyone may find these habits useful, though I hope they can give some inspiration!

## Review and Reflection

As professionals, we should be striving to deliver the best work we can (given our skills and resources).
Unfortunately, our brains do not always work towards this goal.
Sometimes the brain gets lost in details, and sometimes it glances over an essential piece of the solution.
I have the feeling that my brain works rather sloppily, despite pretending to be fully focused.
How are we supposed to deliver a clean and correct implementation if our brains work against us?

The trick I use is simple: Before I ask a colleague to review any of my work, I first review it myself.
I apply this trick to code and any other written output (e.g., design proposals or incident reports).
Almost always, I spot some leftovers, inconsistencies, or other shenanigans that my brain seemingly ignored.

Looking at the diff as a whole brings lots of value:
* Coworkers are less likely to encounter "stupid" mistakes, allowing them to focus on more important things (e.g., correctness and mantainability of the solution).
Automated tools can catch some of these mistakes, but they reach their limits quickly when "stupid" mistakes go beyond inconsistent syntax.
* I gain a more holistic view of my code. This insight allows me to document certain decisions explicitly.
* I feel smart for fixing mistakes that I made 15 minutes ago.

I am not sure if self-reviewing code works for everyone—but I can recommend you to try it.

## Ownership and Communication

Shockingly, even the most thorough tests and reviews allow bugs and other issues to slip through.
This will happen to any professional engineer at some point—and that is completely fine.
The crux of the matter is how we react to such an issue.
Whenever an incident or bug appears, I react like this:

* Communicate that an issue may have appeared.
* Assume that one of my recent changes may be related to the issue.
* Verify the scope of the issue and investigate the cause.
* Fix the issue if needed or involve other engineers as needed.
* Write a report that analyzes the root cause and possible actions to prevent the issue in the future.

Communication is crucial here—even if we are initially unsure about the scope of the issue.
Some issues later turn out to be non-issues.
It's not productive to be alarmist, but communicating a potential problem early shows respect to other stakeholders.

Keep in mind that not everyone is aware of all the technical details—so communicate appropriately for your target audience.
Most stakeholders will not care about a spike in database replication lag.
Nonetheless, they _will_ care about users not receiving up-to-date data.

## Appreciating Feedback

How did I learn about ownership and communication?
Before I started working as a software engineering intern, I had only ever coded for fun.
No one cared about the correctness of my code, how clean it was, or if anyone would understand it a few months later.

A few months into my internship, I received feedback about a formatting issue (back then, no formatting check was part of the CI pipeline, but that's not my point).
The reviewer suggested that I check my code for such issues before asking for reviews, including this line that stuck with me:

> This should be a habit for a professional.
> 
> —The reviewer

The reviewer's feedback was honest, direct, and yet humble: the reviewer did not blame the intern for writing bad code.
Instead, they gave me concrete suggestion on how to improve the quality of my code.
As you, the dear reader, already know, I turned self-reviews into a habit after receiving this feedback.

## Being a Humble Coworker

It's great to remember one thing: professionalism is about humans, not computers.
We should strive to give honest feedback and ourselves be open to receive feedback.
It is important to have difficult conversations–but we should not have them interfere with personal connections.

If you would like to give me some (positive or negative!) feedback on my blog post, please feel free to message me on [Twitter](https://twitter.com/ppati000) or [LinkedIn](https://www.linkedin.com/in/patrick-petrovic/) (only if you are a _true_ professional). Looking forward!









