---
date: 2026-02-01
title: "The path to pwning the PS VR2 (part 1) - \"Recovery mode\""
description: "How a fatal mistake in the PS VR2 authentication code leads to us discovering a way to enter a \"recovery mode\", allowing us to downgrade the PS VR2 to any firmware."
author: whatdahopper
---

# Preface
I am not a security researcher, there are people who are far better than me that are more knowledgable than me on things such as exploiting bugs inside poorly written code. All of this is "for fun" and purely learning experience for myself and others.

# The story
So, for months, I had been attempting to research potential entry points for jailbreaking the PS VR2 for months, and found nothing. The context is that, I had been wanting to enable more features on PC, such as eye tracking camera frames and headset vibration, having these features on PC would open the door for so many features and possibilities, but alas, it seemed impossible.

While we haven't achieved these goals just yet, this is one of the stepping stones we believe is required to get us to that goal.

# Something changed
On Oct 17th, 2025, I had decided to [download the latest PS VR2 Linux kernel source available from Sony](https://www.playstation.com/en-us/oss/ps-vr2/linux-kernel/) and from there I identified that all of the kernel code that Sony wrote for the PS VR2 was in a folder called `kern_module` and quickly identified that all of the USB communications for the PS VR2 were handled by the `sieusb` kernel module.

Upon first glance, nothing really caught my eye, I mean, it was clear that it was using standard Linux USB gadget code. I thought it was again, a dead-end and that I was wasting my time as usual. That is, until I started looking at `authentication.c`, and something caught my eye...

# Sony was doing cursed stuff for USB communication
I guess in an effort to curb potential attack vectors, Sony had decided to make it so that all USB interfaces were not handled by the kernel, but *instead*, by user-mode processes on the PS VR2. I can only speculate that this was to reduce the attack vector heavily, so that if there is any memory being leaked/written to by USB requests, that it will only have access inside the process itself, rather than access inside the kernel.

However, there is one USB interface that isn't *entirely* handled by user-mode processes, the PS VR2 has a USB interface called "Control". The usage of this USB interface is pretty interesting, in an effort to prevent cloned/fake hardware from being used on PS4/5, Sony requires that periphereals have authentication, so that the console may verify if it is allowed to be used and the PS VR2 is no exception. **The entire authentication process requires keys that only Sony is authorized to use, usage of these keys not authorized by Sony is a legal grey area we do not want to touch or endorse, nor are we interested in, at all.**

Well, as it turns out, all of the USB communications and request handling for this USB interface are handled by `authentication.c` (which is kernel-level code), and I started looking closely at the code, it did not take me long to find a major fuckup by Sony... As I was reading the code inside the `usb_auth_set_auth1_data` function I noticed that they were allocating `auth1_data` into the stack (which is a 64 bytes structure), and then clearing it using `memset` (with the size of the structure, 64 bytes), and then... **Oh, Oh No.** ***This is bad.***

I'll let the poorly written (kernel-level) code I saw speak for itself:
```
static int usb_auth_set_auth1_data(struct usb_request *req, struct usb_auth_interface *auth)
{
	struct usb_auth1_data auth1_data;
	int ret = AUTH_SUCCESS;
	int length = 0;
	unsigned long flags;

	GADGET_AUTH_LOG(LOG_DBG, "start set auth1 data.\n");

	length = req->length;
	memset(&auth1_data, 0, sizeof(auth1_data));
	memcpy(&auth1_data, req->buf, length);
	dump_auth1(&auth1_data);
```

Do you see the problem? They're clearing the the structure on the stack with the correct structure size, then they're... **copying the USB request buffer data onto the structure and blindly trusting the USB request buffer length, with zero checks at all**. This means that a USB request buffer with a size larger than 64 bytes **could potentially write onto the stack** (stack overflow), I don't know how Sony missed this, but for me, this was the breakthrough I wanted for months, and I felt like I had something on my hands. So, I started contacting friends who were more versed in exploiting devices through poorly written code, so I could get their opinion on the exploitability of this potential vulnerability.

# Lost hope
While contacting friends who I thought would be more versed in this field than I am, almost all of the friends I contacted told me **"it probably wouldn't work if the max USB request buffer length was the size of the structure"**, which, unfortunately, it was the size of the structure (64 bytes). I thought this was the end, as surely, both the device and host would deny sending the USB request if it exceeded that length, and that all the hype I had for this potential vulnerability was for nothing. I did not bother implementing it initially for this reason, as I didn't want to waste anymore time and I very quickly lost hope.

Regardless, I sent my findings off to [Supremium](https://github.com/RealSupremium) (PSVR2Toolkit developer) and told him that it probably wouldn't work from what my friends told me, I mean, *logically*, it made sense that the device and host would deny sending a large USB request that exceeds the maximum length of 64 bytes. Why should I have doubted that logic? Given that I had never worked with low-level USB communications, it made sense to me, at least.

# Something amazing happened
Despite me telling [Supremium](https://github.com/RealSupremium) that it may not work, he went ahead and attempted to implement the vulnerability, and sure enough, on Oct 19th, 2026, something amazing happened.

<img src="https://github.com/BnuuySolutions/BnuuySolutions.github.io/blob/main/images/something_amazing_happened.png?raw=true">

Against all odds, somehow, not only does the host not check the maximum USB interface request buffer length, neither did the device. I was blown away, my assumptions were wrong, and I can't blame my friends for being wrong either, this is some really odd behavior.

# "Recovery mode"
While researching other potential vulnerabilities with [ShinyQuagsire](https://github.com/shinyquagsire23) in November 2025, after we found this stack overflow exploit, we found that Shiny's PS VR2 ended up somehow running v01.10 (initial PS VR2 firmware) while bruteforcing stack overflow bytes, which we found extremely odd.

We decided to figure out how this happened, and it wasn't long after that [Supremium](https://github.com/RealSupremium) found out that if you constantly try sending a single byte stack overflow which crashes the device as soon as it is connected, right on the Sony logo, and then proceed to unplug and replug the device and, and do it a few times, that the device will enter a "recovery mode", which runs v01.10.

We had previously thought that the PS VR2 used [eFuses](https://en.wikipedia.org/wiki/EFuse) to prevent downgrading firmware, but you can imagine our shock to when we found out that once in "recovery mode", that we were able to then upgrade to any firmware, despite previously being on a far newer firmware. By replacing the ".CUP" (Caesar Update Package) file inside the PS VR2 app for PC, we could theoretically downgrade to any firmware available, including more exploitable firmwares, such as v06.00.

# Final thoughts
So, today, we are releasing this stack overflow exploit for everyone, which will let you downgrade your PS VR2 to any firmware. We recommend downgrading to v06.00 and staying on that firmware, as it is more vulnerable and will be very useful in the future, as we intend on updating the exploit program in the future, hopefully. Though, you can always downgrade to a firmware lower than that and then upgrade to v06.00 at any time far more easily than downgrading again.

You can download the exploit program (called "vr2jb") from [GitHub releases](https://github.com/BnuuySolutions/vr2jb/releases), please make sure you read the release notes attached for instructions and more information on the program itself.

Our hope is that by providing this exploit publicly, we can get more people interested in trying to jailbreak the PS VR2. Although this isn't a full PS VR2 jailbreak yet, we hope this pushes things into the right direction.

**NOTE: We are not responsible for any hardware damage that may occur, use at your own risk! Although we haven't had any hardware break by using this exploit, we cannot guarantee others won't have issues.**

# Credits/Thanks
These are the people who have helped thus far with jailbreaking the PS VR2, or people who I think deserve a shoutout:
- [Supremium](https://github.com/RealSupremium)
- [ShinyQuagsire](https://github.com/shinyquagsire23)
- tomoeko
- alula
