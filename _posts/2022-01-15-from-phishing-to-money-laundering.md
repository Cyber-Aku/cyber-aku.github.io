---
layout: post
title: From Phishing To Money Laundering
tags: OSINT Phishing
---
Hello World, for my first blog post I looked at wide-spreading phishing attacks (to be specific smishing) in Iran, I’ve tried to understand what’s going on behind phishing attacks.

## Be a victim

To start out my research, I should have either an SMS or a link, sadly I don’t have either of them, so I use my best friend Google to find some active pages.

It’s judicial services phishing page.
so with simple Google search, you can find some fake ones.
and I’ve got this active phishing page

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/webpage-1.png" width=750 >
</div>

After I’ve got a page, I tried to catch more pages using this sample that I already found, I use some common techniques Like:

- Favicon hash search
- HTML hash search
- File names search

I don’t want to explain this part in detail because Maciej Makowski already has an awesome [post](https://www.osintme.com/index.php/2021/12/06/how-to-investigate-a-massive-phishing-campaign/) about it.

To give you an idea of how was the result come out, this is just searching the banner image SHA-256 in urlscan :

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/webpage-0.png" width=750>
    <figcaption>These domains are likely to be suspended by the time of publishing</figcaption>
</div>

Almost all of these page gives you an APK file.

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/webpage-2.png" width=750 title="Download APK button">
</div>

## Malicious APK

Let’s be positive for a second and assume it’s not malware and check it on the [VirusTotal](https://www.virustotal.com/gui/file/e8601caa5bdb4a64c3a95824d6a802407e53b04a0616261ad27466a032691367):

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-0.png" width=750 >
</div>

Nope, it’s really malicious.
I’ve also checked it on [hybrid analysis](https://www.hybrid-analysis.com/sample/e8601caa5bdb4a64c3a95824d6a802407e53b04a0616261ad27466a032691367/61e0aa65147378792346de28):

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-4.png" width=750 title="Hybrid analysis result">
</div>

Now let’s check what happens if someone installs it on their device.
After installing the malware, attackers will get a message:

Screenshots don’t belong to the same cases

<div style="text-align: center">
<img src="/assets/images/from-phishing-to-money-laundering/malware-1.png" width=750 >
</div>

A copy of all incoming SMSs will get sent to the attacker:

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-2.png" width=550 >
    <figcaption>An SMS containing OTP received from the bank</figcaption>
</div>


<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-9.png" width=550 >
</div>

<br>

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-8.png" width=550 >
    <figcaption>A message which claims the app uninstalled itself from the phone (it is a self-hiding behavior in malware)</figcaption>
</div>


Credit card information :

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-3.png" width=750 >
</div>

Now its time to spread :

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-5.jpg" width=350 >
    <img src="/assets/images/from-phishing-to-money-laundering/malware-6.jpg" width=350 >
    <img src="/assets/images/from-phishing-to-money-laundering/malware-7.jpg" width=350 >
</div>

This person is got a crazy number of SMSs. and all of them are from private numbers belonging to the infected users.

You may ask yourself where is this malware come from?
They buy it! not everyone can develop malware, but everyone can buy an existing one.

There are different groups that sell you the malware in different templates

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/malware-10.png" width=550 >
</div>

All of these are normal, the interesting part is how they transfer money (at least interesting for me).

## Money laundering

I know it is exaggerated to say money laundering for money transfers on this scale, but I don’t have any alternative.

So how do they transfer money?
The answer is: it differs. but the goal is the same, make money untraceable and clean.

In the example below, the attacker transfers money to an account (a victim account owned by the launderer) then the laundering process starts.
eventually, the attacker gets his money as TRON or Tether, and the launderer got his share of the money.

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/launder-0.jpg" height=600 >
    <img src="/assets/images/from-phishing-to-money-laundering/launder-1.jpg" height=600 >
</div>

During my research, I’ve seen a group that accepts up to 1,500,000,000 Rials.

And sometimes attacker might use their access to buy credit codes for prepaid SIM cards then they can use it themselves or sell it to others for Recharging purposes.

<div style="text-align: center">
    <img src="/assets/images/from-phishing-to-money-laundering/money-1.png" width=350 >
</div>
