---
title: OSEP & CAPE in 2~ years from scratch
date: 2026-09-26 16:00:00 +/-0000
categories:
  - HackTheBox
  - Offsec
tags:
  - hackthebox
  - offsec
  - cape
  - osep
description: A retrospective on what is new since the last big blog post.
image: /assets/img/posts/progresstwo/progresstwo.png
---
**So what have I been doing for the last year?**

# Getting a job from a blog post?

I know the heading sounds like some made up clickbait to get you to read the rest of this blog, but, no really, `this is actually what happened`, kinda...

#### The blog post

Last year I wrote [this](https://scotsec.github.io/posts/Progress/) blogpost about passing OSCP and CPTS within one year of starting to study IT, cyber security and ethical hacking. It was shared on both the HackTheBox discord and on their Reddit page, and it blew up a bit, `reaching an audience of over 60k+ people/bots/ai agents`. Lucky for me, it turns out actually `taking the time to write out proper non-AI-slop posts means that some people will actually take the time to read them`. In this instance, one of the principal testers from my current employer happened to be scrolling through the HackTheBox Reddit and decided to read that post, then enjoyed it enough to reach out to me on Linkedin. Queue sending of the CV, one phone interview and then one face to face interview. I must've impressed someone along the way as shortly after `I received an offer of a penetration tester position` (nope, not a junior position). 

#### Starting as a pentester

Due to starting dates `I had to ask to leave my previous position a month early to begin the new job`, they were happy to support this. Additionally, the local college were able to prove funding for both my CSTM training and the exam out of the reskilling fund that had been set up for refinery workers moving into new careers to ease the transition. These are not cheap courses so the fact that this support was provided definitely made the move into pentesting significantly easier, with CSTM/CTM basically being a standard requirement for anyone doing normal pentesting within the UK. 

Since then I've been `working as a pentester`, doing all sorts of work, anything from your standard infrastructure, web and cloud testing into more niche areas like `physical testing, OT/ICS and of course, my favourite: Simulated attack/Adversary simulation`. The work has been quite varied, which has helped to keep things interesting, as `I had heard horror stories from other hackers about being stuck doing web app testing` when their main area of interest in infrastructure/AD, however this was not the case for me. My very first job was on-site, doing infrastructure testing against a mix of both Windows and Linux domain joined hosts, perfect! 

### Doing some good hacking

I would say that `I progressed quite quickly at work`, gaining a reputation for having a knack at `pulling off some interesting and complex exploit chains`. This was highlighted by me `discovering my first (and currently only) CVE` on one job, [CVE-2025-10659](https://www.cve.org/CVERecord?id=CVE-2025-10659),  which was a CVSS v3.1 `9.8 - CRITICAL` this even came with an `ICS advisory from CISA` ([link](https://www.cisa.gov/news-events/ics-advisories/icsa-25-273-01)) due to the type of sector that the affected software is used within. The `responsible disclosure process was nice and smooth with CISA, no complaints`. The CVE basically allows an unauthenticated attacker on the network to obtain remote code execution in the context of the software service account, pretty bad! The vendor resolved this pretty quickly and due to the sensitive nature of the CVE and it coming from one our clients, I didn't think that disclosing the exploit details was the right thing to do, I think the UK has seen enough OT and ICS attacks recently.

### ProgressTwo

Anyway, I did a year and bit of good hacking and consulting, recently getting promoted to `Senior Penetration Tester`, nice.

During this time I continued keeping up to date with my studying, achieving `CAPE`, `OSEP` and `CRTO`, in addition to the UK specific things like `CSTM`, `CTM` and my UKCSC professional title `PraCSP`.

Moving on...

# OSEP vs CAPE - The part you actually came here for!

### Content

**OSEP**: Honestly, the `concepts are solid if you want to learn basic and old AV evasion techniques`. Yes, most of them wont work these days on modern defences, but they at least can `teach you about the fundamentals`, and there is always good value in that. You will end up learning about a load of weird and wonderful ways of carrying out attacks and obtaining beacons, I've yet to come across a situation where most of these might be useful, but never say never. I did this cert after `CAPE` so the AD section was a bit of a joke, don't expect to learn much here.

**CAPE**: Similar story with things starting to look a bit dated, however, all the `techniques within the course are the exact type of attacks that you will be performing in any modern AD environment`. Most, if not all, of the content is already available online through blogs, but having a nicely designed learning path with practical labs to reinforce the learning is the big selling point for me. 

**WINNER**: `CAPE` - Learn advanced techniques you can instantly apply in any enterprise environment.


#### Labs

**OSEP**: Surprisingly, given the somewhat negative outlook on the course content, the challenge labs are `some of the most interesting and technically demanding labs` I have ever done. They include things like phishing active users and developing advanced (`see: jank`) payloads to bypass defences. Many of the `payloads created during these labs have actually been useful on engagements when bypassing defences like WDAC` and zero trust solutions (blog soon maybe?). I just ended up skipping the module labs, so I don't know if they are any good, probably not, based off the course content.

**CAPE**: The CAPE labs are good, just `not as good as the OSEP challenge labs`. However, I do like the fact you are forced into doing them. It ensures that you are really understanding the course content and reinforces the concepts that you just learned. One `big downside is that there are no capstone challenge labs` to practice before your CAPE exam, you are pushed onto the labs platform to pay for VIP+ and Prolabs. It would be nice to see a final big practice lab for the exam similar to what is available within CPTS in the form of "Attacking Enterprise Networks".

**WINNER**: `OSEP` - The challenge labs are just too good. 


### Exam

**OSEP**: This is a `48 hour` exam with an additional `24 hours` for reporting. I completed it in `2 hours and 30 minutes`. If you did the challenge labs and understand what you are doing, you should breeze through this. Reporting took around `3-4 hours`, you only need to deliver a walkthrough similar to a box writeup, not a full professional report.

**CAPE**: This is another HTB classic `10 day exam`. I completed the practical section in just under `3 days` with the reporting taking around another `3 days`. `This exam is absolute hell on earth and will test your AD knowledge to the absolute limits`, however being finally done is a rewarding experience. Don't be surprised if you are stuck on certain flags for a significant amount of time, just stick to the course material and use your brain, `you can get through it`. 

**WINNER**: `CAPE` - No contest here, `the CAPE exam will have you feeling and acting like an AD wizard` once you have completed it.


### Price

**OSEP**: Fine, if someone else is paying.

**CAPE**: Significantly more expensive than CPTS but manageable.

**WINNER**: `CAPE`


# TLDR / Conclusion

**TLDR**: These certs `teach you two different things` and are not directly comparable (lol). If you want to learn ancient evasion techniques or go for OSCE3 then yeah, OSEP is fine, otherwise `don't waste your money (Buy MalDev Academy instead)`. CAPE on the other hand also contains a lot of older AD techniques that are widely published online, however these are still `fundamental to modern AD attacks and can instantly be applied both on real engagements` and in labs/CTF. Both could do with an update at this point but the bottom line is that both will require you to think outside the box and teach you new things, however CAPE will teach you more applicable techniques (and also make you think about 20x harder).

For both exams I used `Sliver C2` and a custom shellcode loader, plus some other tricks.

### Advice for those taking the exams

**OSEP**: `Do all of the challenge labs`, if you can get through these, even with the occasional hint or nudge, you will be more than fine for the exam. Prepare various payloads in advance so you can make tweaks rather than needing to make major changes. The Cybernetics Prolab is also good practice if you want to do something on HTB.

**CAPE**: Stick to the course material, but also think outside the box when you are stuck, `it's usually not as complex as you think`. You can find a list of recommended boxes and Prolabs for the exam maintained by VegeLasagne [here](https://thepastamentor.github.io/cape-prep-guide.html). Highly recommend `completing as many VulnLab machines as possible`, as these tend to be the closest to the exam.

**BOTH**: `Make sure you have good knowledge and experience with your C2 of choice`, for both exams I used `Sliver`. However, I can also recommend using `Adaptix`. The exams are both possible without using a C2, however I would say that using them makes it significantly easier. If you have completed all the labs and Prolabs and want even more C2 experience while also grabbing a cert along the way take a look at `CRTO`, it's sick, you should do it.


# What am I doing now?

At the moment I am continuing to develop my `red teaming and adversary simulation skills`, keeping up to date with HTB where possible and working on learning `C` and malware development on `MalDev Academy`. If I find the time I would also like to publish another blog post about some techniques and tooling that I have created for evasion and bypassing `WDAC`. But we will see!

Thanks for reading!

PS. Imagine writing ANOTHER blog post with no AI.
