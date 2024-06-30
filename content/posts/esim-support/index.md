---
title: "How to use eSIM on a non-eSIM supported device"
date: 2024-05-24T17:18:00+02:00
description: How to create custom scheduler in Spring
menu:
  sidebar:
    name: Unlocking eSIM
    identifier: esim-support
    weight: 50
tags: ["eSIM", "HowTo"]
categories: ["Basic"]
---

While not all phones come with eSIM support, you can still enable this feature with a bit of tinkering. In this guide, I'll show you how to set up eSIM on phones that don't have it built-in, so you can take advantage of this convenient technology.

### What is eSIM?

An eSIM is a small chip embedded into a device’s motherboard that replaces traditional SIM cards. It allows for remote provisioning, meaning users can download and activate carrier profiles over the air using QR codes or activation codes provided by carriers. The device’s operating system and embedded Universal Integrated Circuit Card (eUICC) software manage the secure storage and execution of these profiles. The eSIM can hold multiple profiles, enabling easy switching between carriers or plans through the device settings without needing to physically change SIM cards. 


But wait, if eSIM requires an actual hardware embedded into motherboard, how are we going to use eSIM on devices that doesn't have this chip? This article aims to answer this question.

### eUICC comes into play

We mentioned eUICC (embedded Universal Integrated Circuit Card) above. Although, these eUICCs normally embedded into the motherboard, we can actually take a consumer ready eUICC chip, put it in the physical SIM slot and solve the hardware problem that way. But that's not enough. eSIM functionality requires specific software components within the device's operating system to manage eUICC profiles, handle remote provisioning, and facilitate secure communication with mobile networks. Phones that don't natively support eSIM lack these software features, so even if you were somehow able to insert an eUICC, the phone wouldn't have the necessary software infrastructure to utilize it.

### Consumer ready solution

There is already a company that tries to address this problem, "esim.me". They sell a physical SIM card, which you can then use with their mobile app to manage and install eSIM profiles. But they still don't support all the devices and require the phone to be compatible. I assume this is because they still require operating system to support OMAPI (Open Mobile API).

{{< img src="esimme.jpeg" height="410" width="260"  title="esim.me is not supported" >}}
</br>
<blockquote>
EDIT: With a new sotware update, my phone (Nothing Phone 2) now supports esim.me, but this tutorial is not Nothing phone specific, so if your phone is still not supported, it might still make sense to follow this tutorial.
</blockquote>

### DIY path

But worry not, there is still a way! You simply need to purchase a removable eUICC, insert it into a smart card reader connected to a PC, download an eSIM profile into the chip using a tool like EasyLPAC, and then reinsert the card into the device. Voila! Your device can now connect to the mobile operator you have configured! Sounds easy, right? RIGHT!?

This solution will require you to remove and connect your eUICC card to a PC every time you want to switch or manage eSIM profile. Yes, it is not ideal, but it works. So it is better than nothing. And there is actually an open source Android application to manage the eSIM profiles but I will come to that later.

### Why Though?

Btw, why are we even going through all these troubles for an eSIM? What is wrong with using physical SIM card anyway?

Nothing, it is just the convenience the eSIM brings. I want to travel a lot to overseas, using my local SIM package is not an option, and I don't want to buy SIM cards for every country I visit. I know there are also SIM cards that are valid in multiple regions, but they are quite expensive compared to eSIM profiles. You can store multiple profiles in the card and switch to any of the ones you want. While this solution may compromise some of the convenience of eSIM, it still remains beneficial as the profiles will be more affordable and eliminate the need to search for physical SIM cards. 


# Let's Get Started

##### 1 - We need eUICC and a card reader
There are probably multiple vendors out there but I came across [sysmocom](https://sysmocom.de) and decided to buy the hardware from them. They are the team behind [Osmocom](https://osmocom.org), a non-for-profit, community-driven project creating various FOSS projects related to mobile communications. They have really nice tools and documentation around eUICCs. 
I went ahead and purchased the [sysmoEUICC1](https://shop.sysmocom.de/sysmoEUICC1-eUICC-for-consumer-eSIM-RSP/sysmoEUICC1). Please note that I am not affiliated with Sysmocom or Osmocom, I just liked that they are developing these open source technologies and also providing these hardware at an affordable price.

They also sell card readers on their shop but I decided to buy that from a different online store. And my choice was Omnikey CardMan 6121, because of its small form factor. And I saw the same reader in the sysmocom shop, so it must have been compatible.

{{< img src="euicc1.jpeg" height="410" width="260" title="euicc and reader" >}}

##### 2 - We need a software to manage the eUICC
The [manual](https://sysmocom.de/manuals/sysmoeuicc-manual.pdf) from sysmocom mentions some tools to manage the chip. I decided to use [EasyLPAC](https://github.com/creamlike1024/EasyLPAC). EasyLPAC is a GUI Frontend of lpac, which is a C based eUICC LPA. It allows you to manage your eUICC Card using a PC/SC card reader through a graphical interface.

There is also an Android app mentioned in the sysmocom documentation, [OpenEUICC](https://gitea.angry.im/PeterCxy/OpenEUICC). It is an open source implementation of the eSIM LPA (Local Profile Assistant) function for consumer eSIM and runs on Android devices. It only support SM-DP+ whose TLS certificates are signed by the GSMA production CI, such as sysmoEUICC1-C2G.
I actually wasn't aware of this application before purchasing the card reader, so I could have saved some money if I knew it beforehand. 
Also this basically eliminates the need of removing the eUICC each time, so it is more convenient. But I still haven't tried this app, so I can not provide feedback on it.

##### 3 - We also need an eSIM profile to test the solution
Although there are test profiles, I wanted to test it with a real profile in my phone. I checked online and found a very cheap profile from a company called [jetpac](https://www.jetpacglobal.com). Again, I am not affiliated with this company, I just chose them because of the cheap price and good reviews.


##### 4 - Process

I've put the eUICC into the card reader, plugged it into my Windows 10 PC and then run the EasyLPAC application. It immediately detected the card reader and the chip information.
Then I went to `Profile -> Download` and entered the `SM-DP+` address and `Activation Code` provided by the eSIM provider. It took 1-2 mins for the download to be completed. Then I disabled the default profile the chip came with and enabled the newly downloaded profile.

{{< img src="easylpac1.jpg" height="391" width="570" title="easy lpac gui" >}}

##### 5 - Result

I have removed the card from the reader and inserted into my phone (Nothing Phone (2)). After a short time, I have received an SMS:

{{< img src="jetpac-activation-message.jpeg" height="220" width="400" title="easy lpac gui" >}}
<br />
I have activated data roaming and then:

{{< img src="profile-connected.jpeg" height="250" width="400" title="easy lpac gui" >}}
<br />

Everything worked perfectly fine.

I am now looking forward to my next trip to Asia to justify this setup :D 

