---
layout: post
title: "Minimal Linux distribution - Part 1: LFS"
# category: "linux" // categories are working based on path also, no need to tag :)
---

A little while back, I decided to take the plunge and build my very own Linux distribution using the **[Linux From Scratch](https://www.linuxfromscratch.org/)** project. Spoiler alert: it was a wild ride!  

After plenty of hiccups, a few days of battling my own mistakes, and occasionally Googling things like “why the heck this thing is not booting?”,
I finally got it up and running. And let me tell you, the feeling of seeing it boot for the first time? Absolute magic.  

![LFS terminal](/assets/linux/minimal/lfs1.png)  
*Welcome to the Bash Shell!*  

![LFS top](/assets/linux/minimal/lfs2.png)  
*Behold: Top, showing just a handful of processes chugging along.*  

I was genuinely blown away by how fast and lightweight this thing is. Sure, it's not quite production ready
(it's definitely not replacing my current daily driver, **Linux Mint Debian Edition 6**), but the experience of building it taught me so much.  

Was it frustrating at times? Yes. Did I liked every second of it anyway? Also yes. If you`re curious about Linux internals and want a hands-on challenge, 
I can't recommend Linux From Scratch enough. It's an adventure worth taking!

This whole experience got me thinking, *"How low can we go?"* What's the absolute bare minimum of software needed to boot this thing? Could I strip it down even more?  

**Linux From Scratch** includes network support, GCC, Vim, and a bunch of other packages. But what if I will go further? A while back, 
I stumbled across an excellent Linux Foundation conference talk by **Rob Landley** on [YouTube](https://www.youtube.com/watch?v=Sk9TatW9ino). 
In it, he shared his journey of creating [Toybox](https://github.com/landley/toybox) (a interesting alternative to [BusyBox](https://busybox.net/)) and demonstrated just how minimal you can go.  
It got me wondering: could I pull that off too? Maybe use both **BusyBox** or **Toybox** to build a super-slim system? Or, for the ultimate challenge, could I write my *own* simple shell completely from scratch?

The possibilities are exciting and slightly terrifying. But hey, what's life without a little experimentation?

Fingers crossed, hope in my next post, I`ll have some hands-on experience to share about this idea. So stay tuned!  
