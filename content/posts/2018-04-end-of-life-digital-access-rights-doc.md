+++
title = "Authoring an end of life digital access instruction document"
date = "2018-04-12"
tags = ["legal"]
aliases = [ "/blog/authoring-an-end-of-life-digital-access-instruction-document/", "/blurbs/authoring-an-end-of-life-digital-access-instruction-document/"]
+++

The other day I established my Last Will, Living Will, and Power of Attorney
after not having one in place, ever. Though I put these docs in place
relatively late in _my_ game, I have thought
about end of life activities before, but mainly limited in scope to digital
access. Specifically, how to access my encrypted computer, encrypted cloud
backups, and encrypted drives stored on premises where I reside when something happens.

I wanted to share what I came up with for feedback or, even, to help anyone else
come up with a strategy for themselves. I have not consulted an attorney for this work,
so there might be bullet holes, but this document is really meant to be more instructional
for trusted individuals rather than a legal document.

First let me highlight my personal machine backup strategy:

- I use a cloud backup provider for selective backups which are entirely encrypted
with a key I maintain
- I have an encrypted platter drive that runs Time Machine backups on my main
laptop
- I have an encrypted solid state drive that runs rsyncs about once-a-month
and resides within a fire safe, in my apartment

The intent of the document is to give information to my family such that they can
unlock these local drives and use that data to get access to my password vault, cloud
backups, etc... Caveats are listed below...

This how the document reads:

> [Fullname and address]

> [Date]

> [Recipient Fullname and address]

> [Intro to Recipients -- Dear xxx]

> I, [Fullname, initialed], am writing you on [Date] to inform you of the methods established for next of kin access to my data, accounts, and systems in event that I am unable to make decisions or actions on my behalf for a reasonably extended period of time or in the event I am deceased. This information document is referenced in and supplementary to my last wishes.

> This letter will additionally be sent to:

> [List of additional recipients]

> The aforementioned individuals, the collective you, will open and maintain active communication in the event you act upon the information in this document so as not unintentionally or intentionally cause data loss. Additionally, the following people should be granted access to data:

> [List of individuals granted access to data but not recipients of this letter]

> Due to the variable and expansive changes around access rights for next of kin, if you are to ever leverage this document and/or its information, it should be done so as an impersonation of me. There are no explicit agreements between myself and my service providers, accounts, etc...

> A particular on premise, encrypted device resides at my current residence labeled [Device visible name]. The device can be found within the current residence by [Describe location]. The device is encrypted via [Describe encryption method].

> This device maintains a monthly synced copy of all my data including unencrypted exports of my password vault.

> The drive is an encrypted [Describe drive decryption methods]. The decryption key/password for the volume, [Device visible name] is:

> [Decryption key/password]

> Keep this letter in a safe place. Please do not scan or digitize the letter.

> Sincerely,

> [Fullname]
> [Full address]

Some broad assumptions and bullet holes in this strategy are:

- Trust in family -- This is what at paid attorney is for; they hold the keys
and your wishes on how to distribute the information. I'm cheap. Ha.
- If no one else has cloud backup encryption keys and you rely on physical access
to physical devices, you're assuming that they'll be obtainable -- not destroyed,
not stolen, etc...

Either way, when considering end of life access to digital records, it makes sense to
both appoint someone who can manage your digital estate **and** give said individual(s)
access credentials to your accounts.
