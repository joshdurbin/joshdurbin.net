+++
title = "My personal privacy posture"
description = "Details of my desktop, mobile, and cloud security and privacy setup"
date = "2018-10-08"
tags = ["privacy"]
+++

Updated: October 11, 2018

This is a privacy post about my efforts to regain control of my digital privacy.
I won't answer why, you can derive that from my opening statement, but I will reference a mountain
of documentation on steps you can take to enhance your digital privacy. More than anything
the work outlined in all this highlights my attempt to limit the data I share with companies like: 
Google, Facebook, Amazon, Verizon, ATT, etc... all while remaining minimally engaged with their services. 

First off, I highly recommend stepping through [privacytools.io](https://www.privacytools.io/) and [Restore Privacy](https://restoreprivacy.com). Their content is **very**
informative and approachable by providing actionable steps, recommendations as replacements
for services we're all expected to use everyday ("let me google that for you...").

Let's jump into my posture and recommendations...
   
### Global Considerations

- Keep devices locked at all times whenever you're not present. 
- Keep all systems up-to-date with security patches, OS upgrades, etc...
- Enable network-wide adblocking via the [pi-hole](https://pi-hole.net) project. Use a low-power, always-on device like
a [Raspberry PI](https://www.raspberrypi.org/) to run pi-hole and configure your home router's DHCP settings to hand out a static
address to the RPI running pi-hole. Customize the DHCP settings to direct all clients on the network
to use the pi-hole for their DNS queries. This allows pi-hole to keep stats on your network
requests, use block lists to flat out ban requests to ad-serving content providers, and otherwise
further secure your network.
- Use [encrypted DNS](https://en.wikipedia.org/wiki/DNSCrypt) servers, ideally those that don't track you,
to protect the privacy of the domains you request. DNS queries, different than the traffic itself, is generally unencrypted and allows
ISPs, governments, etc... to keep a log book of what systems a device talks to (unless those too are encrypted).
Read more about that topic [here](https://arstechnica.com/information-technology/2018/04/how-to-keep-your-isps-nose-out-of-your-browser-history-with-encrypted-dns/). I use [Quad9](https://en.wikipedia.org/wiki/Quad9) for unencrypted DNS queries
and [okturtles](https://okturtles.org) for encrypted DNS traffic. See [this](https://github.com/dyne/dnscrypt-proxy/blob/master/dnscrypt-resolvers.csv)
CSV for a list of dns resolvers. (Alphabet's Jigsaw [just released an app](https://techcrunch.com/2018/10/03/googles-cyber-unit-jigsaw-introduces-intra-a-security-app-dedicated-to-busting-censorship/amp/) to prevent this for mobile
users, specifically in countries with governments that engage in censorship.)
- Enable disk and cloud encryption of everything, preferably in a way where you manage your own keys. See [this](https://en.wikipedia.org/wiki/Comparison_of_disk_encryption_software) comparison of software options.  
- Use **absolutely no** [OAuth](https://en.wikipedia.org/wiki/OAuth) to cloud providers like Amazon, Google, Facebook, Twitter. Read [this](https://lifehacker.com/5918086/understanding-oauth-what-happens-when-you-log-into-a-site-with-google-twitter-or-facebook)
to fully understand what information is shared when you authorize an application or service to authenticate
via OAuth.
- Use [DuckDuckGo](https://duckduckgo.com/) as default search platform, or similar.
- Enable [two factor authentication](https://en.wikipedia.org/wiki/Two-factor_authentication) for all the sites/apps/systems you use
that [support it](https://twofactorauth.org/)!
- Leverage password management via [1Password](https://1password.com/), [bitwarden](https://bitwarden.com), [LastPass](https://www.lastpass.com), [Keepass](https://keepass.info), etc... with
the strongest possible passwords for each site and **absolutely zero** password re-use.
- Leverage secure messaging platforms like [Keybase](https://keybase.io), [Signal](https://www.signal.org), and [Telegram](https://telegram.org) wherever possible.

### iOS

- Use a complicated pin to unlock your device
- Understand how to engage the "lock out mode" on your device without looking at it; while it's
in your pocket, for example (by tapping the side button 5 times, with the proper minimum version of iOS)
- Limit app permissions for camera, location, contact, background processing tailoring the permissions
to your usage of any individual app 
- Use an ad-blocker with Safari content blocking integration like [AdGuard Pro](https://adguard.com/en/adguard-ios-pro/overview.html)
In addition to content blocking, AdGuard Pro also has a local, pass through VPN which aims to
achieve something similar to the aforementioned Jigsaw's Intra app. With the VPN enabled; all traffic
flows through the app which uses configurable blacklists to block outbound DNS requests (similar to pi-hole).
Also similar to pi-hole, AdGuard Pro enables custom upstream DNS support enabling you to pick
DNS endpoints for your device (encrypted or non-encrypted). I use [just domains](https://github.com/justdomains/blocklists)
as the basis for the majority of my block lists.
- Use [Firefox Focus](https://www.mozilla.org/en-US/firefox/mobile/) for super sensitive queries
- Use VPN services like [encrypt.me](https://encrypt.me/) to provide LTE and WiFi VPN connectivity on untrusted networks.
- For more tips read through [iOSPriSec](https://github.com/harleo/iOSPriSec) (Useful tips on how to maximize and balance security and privacy on iOS)

### MacOS

- MacOS Mojave added many privacy-focused features including permission prompts for audio
and video input. In addition to these features, I use [Micro Snitch](https://obdev.at/products/microsnitch/index.html) to keep
tabs of what and when apps request mic access.
- MacOS has a built in firewall which you should enable.
- Use [Little Snitch](https://obdev.at/products/littlesnitch/index.html) to keep tabs of what
connections applications are making **outbound** and create open, closed, or policies somewhere
in the middle.  
- FileVault should be enabled **without** iCloud recovery enabled, using the only the generated
recovery code as fallback (don't lose it!).
- Don't use Find / Locate my Mac and or the remote wipe functionality. 
- Use [Firefox](https://www.mozilla.org/en-US/firefox/new/) for primary browsing hardened
security settings including most of those referenced [here](https://www.privacytools.io/#addons).
Many of the recommendations in the aforementioned article **and more** can be dropped in by default
with this amazing tool, [ffprofile](https://ffprofile.com/). The specific list of Firefox add-ons I use includes:  
    - [Privacy Badger](https://www.eff.org/privacybadger)
    - [uBlock Origin](https://addons.mozilla.org/firefox/addon/ublock-origin/)
    - [Cookie AutoDelete](https://addons.mozilla.org/firefox/addon/cookie-autodelete)
    - [HTTPS Everywhere](https://www.eff.org/https-everywhere)
    - [Decentraleyes](https://addons.mozilla.org/firefox/addon/decentraleyes/)
    - [Don't track me Google](https://github.com/Rob--W/dont-track-me-google)
    - [1Password](https://1password.com)
- Make use of [Multi-Account Containers](https://support.mozilla.org/en-US/kb/containers) to group and separate sessions. My default
 container isn't authenticated with Google, Facebook, LinkedIn, Twitter, etc... I do use more
 global services/sites that I have fewer if any privacy or tracking concerns about, like "[github.com](https://github.com)",
 within the default container. I tend to break up session sharing into the following categories:
    - personal google
    - work google
    - commerce
    - finance
    - social (Facebook, Twitter, LinkedIn, Instagram, etc...)    
- Occasional usage of Safari for sites that are uncooperative with Firefox.
- Chrome usage for things like Drive, Google Cloud, Hangouts, when appropriate.
- Both Safari and Chrome have the aforementioned list of applicable plugins installed/configured
if available (mainly in the case of Safari). 
- [Crashplan](https://www.crashplan.com/en-us/) cloud backup with self-managed encryption keys.
- For more tips read through [drduh's](https://github.com/drduh) [macOS security and privacy guide](https://github.com/drduh/macOS-Security-and-Privacy-Guide)
  
### Cloud Considerations

- Personal, custom mail/calendar/contacts infrastructure via [sovereign](https://github.com/sovereign/sovereign) or, even easier [mailinabox.email](https://mailinabox.email/). I use sovereign but my [fork](https://github.com/joshdurbin/sovereign) is highly customized. 
- Encrypt any sensitive data at rest and transit -- **do not** trust Box or Dropbox with unencrypted data.
- Limit use of cloud applications like those in G suite, or do so via paid accounts with differing terms from free accounts
- **Always** read terms of service, terms of use.

### Application Engineering Considerations

- Stay up to date with [recommendations and publications](https://en.wikipedia.org/wiki/OWASP) by the Open Web Application Security Project (OWASP).
- Prefer portable encryption solutions ([vault](https://www.vaultproject.io/)) over those offered by cloud providers ([google](https://cloud.google.com/kms/) or [aws](https://docs.aws.amazon.com/kms/index.html))
- Run applications using the fewest privileges possible, particularly in cloud scenarios.
- Run regular [metasploit](https://www.metasploit.com) scans of your infrastructure and applications.  
- Use a bug bounty program like those offered by [hackerone](https://www.hackerone.com).

### Closing Remarks

Many large, open source projects exist only by the time and resources of individuals who care
deeply about its mission. Consider donating your time or resources to the projects you use
most to keep those projects alive and well for a continued open and free internet.

### See Also

- [DNSChain + okTurtles](https://okturtles.org/other/dnschain_okturtles_overview.pdf) 
- Projects and general thoughts by [Patrick Wardle](https://twitter.com/patrickwardle) at [Objective-See](https://objective-see.com/products.html) are interesting.
- [Internet censorship circumvention](https://en.wikipedia.org/wiki/Internet_censorship_circumvention)
- [Privacy concerns with social networking services](https://en.wikipedia.org/wiki/Privacy_concerns_with_social_networking_services)
- [Privacy concerns regarding Google](https://en.wikipedia.org/wiki/Privacy_concerns_regarding_Google)

This document will be updated with useful information as I find stumble upon it.     
