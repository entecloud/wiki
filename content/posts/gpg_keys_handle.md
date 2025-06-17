---
date: 2025-05-27T14:25:00+05:30
title: Handle GPG keys properly
tags: [gpg, security]
---

Ever stuck in a dilemma of how to manage GPG keys? Let's dive right in.

<!-- truncate -->

# Handle GPG keys properly

:::warning[Statutory Warning]

I made this document as I went along reading other people's best practices notes. **DO NOT** treat this documentation as the goto or final guidelines for GPG.

Always look for an updated articles about the topic to make sure everything is relavant as you read this.
:::

## Introduction

Before we start creating keys, we should know some basic terminologies, and stuffs.

I'd like to extend my sincere thanks to [/u/Saklad5](https://www.reddit.com/user/Saklad5/) for his [post on /r/GnuPG](https://www.reddit.com/r/GnuPG/comments/vjas2e/proper_key_management/).

This document is almost an extended TL;DR of the above post. But I'll be also making some more articles that are related to this.

### Primary Key

As the name suggests, this is the main GPG key and has a 1-1 relationship with UID

### User ID (UID)

This is going to be your email address. Create one primary key for one UID, and create multiple subkeys for signing and encrypting separately.

Have read that people use **one single** mother key for all the subkeys with a strong passphrase. This is easier to maintain, although I belive it'd be better to split into personal / work.

### Subkeys

A primary key can have one or more sub keys. It's better to have separate subkeys for separate functions.

PGP has mainly four functions.

- Signing
- Authentication
- Certifying
- Encrypting

**Signing and Authentication are fungible**[^1], which means it doesn't matter which key is used for the operation. We can interchange the key as long as it share the same primary key. Because of this, **we should generate new subkeys** for the specified purposes on **each device** you have. You don't need to keep copies of the keys across machines as they are replaceable because it has the same parent primary key. So ideally, if you stop using an old device, **revoke** it accordingly.

Now for the other two functions, **Certifying and Encrypting are not fungible**[^1]. We've to use the exact key for the operation and can't be replaced with any of the subkeys. Treat this keys as **most important and sensitive**. These keys have to be shared across devices, and should be locked with strong passphrases.

## Managing Keys

### Key Capabiltities

By default, GnuPG will generate a primary key for signing and certifying, and a sub key capable of encryption. As we learned about subkeys a while ago, it's better to avoid this behaviour. We shouldn't sign anything with our primary key. Also, it's better to not generate encryption keys unless we have an actual use case.

If you generate the key with `--full-generate-key`, there's a menu which will let us set the capabilities. If you want to change the default behaviour, alter the gpg conf with below setting.

```config
default-new-key-algo ed25519/cert
```

Here we're setting the default algoritm to **ED25519** and the default capability to **Certify**

### Key Expirations

I always had a habit of setting the keys to never expire all the time (I also had a habit of creating several primary keys for several UIDs just to sign commits). But we're going to put a stop to that habit.

:::danger[Set expiration date for all keys!]
**Every single key SHOULD have an expiration date especially primary keys**.

This sounded excessive at first for me, because I always thought this "expiration" in the context of access tokens, PATs etc. Little I knew I misunderstood the meaning of expiration.
:::

An expired key **should not be considered as invalid**. It just merely means that the key is **outdated** and in need of a refresh. You can _always_ keep renewing the expiration dates as needed.

Expiration dates are for the copies _everyone else_ has. We set an expiration dates so that others can know whether they key they got is outdated or not.

### Key Recovation

Now this is what you need to **invalidate** the key. Functionally, it's similar to the expiration we discussed abouve, except this is a one-way operation. Once you revoke a key, it cannot be reversed. While the definition sounded easy, here as well, there's a slight difference in meaning.

Expiration indicates a key should / has to be updated but revocation means that **a key should not be used again**. The **important** point here is that key recovation includes a reason. Unless the key is revoed due to compromise, **the key should still be considered trustworthy**.

We can still treat a signature as **valid** even if the key got revoked later. Of course, if the signature is newer than the recovation date, stay away.

- Always revoke keys if you're not intending to use
- If you're transitioning to new key, revoke the old one but put the new key's fingerprint as the recovation comment so that others will know what happened.

## Distribution of keys

I'm not gonna lie, for all the time now I never thought about this. I always took the bakups via `--armor` and the `~/.gnupg/` (told you I created several primary keys unnecessarily). Sure, I was aware of the services like [Keybase](https://keybase.io) and [OpenPGP Key directory](https://keys.openpgp.org) but never used those.

In GnuPG, there's an option to set a preferred keyserver, and you can set it to a URL where someone can get the key. The advantage of using a keyserver would help anyone who has the copy of your key. They can trivially get the newset copy, with complete recovations, additions etc.

:::tip[Resolving key server automatically]

```bash
--keyserver-options honor-keyserver-url
```

If you pass the above options go gpg command, it'l auto resolve the keyserver url.
:::

If we set the preferred keyserver in the setting, the signatures we create will also include the keyserver url. With this and assuming GnuPG set to auto retrieve keys, running

```bash
git log --show-signature
```

will make git to autofetch the current key and validate the commits according to the trust system of the local keyring.

### Web Key Directory

As we mentioned about [Keybase](https://keybase.io) and [OpenPGP Key](https://keys.openpgp.org), will help us and others to find a key of a given user ID or email.

If you own a domain, and the UID is based on that domain, you can actually host a WKD without any cost with the help of GitHub pages. But more on to that later. I don't want to extend this article beyond the basic explanations of GPG and the other related stuffs.

And of course, I'll be creating a step-by-step to create gpg keys as well as hosting a WKD on our own in next blogs.

Thanks for spending nearly 6 minutes to read this article. I seriously hope this did help you to an extend.

Bye!

[^1]: [Fungible ADJ](https://www.dictionary.com/browse/fungible)
