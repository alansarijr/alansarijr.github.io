---
layout: post
title: "OverTheWire Bandit — Full Walkthrough Series"
description: A complete beginner-friendly walkthrough series for the OverTheWire Bandit wargame, covering all 34 levels.
categories: [Walkthroughs, OverTheWire]
tags: [OTW, Bandit, Linux, beginner]
---

> **Bandit Series** — This is the **intro post** for the full walkthrough series.
> Jump directly to a level: [Level 0](/posts/bandit-otw0/) • [Level 1](/posts/bandit-otw1/)
{: .prompt-tip }

# Introduction  
OverTheWire is a great free platform where you can learn and practice cybersecurity by playing wargames. Each game focuses on a different area of security, and the one most people start with is called Bandit. It’s perfect for beginners because it teaches the basic skills needed for the other challenges—mainly Linux commands and a bit of Git.

I recently completed the Bandit levels and decided to write my own walkthrough for my blog. Sure, there are already tons of walkthroughs out there, each with their own explanations and methods. But I still wanted to share mine—partly to get more comfortable with writing and documenting what I learn, and partly because my way of explaining things might help someone who didn’t quite get it from another source. Plus, it’s always helpful to have something to look back on in the future.

I’ll keep the explanations simple and to the point, but keep in mind that each concept can go way deeper than what’s covered here. Just like the game itself encourages, I highly recommend doing your own research along the way.

Each level of Bandit builds on the last, and if I expand this into a full walkthrough series, each article will follow this structure:  
- **Login Information**: How to access the level.  
- **Task**: What you need to accomplish.  
- **Theory**: A quick explanation of the concepts or tools involved.  
- **Solution**: A clear path to success (with room for you to experiment first!).  

For now, let’s get you excited about what’s ahead and ready to tackle Level 0!

# The Levels: What You'll Learn  
Here's a quick overview of the journey from Level 0 to Level 33:  
- **[Level 0](/posts/bandit-otw0/)**: SSH basics—logging into a remote server.  
- **[Level 1](/posts/bandit-otw1/)**: Reading files with simple commands.  
- **Level 2** *(coming soon)*: Handling weirdly named files.  
- **Level 3** *(coming soon)*: Dealing with spaces in filenames.  
- **Level 4** *(coming soon)*: Finding hidden files.  
- **Level 5** *(coming soon)*: Identifying human-readable file types.  
- **Level 6** *(coming soon)*: Filtering files by size and permissions.  
- **Level 7** *(coming soon)*: Searching for files by ownership.  
- **Level 8** *(coming soon)*: Intro to `grep` and piping commands.  
- **Level 9** *(coming soon)*: Using `uniq` and `sort` to find unique lines.  
- **Level 10** *(coming soon)*: Extracting text with `strings`.  
- **Level 11** *(coming soon)*: Decoding Base64.  
- **Level 12** *(coming soon)*: Cracking a Rot13 cipher with `tr`.  
- **Level 13** *(coming soon)*: Working with hexdumps and compressed files.  
- **Level 14** *(coming soon)*: SSH keys and file transfers.  
- **Level 15** *(coming soon)*: Networking with `netcat`.  
- **Level 16** *(coming soon)*: Secure communication via OpenSSL.  
- **Level 17** *(coming soon)*: Scanning ports with `nmap`.  
- **Level 18** *(coming soon)*: Comparing file differences.  
- **Level 19** *(coming soon)*: Remote command execution via SSH.  
- **Level 20** *(coming soon)*: Linux permissions and SUID binaries.  
- **Level 21** *(coming soon)*: Background jobs with `netcat`.  
- **Level 22–24** *(coming soon)*: Exploring cronjobs and bash scripting.  
- **Level 25** *(coming soon)*: Brute-forcing with scripts.  
- **Level 26–27** *(coming soon)*: Escaping restricted shells with `vim` and SUID.  
- **Level 28–32** *(coming soon)*: Git basics—history, branching, tagging, and pushing.  
- **Level 33** *(coming soon)*: Shell variables and escaping.  

# How to Start  
Ready to jump in? Head to the [Bandit homepage](https://overthewire.org/wargames/bandit/), grab the SSH login details for Level 0 (`ssh bandit0@bandit.labs.overthewire.org -p 2220`, password: `bandit0`), and start exploring! If you’re stuck, check the level pages on the site, use `man` commands, or search online—learning to troubleshoot is half the fun.

# Conclusion  
Bandit is more than just a game—it’s a crash course in Linux, security, and problem-solving. By the end, you’ll have a solid grasp of commands and concepts that’ll serve you well in future CTFs or even a tech career. I hope this intro gets you pumped to start at Level 0! If I turn this into a full walkthrough series, I’d love your feedback—feel free to reach out to me on [LinkedIn](https://www.linkedin.com/in/moelansari/), [Twitter](https://x.com/alansarijr_) or on my [YouTube](https://www.youtube.com/@mohamedselansari). What did you think? What else do you want to learn? Let me know, and happy hacking!