---
title: "auto-scaling-groups"
weight: 30
---

# Auto Scaling Groups

The process works fine for one server, but  if we needed _hundreds_ of servers?

[Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html) (ASGs) help with this. We can give AWS an AMI and tell it how many instances we want.

This gets even more powerful as we _instrument_ our servers. If a server starts to get **busy**, AWS can automatically provision more. If servers are sitting **idle**, we can reduce the server count.

Another thing I use these for is to rebuild a server if it crashes (an ASG of 1). My cat photo server is not known for its reliability, and I'm not known for patience. So if my single server crashes, the auto scaling group throws it away and starts a fresh one from the AMI.

## Stateful and Stateless Applications

Servers should be treated like cattle, not pets.

 Abraham Lincoln (probably)

Cattle are raised by the rancher for a functional purpose, whereas pets are adored, named, and become a part of the family they reside in.

What does it mean to treat servers like cattle? It means we describe the kinds of resources we need and how many, then let specific instances be created and destroyed automatically as needed. We should not feel emotionally attached to a particular VM.

In AWS, ASGs and Launch Templates allow this. When a server fails, there is no need to patch it, nurse it back to health, or come up with clever names for it. It served its purpose and can be replaced by a new server.

However, your application needs to be designed for this. If your app server holds critical customer data, or data that can't be recreated, deleting that server would be very unfortunate.

That's why many backend developers like to write stateless applications. All the critical bits of our app are stored in a separate stateful database, but the compute servers (usually HTTP/REST/JSON servers) can be created and destroyed as needed.

    Stateful = stores critical data that can't easily be recreated either in memory or on disk.
    Stateless = stores no critical data; operates on data as it passes through the network.
