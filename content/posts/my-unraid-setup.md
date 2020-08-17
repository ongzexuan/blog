---
title: "My Unraid Setup"
date: 2020-08-14T00:49:17-04:00
author: Ze Xuan Ong
draft: false
feature_image: /my-unraid-setup/unraid.png
tags: ["dev"]
summary: "Documenting going down the rabbit hole after looking for a basic hypervisor."
---

In the last year or so I've found myself in greater need of GPU resources for running various deep learning models. Instead of paying for AWS instances, which is the default supported choice of the various classes I was taking, I thought it made more sense to build a server to do that and pocket the AWS credits. The pros are that I appear to save money. The cons are that I have to deal with all the problems of setting up, running, and maintaining a server, but if you put a positive spin on problems self-imposed you can call it education, which I will likely pay for with the rabbit-hole-ism of such projects that are likely to end up costing more in the long run. 

Anyway after putting it off for a while, experiencing a Black Friday where for the first time I bought a CPU and a pillow case for their thread counts (w.r.t to price of course), and then subsequently having some minor success setting up and running my home server, I thought I would document some parts of it here.

### Choice of OS

As is usually the case, I initially reached too far and felt I needed a bare metal hypervisor to do whatever it was I wanted to do. I went with Proxmox because it was supposedly simple and easy, but my inexperience with such networking work left me only able to run a few VMs and not much else.

After an embarssing defeat by the networking gang, I settled for turning the server into a NAS, to backup some photos and data. It turns out that NASes these days can run containers and VMs, which is a pleasant surprise. Unraid came out as a simpler choice, with a free license for 30 days, thereafter until shutdown, which was forcibly imposed on me when a tree fell on my power line during a storm (if you look closely at the picture above you'll see that I have a legit license now).

### Computationally Heavy Workloads are Gateway Drugs

How does one get from running some models on another machine to setting up another server entirely? The rabbit hole works approximately as follows:

0. I want to stop paying for AWS (which I just realized seems to be a recurring theme on this page).
1. But I need a machine to run models, preferably not my main machine so I could do other things while waiting.
3. I need another machine (oh look let's conveniently buy a new machine on Black Friday so we can obsolete the old one and make it a server)
4. Wait we can have a server? Now we can run multiple services on the server!
5. Let's... attempt to do that the correct way by running multiple containers/VMs.
6. Ok I need a hypervisor.
7. Do I really need a hypervisor for running deep learning models?
8. Yes. Actually no. Ok yes.
9. Now let me add lots of services to justify having put a hypervisor on this machine.

### What It Does

Having run the server for almost half a year now, here's a number of things that the server does do, and quite effectively so.

1. Runs a music streaming server (Airsonic) so I can stop paying for Spotify.
2. Runs a bunch of Scrapy bots to collect data to build datasets so that I can stop paying for ScrapingHub and S3.
3. Runs a VPN so that I can fix and access the network from anywhere.
4. Runs an Apache Airflow VM for overkill scheduling of jobs that need to run on a regular-ish basis.
5. Runs a few databases for personal data collection (I collate all my credit card data in a single place here, but that's a post for another time), and for school work.
6. Backup my data, work like a NAS like, what it was invented for.

### What It Doesn't Do

1. Run deep learning models

It turns out the original use case has dwindled in value since the idea was mooted (to myself). I still run a model or two every now and then, but not at the frequency that I had originally anticipated, and certainly not at a level that justifies an external machine for it. Oops, but hey we have a fancy server now!

### I Save Money... Right?

I can say with certainty that the reduction of OPEX from unsubscribing from various online subscriptions has definitely saved me money. Furthermore, the place I currently stay in has a fixed electricity bill, so there's no marginal electrical expense from this server.

That being said, the OPEX costs have been convereted into CAPEX, which seems to be the opposite of the general trend these days. I lost a cache drive the other day, and while replacing it was easy and painless (slide in a new drive and turn on the drive array), it of course cost some money, which sort of set me back 6 months of supposed savings. 


A common response when I tell people that I go through the trouble of doing all this so that I can unsubscribe from Spotify is that it's far more trouble than it is worth, especially when you consider the value of my free time. I have to say I wholeheartedly agree (especially if it's just for Spotify), and it sure felt that way for the most part when I started doing this. However, since setting up my Unraid server I've found it really easy to deploy new services for any new projects I may have, non-business of course. Need a new database? Just spin up a new container or VM. Need a new backup location for service X? Just create a new folder in the NAS. This feels like a much cleaner separation of work with respect to my personal computer. It's also really easy to operationalize some script which would have otherwise been a one-off function to be scheduled manually. For example, I have triggers that upload a copy of my music to YouTube Music (previously Google Play Music) whenever new data comes in.

### More Stuff

I'll go into greater detail about some of the more interesting things I'm running on this machine in subsequent posts.



