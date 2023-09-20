---
title: "Fancontrol Script for Dell iDRAC"
date: 2023-09-20T21:45:00+09:00
description: "Telling your noise friend to shut up"
draft: false
tags: [hardware]
---

I have looked into some solutions for my noisy Dell PowerEdge R730 in the post [iDRAC 8 Settings and Tricks](/posts/idrac-8-settings-and-tricks). This is the solution I have come up in the end.

# Thought process

Through experience and experiments I have found that under my loads, CPU temperatures will hardly go over 50 degress celcius even with a 5% fan speed. Thus my idea was to just keep it simple and set two thresholds for temperatures that I consider to be harmful and dangerous. They can be modified themselves.

Also, the script is designed to run on the server itself, so IPMI usernames and passwords are not neccesary.

# Usage

All that is needed comes in this script. To run it, use this one-liner.

```
wget https://github.com/nagaeki/dell-idrac-fancontrol/raw/main/setup.sh -O setup.sh && chmod +x setup.sh && bash setup.sh && rm setup.sh
```

Check out the [Github repository](https://github.com/nagaeki/dell-idrac-fancontrol) for more.