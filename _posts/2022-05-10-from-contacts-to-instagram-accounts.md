---
layout: post
title: From Contacts To Instagram Accounts
tags: OSINT Contact-Exploitation
---
Hello everyone hope you’re doing good.

I was watching some Trace Labs videos on YouTube and came across [this video](https://www.youtube.com/watch?v=bz4oZBR3LEk) by “Mishaal Khan” in which he explains a new technique in OSINT called “contact exploitation” which I never heard of before.
He uses some emails to find their corresponding LinkedIn profile.
and it worked!

So I want to try it for Instagram.

<div style="text-align: center">
    <img src="/assets/images/from-contacts-to-instagram-accounts/meme.png" width=550 >
</div>

 If you are an Instagram user, you come across suggestions before; what it actually does, is after getting the permission to upload your contacts, it looks for accounts with associated contact information to suggest to you.

Because it thinks if someone is in your contacts, you probably know him, right?

So for starting out we need a fresh sock puppet account and some emails or phone numbers.

After you’ve created your IG account (I highly recommend a new one) go to `settings > manage contacts` to see your contacts, it should be something like this:

<div style="text-align: center">
    <img src="/assets/images/from-contacts-to-instagram-accounts/synced.png" width=750 >
</div>

So, the next step is uploading our target emails and phone numbers or either of them. </br>
I’ve created a simple python script using the “Instapi” library which helps us to access Instagram’s private API.

After installing “Instapi” with the below command, you’re ready to run the script.

`pip install instapi`

To get the script to work, you need to convert your information to JSON and I have two recommendations :

- Mix your target information with some fake information and sock puppet accounts you already know their account.
- Submitting emails and phone numbers as separate contacts gives you better results.

<div style="text-align: center">
    <img src="/assets/images/from-contacts-to-instagram-accounts/contacts.png" width=750 >
</div>

You can get the code from this [gist](https://gist.github.com/Cyber-Aku/610074aa35c7da81623301edfc38f63b#file-ig_cntct_explt-py).

You should get this as output:

`{'items': [], 'users': [], 'status': 'ok'}`

After waiting some time (I suggest if it’s not synced after 2~3 hours try running the script again), if you check your contacts in the setting, you’ll see new contacts added, and make sure your target is also added:

<div style="text-align: center">
    <img src="/assets/images/from-contacts-to-instagram-accounts/synced-1.png" width=750 >
</div>

So now it’s time to find out if we catch something or not.

To check if we get something or not, I personally go to some private page or a public page with 0 posts, then I scroll to the bottom and it shows me the accounts (I don’t know its the appropriate way or not, but it works).

<div style="text-align: center">
    <img src="/assets/images/from-contacts-to-instagram-accounts/suggestions.png" width=750 >
</div>
