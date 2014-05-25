---
layout: post
title: "Show HN: I Rebuilt My Side Project In Golang.js because How the Hell Else Will It Get to the Front Page?"
---

*Did it work?*

Let's just come out and say it: fuck Python. Fuck Ruby on Rails. And fuck whatever PHPooper-to-C optimization jank is in vogue at Facebook this week. Those pedestrian, battle-tested frameworks are for chumps. Sure, the extensive Stack Overflow answers are nice, but since when did the karma-controllers at HN give two shits about Python or Ruby? Not since PG was forbidden from watching PG-rated movies.

Using Perl? They will award you no points, and may God have mercy on your soul.

That's why I shifted gears and rebuilt everything from the ground up using Golang.js. Heard of Golang.js? I bet you haven't. You're forgiven if you think it's some kind of Golang-to-JS interpreter, which is what I initially thought, until the two people on #golangjs called me a Cro-Magnon troglodyte for even attempting to ask that question. They didn't have to say all those nasty things about my mother, though.

Anyway, Golang.js is a Rust framework that compiles directly into an experimental version of Dart (duh). It's compatible on 5 of the 7 Chrome nightly builds that came out last week. And it's the motherfucking future, guys.

Worried about performance at scale or managing a CDN? Worry no more. It taps into Mechanical Turk to dynamically request human drones who spin up Nginx servers off Raspberry Pis, while its S3 replacement leverages SpiderOak illicitly and is ultra-secure...somehow. Look, I don't have to understand the framework to understand the impact it will have on web development. Did I mention it's completely open source? I can't exactly find the source, but it's open, somewhere. Maybe it's on some dude's SpiderOak account.

But I digress. Are you interested in turning your barely dynamic webpage into a barely dynamic webapp that runs on the equivalent of web-x86-assembly? I bet you are, fellow HN'er. You're on the bleeding edge, and you're willing to hemorrhage a metric shit-ton of technical debt to keep that edge bloody. Because that's how you stay competitive in this dog-eat-dog world of Randian objectivism that we all delude ourselves into believing we're in.

The steps are simple. First, make sure you have a FreeBSD image running on a virtual machine somewhere. It has to be FreeBSD, and it has to be a virtual machine. For some reason it won't work natively, but I submitted a bug report to their BitGit repo wiki (BitGit, by the way, is a forked version of git that borrows concepts from the bitcoin blockchain. The maintainers demand you use BitGit for its distributed wiki. Also, totally the future, so watch out.) 

Got that rolling? Awesome. Get on the #golangjs IRC channel. There should be a guy there 30% of the time. He's going to call you a lot of names and insult your parentage. Ignore this. Graciously ask him for a precompiled toolchain for FreeBSD 9.1.

Oh shit, are you running FreeBSD 9.2-RC4? Yeah, you're gonna want to roll back.

Are you rolled back? Great. Things are kinda hazy from here, but if you search diligently on usenet you can find something called "Golang.js for Fucking Idiots." I had a copy but it's actually part of this self-contained OS image that self-destructs itself---and any copies on the local network---after 10 hours of opening. Really cool tech, you should check it out; definitely the future of reading.

Anyway, now comes the tricky part. You're going to have to read that self-destructing book in 10 hours and hope to the Flying Spaghetti Christ-Monster you retain some of it. This particular framework relies heavily on an esoteric concept called "funhouse reflection," whereby it will forcibly mutate supposedly immutable objects in your code and change variable scope at will, sometimes randomly. Also, best get your Towers of Hanoi out, because you're going to need your recursion toolbox. Due to some pretty clever implementations with the Dart interpreter, functional recursion is considered the preferred looping method.

Eventually, you'll wind up with some Dart code. DO NOT RUN THIS CODE. There's a special Dart linter you MUST run---but it only works on Windows 8.1 machines natively. Doesn't work in virtual images for some reason---still haven't found the maintainer of the linter, so I haven't submitted a bug report, but apparently they have a Gopher page and run an off-label meshnet BBS so I'm sure I'll be able to submit a bug report eventually.

Okay, at this point, if you're running a Chrome nightly, you'll finally see your pages 71.4% of the time. Unless Spideroak is down and no one has Turked your Nginx server, in which case you probably need to send some emails. To whom, I don't know, but that's an integration detail. At scale this system would work fantastically well, and if you're not preparing for scale...I mean, why are you even building it? If it breaks at not-scale, that's okay.

And that's it! With Golang.js, my sparsely-dynamic five page website took, soup-to-nuts, only three weeks of development time. Spent on nights and weekends. Without sleeping on the nights. Or the weekends. And it's not connected to a database yet, but Firehose support is supposedly only a few point releases away.

So, what do you think HN? How long until I redo it in Django? Feedback welcome, especially if it shatters my fragile self-worth!

{% include tbtc.html %}
