---
layout: post
title: Is IRC technology flexible enough to scale?
date: 2016-01-31 21:17
last_modified_at: 2018-10-28 20:00
author: alex
tags: [irc]
threadId: axgy-1454206193433
---

**Update: October 28th 2018**

When I talk about scalability here, it's a wider focus than avoiding hard bottlenecks in the form of networks. It was not something which I expressed very well, so I will try again: Scalability in a technology project also includes aspects of how theses systems work for an entire team, and frankly, in traditional IRC set-ups this is an area which is very lacking.

It requires a team to have access to the systems they need, as well as the operational knowledge to maintain them, all while keeping tight security. It's a very tight balance and one I don't think any open organisations have proven that they manage very well: In fact, an appearance of stability usually hides one or many key-man dependencies behind the scenes, and a lot of (hopefully documented) institutional knowledge.

In general terms this is manifested by service disruption, and in IRC, by frequent netsplits. 

Here I was noting some of the thoughts I'd had at the time, thoughts which proved a little too ambitious for an open project. However since then some smaller, more practical steps have been taken, with many more coming. Hopefully this will make our implementation of _IRC technology flexible enough to scale_, within our organisation.

As for IRC itself, there's lots of great work happening with [IRCv3](https://ircv3.net/) which I strongly encourage any passers-by to follow and contribute to and I'm hoping someone will finally build that P2P "mesh-net" server-to-server protocol. 😅

The original post, as written in late 2015, remains below after the break.

---

For a long time now I have led a small IRC network which a great deal many people call home.  Overall it has been a very rewarding experience, and I've gained many useful technical and managerial skills, however it's not all sunshine and roses.  Perhaps one of my biggest pet peeves with the current state of IRC is something that I would very much like to improve, and that is the practical scalability of IRC. (i.e. not just more users, but more developers/staff and coping with quick changes)

To provide a bit of background, the vast majority of IRC networks run servers following a star topology, on hand-configured/secured servers.

    Hub 1
    |-- Leaf A
    |-- Hub 2
    | |- Services
    | |- Leaf B
    | `- Leaf C
    `-- Leaf D

Traditionally this has always made sense, but then what happens if a central node (i.e. in this diagram, "Hub 2") goes down?  Massive service disruption.  Even if someone has thought ahead and has set up automatic fallback routes and servers, there are other mission-critical components in a traditional IRC network such as the services and security application servers which, should they be available on a network, massively enhance the experience offered by a network and may include fundamental features that users have come to expect from a particular network.

With many networks, including Rizon and StormBit, this type of disruption - known as a netsplit - is considered the most unpopular aspect of the overall service.  To a robust community they are definite dealbreakers and understandably so, after all who wants to be in the middle of a conversation (or heated debate!) only for half of the participants to go up in smoke?

Indeed, going by [the 2015 user survey](http://stormbit.net/2016/01/17/year-of-the-monkey.html) the greatest cause of dissatisfaction (a whopping 56%) with StormBit's service was the downtime and disruption caused by netsplits, with similar numbers from 2014.

Ultimately there's no easy way of escaping this type of disruption, and while many networks benefit from running on robust infrastructure with very stable internet connectivity, it seems fairly backwards that this should continue to be the best answer, especially considering the snowflake state that these sorts of servers often fall into, and the reliance on many points of failure that are non-trivial to recover.

Now this is not at all a new topic, and [Randy Bias gave a talk in 2012](https://www.slideshare.net/randybias/architectures-for-open-and-scalable-clouds#20) on scalability advocating a distinction between the approach of treating servers as "pets", in which servers are set-up, maintained, updated and repaired by hand; and treating servers as being "cattle", where all of these processes are managed through automation (for instance, configuration management tools such as Puppet, Ansible or Chef) meaning that if and when problems arise, new servers can be spun up in seconds, not weeks.

Since some of the largest IRC networks in the world, such as QuakeNet and EFNet, have survived for decades following the older practices, the former method of treating servers as pets has proven to be successful however I do not doubt for a second that their infrastructures have become tangled snowflakes, painful to maintain or support.  Ancient practices have not held back technological progress in other areas, so why should they here?  IRC is still run today as it was in its infancy despite much progress in operations.

In terms of the protocols used to communicate between IRC servers, these have come a really long way since the original network.  A major modern contender is the [TS6 protocol](https://github.com/charybdis-ircd/charybdis/blob/master/doc/technical/ts6-protocol.txt) which, for example, introduces concepts designed around tagging individual servers, clients and messages with unique IDs and timestamps to ease the complexity of propagating information to new servers, to ensure that messages are delivered to clients correctly and to try and reduce desynchronisation between servers - especially when attempting to resolve the differences after an outage.

However, this is still very much centered around the idea of a star topology, and to my knowledge, all of these protocols still involve a full resynchronisation of the entire state of the network on every link, which in a production network can result in massive amounts of unnecessary information being distributed over the presence of users and channels which aren't necessarily prudent.  An example here would be a server carrying 60 users rejoining a network with 10000 users; it's highly unlikely that each of the new users is going to have a common point of contact with the remaining 10k, so why tell this server about users that are not relevant?

This kind of inefficiency seems irrational considering the progress that the IRCv3 protocol has made over the original, introducing advanced functionality such as progressive client enhancement, and SASL.  Slowly but surely we've been seeing movement in the world of IRC clients, most certainly accelerated by the dawn of other chat platforms such as Slack which are growing in success proving that IRC's relative user unfriendliness and lack of features are the only reason it is seeing a supposed decline.  The technology itself is fairly primitive and works quite well, but the basic nature of clients pushes away users who value convenience above all else.  However there hasn't been much movement in the backend side here, as I hope I have made clear.

With the ambitious goal of running a self-repairing, auto-scaling service for the future, at StormBit our recent architectural goal has been to build a new network to address these concerns from the very beginning - hosted primarily in a high-availability cloud environment.  We are relatively undecided about which IRC daemon to use; currently [InspIRCd](http://www.inspircd.org) is our choice as it allows for an unstructured linking set-up by means of its TS6-like [spanning tree protocol](http://www.inspircd.org/wiki/Modules/spanningtree/Protocol.html).

While no decision has yet been made about exactly which hosting provider to go with, the most popular choices at time of writing are [DigitalOcean](https://www.digitalocean.com/) and the rather stellar [Amazon Web Services](https://aws.amazon.com/).  Both have advantages, but the most important aspects to the selection process are an uptime guarantee, API access for automating provisioning systems, and the ability to communicate via a private network.  A number of providers fit the bill, but there are also questions on convenience and operational costs.

In terms of technologies, Docker seems to fit the bill for running the systems themselves, as it allows us to generate identical artefacted copies of systems to run on all boxes.  While there are questions over the viability of distributing an _entire operating system_ just to run a handful of applications, it also ensures that there's no possibility of introducing snowflake environments and leaves us feeling safer than relying purely on configuration management systems.  

Obviously it's still a requirement to store the state of the network (i.e. which servers are available) somewhere, and so we have opted to use [HashiCorp Consul](https://www.consul.io/) to allow automatic discovery of new nodes introduced into our cluster, and to adjust our round-robin DNS, both by means of the rather excellent [consul-template](https://github.com/hashicorp/consul-template) which allows applications that remain unaware of service registries to interact with them near natively.
Consul also offers us decent monitoring abilities, and so I have written a small utility in Go to perform a simplistic healthcheck on IRC servers, called [ircd-healthcheck](https://github.com/StormBit/ircd-healthcheck).  This should be readily usable with other monitoring systems too, a few solutions already exist but none that I could easily make work with Consul.

(For future improvements, we are looking at alternative IRCd technology, and the option of better integrating an external server registry into InspIRCd by means of a custom module - so watch this space for that!)

Ultimately I feel like there's plenty of room in this space for better approaches to automatically scale IRC, and to future-proof it in terms of practices.  If anything, the approach that we are taking at StormBit just to improve our resiliency seems like overkill and is definitely daunting to other networks of lower technical ability or of greater size.  I'm hoping that this article might spark some conversations about how to start catching up to the technical proficiency of our relatives in other areas.

If you have any questions or feedback, please don't hesitate to drop me a line.  I'm `@antoligy` on Twitter, `alex` on StormBit, and there's a comment box below.