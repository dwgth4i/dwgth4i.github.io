---
title: "File-based Initial Access - MSI"
date: 2025-11-5
categories: [knowledge]
tags: [redteam]
---

I played with some kind of file-based Initial Access recently and really enjoy it when I realize there are so many type of file extensions that we can abused it as our weapons. However, I personally spend more time with ```.msi``` among them and this blog will be about it, why it is so reliable (imo). This blog will assumed that you are already familiar with some basic features of an Microsoft signed installer (MSI) like install, repair, uninstall.

# Why MSI ?
If you are already know some of the things that we could play around with an .msi file in lab environment, CustomAction must be a friend. However, there are some of its companions that are also important too:
- **Directory**: Installation directories
- **File**: Executables and files to be installed
- **Custom Actions**: Pre/post install actions
- **Binary**: Executables stored inside MSI
These should be the key points to look for and analyze if you are a blue teamer, for us attackers, we should pay more attention and modify them, make them more *innocent* to the receiver (the MOTW will be an issue anyway lol). That was some concepts about MSI, I think the only reason for it to be more reliable for phishing because I think it is more familiar to even non-tech people
# Sharpening our knife (C2, Evasion, ...)
## C2
Here in this demonstration, I will ultilize MythicC2 for the stage 1, the reason for this is (Merlin)[https://github.com/MythicAgents/merlin] agent, this agent is lesser well-known and it can fly through AV like nothing, another thing is, this agent has less functions compares to other which is lightweight and fit for the requirements of the stage 1 beacon.

![alt text](/assets/img/posts/merlin_agent.png)

It would look like this in Mythic if you installed correctly, I will skip the part creating the exe (or shellcode, dll if you want) and go for the next step with MSI

## MSI
Lets spin up Visual Studio, choose Create new project then go for the Setup Project, I recommend to use Wix toolset if you already familiar with the syntax.

![alt text](/assets/img/posts/project_startup.png) 

![alt text](/assets/img/posts/1290790712931293.png)

