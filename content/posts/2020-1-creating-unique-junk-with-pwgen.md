+++
title = "Creating unique junk data with pwgen"
date = "2020-01-06"
tags = ["shell scripting", "one-liner"]
+++

... when you need to generate many files with unique payloads ... enter [`pwgen`](https://formulae.brew.sh/formula/pwgen#default) and some shell scripting.

Though I use [1Password](https://1password.com) to manage access to accounts, I seldom use its password creation mechanism and instead generate and
(sometimes) augment the result of `pwgen` for that purpose.

I've also found `pwgen` useful for generating _many_ junk files with different payloads such that, say, they have different content signatures
as the result of various hashing methods. The following snippet shows the generation of these files:

```bash
for j in {1..5}; do mkdir $j && for i in {1..100}; do pwgen -n $(expr $j \* $i \* 1000) 1 > $j/$i.txt; done & done
```

...where the command uses pwgen to create a file at 1K, 2K, ... to 5K then multiplies that argument by 1-100 to generate a set of files with random
contents. This, of course, is not the most efficient way of achieving this, as you could simply have a prefixed-seed value for each size of file you need and concatenate a random value or counter to obtain different hash results.

Also, I really should upgrade to [sf-pwgen](https://formulae.brew.sh/formula/sf-pwgen#default).
