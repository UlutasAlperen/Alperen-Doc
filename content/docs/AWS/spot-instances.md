---
title: "spot-instances"
weight: 710
---

# Spot Instances

Savings plans are great, but what if you want to save _even more money_? Allow me to introduce you to [spot instances](https://aws.amazon.com/ec2/spot/).

Imagine you're in AWS' shoes. You have to build out new racks of servers to keep up with demand, and you want to streamline things, so you install not just what you need _today_ but what you think you'll need for the _next few years_.

This leads to a problem: what to do with all the new capacity that's not being used yet?

AWS lets you **bid** on this unused capacity with **no long term commitment** at a **~90% discount**. With one small caveat.

> Dear Zach,
> 
> I'll give you this massive discount, but be warned, at any time I may literally delete your server. Don't worry, I'll give you a **2-minute heads-up**.
> 
> Love, Jeffy B

So, if you have a stateless application that only needs to do _async_ or _offline_ compute, spot instances might be the perfect choice. If a server dies, no worries, you can just start a new one and pick up where you left off.

But if you need to be _always on_, spot instances are not a good fit.