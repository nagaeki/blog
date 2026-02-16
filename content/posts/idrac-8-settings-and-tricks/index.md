---
title: "iDRAC 8 Settings and Tricks"
date: 2023-09-20T18:27:00+09:00
description: "Got a Dell R730 the other day and found out some things. Putting them down here so I won't forget about them."
tags: [hardware]
#featured_image: "featured_image.webp"
draft: false
hidden: false
---

Information might change with time. Categories based on left column in iDRAC GUI.

# Server

## System Hostname & Operating System

Normally these two values are provided by the [Dell EMC iSM (iDRAC Service Module)](https://www.dell.com/support/kbdoc/en-us/000178050/support-for-dell-emc-idrac-service-module). Check to see if you can install iSM to update these values in realtime.

However iSM might not support the system you're using. Those times it is helpful to manually set them. You can either use `racadm` or `ipmitool`.

### racadm

To use racadm, either ssh into iDRAC, or use the mobile app `Dell OpenManage`,or use the `racadm` utility on your device. The `racadm` utility does not support any of my current operating systems in use, so I will use the first two methods.

Use the following commands -

```
racadm set System.ServerOS.HostName hostname.example.com
racadm set System.ServerOS.OSName "OSName"
```

### ipmitool

Install `ipmitool` on your system. Most Linux repositories have this utility ready.

To use this, you first need to ensure that IPMI over lan is enabled. Go to iDRAC Settings -> Network -> scroll down to IPMI settings and check `Enable IPMI Over LAN`. Click `apply` in the bottom right corner.

Usage -

```
ipmitool -I lanplus -H $IP -U $USER -P $PASS raw $COMMANDS
```

Note that `lanplus` is constant. To set hostname, the following IPMI raw command can be used.

```
raw 0x06 0x58 0x02 0x00 0x05 0x04 0x4c 0x4d 0x4e 0x4f

Byte 0 : 0x06 Network Function
Byte 1 : 0x58 Command
Byte 2 : 0x02 Parameter Selector (distinguish between set Firmware Version, OS Name, System Name etc.)
Byte 3 : 0x00 Set Selector (this equals 0 implies that the user is setting the parameter)
Byte 4 : 0x05 Check String data
Byte 5 : 0x04  --> data length
Byte 6…n : data bytes

Usage:
For example, to set the hostname to "DELL":

raw 0x06 0x58 0x02 0x00 0x05 --This part is common
0x04 -- hostname string length- four characters(DELL) in our example
0x44 0x45 0x4c 0x4c -- ASCII equivalent for DELL

It is advised to use a ASCII to Hex convertor with the prefix 0x to easily translate the characters.
```

## Virtual Console

Though HTML5 is the shiny new thing, I actually prefer to use Java here because of poor HTML5 performance on iDRAC8 and because how it always gets blocked for unsafe certificates.

## Alerts

This is important!

### Alerts

Firstly enable `Alerts` in the top.

For `Alert Filters`, I would uncheck `Info` because it's often not that important. Then check what alerts you wish in the table down below. Note that you need to do this for every page. With `Info` unchecked, that will be 13 pages.

![Alerts Page Example](images/idrac_alerts_page.webp)

### SNMP and Email Settings

Here you will need to set destination addresses for SNMP and email alerts. Down below you will find SMTP server settings. There are two things to take note here -

1. Regarding authentication, because of how old iDRAC8 is, it does not support newer versions of TLS and newer cipher suites, so you will need to change your mail server settings accordingly.
2. Regarding the username, this is only the username for **authentication**, not the mail **envelope-from address**. This might trigger safeguard mechanisms within your mail server, as this could be used for forging from addresses. You will need to allow this **authentication address** to send mail as the **envelope-from address**. I will mention this again [down below](#dns-idrac-name-and-static-dns-domain-name). 

What's a `envelope-from address`? Just like real-life mail, there is a envelope and the letter itself. The envelope is used by mail servers (post offices) to send the mail to the recipient, however the recipient will most likely get who the mail is from and for using information from the headers (the actual letter inside the envelope). In fact, a lot of email clients does not bother to show the envelope-from and envelope-to address, leading to higher risks. Thus it is normally not allowed to authenticate with one address and use another address for envelope-from, however a whitelist function is also normally in place. Consolidate documentations for this setting.

Read more [here](https://www.mailgun.com/resources/learn/glossary/email-envelope/) and [here](https://www.spamhero.com/support/120008/Envelope_Sender_Vs_Email_Header_To).

## Setup

Here you can choose where the server boots to, as well as does this setting persist or not. This could be useful if you're trying to boot into BIOS but wants to go to the loo real bad.

# iDRAC Settings

## Network Settings

### Network

#### DNS iDRAC name and static DNS domain name

This are the two components that actually make up the envelope-from address, as mentioned [here](#snmp-and-email-settings). The address will be DNS-iDRAC-Name@Static-DNS-Domain-Name. For example,

```
If
DNS iDRAC name = iDRAC-1234
and
Static DNS domain name = example.com

Then the envelope-from address will be
idrac-1234@example.com
```

By default the DNS iDRAC Name value will be IDRAC-<Dell Service Tag #>. Source: [Dell](https://www.dell.com/support/kbdoc/en-us/000176998/configuring-initial-idrac7-network-settings)

#### IPMI Settings

Enable IPMI Over LAN - mentioned previously [here](#ipmitool).

## User Authentication

Here you can change user passwords as well as upload ssh key files.

## Update and Rollback

This is not apparent, but the function to automatically search for firmware updates for your currently installed hardware does exist. For file location choose `HTTPS`, and for HTTPS address use `downloads.dell.com`. Wait for it to load, and then you can forget about finding all the update files one by one.

# Hardware

## Fan

The default fan profile is definitely quieter than older models, such that I can actually sleep with it. However, it can be even quieter.

Context: Testing is done with 2 * E5-2680 V4 CPUs at around 25 degrees celsius ambient.

### Basic static fan speeds

`ipmitool` is used here. How to use? Check [here](#ipmitool)

```
# print temps and fans rpms
ipmitool -I lanplus -H <iDRAC-IP> -U <iDRAC-USER> -P <iDRAC-PASSWORD> sensor reading "Temp" 

# print fan info
ipmitool -I lanplus -H <iDRAC-IP> -U <iDRAC-USER> -P <iDRAC-PASSWORD> sdr get "FAN1"

# enable manual/static fan control
ipmitool -I lanplus -H <iDRAC-IP> -U <iDRAC-USER> -P <iDRAC-PASSWORD> raw 0x30 0x30 0x01 0x00

# disable manual/static fan control
ipmitool -I lanplus -H <iDRAC-IP> -U <iDRAC-USER> -P <iDRAC-PASSWORD> raw 0x30 0x30 0x01 0x01

# set fan speed to percentage
ipmitool -I lanplus -H <iDRAC-IP> -U <iDRAC-USER> -P <iDRAC-PASSWORD> raw 0x30 0x30 0x02 0xff 0xXX
# 0xXX as speed percentage in hexdecimal. 0% would be 0x00, 100% would be 0x64, and so on.
```

Read more detailed usages [here](https://www.spxlabs.com/blog/2019/3/16/silence-your-dell-poweredge-server).

### Fan speed curve

The idea is to check CPU temps every other few minutes using cron, and set fan speeds according the CPU temperature at the time. For example, there is one solution here on [reddit](https://www.reddit.com/r/homelab/comments/x5y63n/fan_curve_for_dell_r730r730xd/).

However there is one problem with this approach as that in extreme circumstances CPU temperatures may rise too fast for cron to respond. Thus a response time of a few seconds is much more resonable.

### Static speed + automatic curve

There is this wonderful docker image on Github [tigerblue77/Dell_iDRAC_fan_controller_Docker](https://github.com/tigerblue77/Dell_iDRAC_fan_controller_Docker) that not only has a second-level response time, but also allows the bios curve to kick in if things get out of hand for the quiet low speed settings.

~~However, in my own testing, even with a fan speed of just 5%, my two E5-2680 V4s never exceeded 70 degress with max load.~~ **But do not take my word for it. Do your own testing for your hardware's safety.**

Turns out I have made a mistake, and the previous test method did not put enough stress on the CPUs.

### Faster responses

In the end I have decided to write a little script myself to regulate the fan speeds based on temperature thresholds. Find more about it in the post [Fancontrol Script for Dell iDRAC](/posts/fancontrol-script-for-dell-idrac).


# Storage

The only thing that needs to be mentioned here is the integrated RAID controller. More can be read in [this post](/posts/using-520-byte-sector-disks/).

# References & Sources
- [Dell: Dell PowerEdge: How Do I Change the System Host Name on the iDRAC?](https://www.dell.com/support/kbdoc/en-us/000141693/dell-poweredge-how-do-i-change-the-system-host-name-on-the-idrac)
- [Dell: Support for Dell iDRAC Service Module](https://www.dell.com/support/kbdoc/en-us/000178050/support-for-dell-emc-idrac-service-module)
- [Dell: Configuring Initial iDRAC7 Network Settings](https://www.dell.com/support/kbdoc/en-us/000176998/configuring-initial-idrac7-network-settings)
- [Dell: iDRAC7 and iDRAC8 SMTP email TLS Encryption settings](https://www.dell.com/support/kbdoc/en-us/000062035/psqn-idrac7-idrac8-smtp-email-tls-encryption-settings)
- [Dell: How to Configure Integrated Dell Remote Access Controller (iDRAC) Email Alerts](https://www.dell.com/support/kbdoc/ja-jp/000131098/dell-idrac-configuring-email-notifications-for-system-alerts-on-idrac7-8-and-idrac9)
- [ipmitool@Github](https://github.com/ipmitool/ipmitool)
- [Reddit: Fan curve for Dell R730/R730xd](https://www.reddit.com/r/homelab/comments/x5y63n/fan_curve_for_dell_r730r730xd/)
- [Reddit: Dell PowerEdge fan control with ipmitool - individual fan speeds](https://www.reddit.com/r/homelab/comments/t9pa13/dell_poweredge_fan_control_with_ipmitool/)
- [Reddit: Dell Fan Noise Control - Silence Your Poweredge](https://www.reddit.com/r/homelab/comments/7xqb11/dell_fan_noise_control_silence_your_poweredge/)
- [Reddit: Manual fan control on R610/R710, including script to revert to automatic if temp gets to high.](https://www.reddit.com/r/homelab/comments/779cha/manual_fan_control_on_r610r710_including_script/)
- [SPXLABS: Silence Your Dell PowerEdge Server](https://www.spxlabs.com/blog/2019/3/16/silence-your-dell-poweredge-server)
- [MYBLUELINUX.COM: What is email envelope and email header](https://www.mybluelinux.com/what-is-email-envelope-and-email-header/)
- [Spamhero: Envelope sender/recipient vs. email header From/To](https://www.spamhero.com/support/120008/Envelope_Sender_Vs_Email_Header_To)
- [Mailgun: Email envelope](https://www.mailgun.com/resources/learn/glossary/email-envelope/)
