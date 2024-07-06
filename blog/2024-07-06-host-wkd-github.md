---
title: How to host your own WKD using GitHub Pages  
description: This step by step guide will walk you through to host a minimal WKD setup using Github pages.
slug: wkd-github-pages
authors: [giri]
tags: [encryption, gpg, linux]
---

A Web Key Directory (WKD) will help us to distribute our keys across users. In this article, we'll try to setup our own WKD page.

<!-- truncate -->

## What is WKD?

A Web Key Directory (WKD) is a way to share your keys on the internet. Publishing keys to a WKD has an advantage as most of the clients (like mail clients) will auto resolve the public key by looking up the WKD. 

If we don't publish our keys, usually, we'll have to send the public key along with the payload so that the end user can verify the authencticity on their own. 

## How WKD Works?

For the example, let's set the UID / email address to `you@example.org`

Upon receving the signed content, the mail client (or any gpg client if configured properly) will check for the existence of WKD. The WKD URL is derived from the domain part of the UID. 

The WKD discovery is done by two methods:

- Advanced Method  (First attempt)
- Direct Method (Fallback. If advanced method fails)

Will explain these two methods later, when we start setting up the WKD.

## Deciding WKD Method  

Depending on how we want the clients to lookup the keys, we can set up WKD appropriately. The file tree structure for both methods are different. 

### Advanced Method

:::warning[DNS changes are required]

This method will involve tweaking of your DNS entries. You are required to have a CA-signed `openpgpkey` subdomain to get this method work.  

:::

This is the first WKD discovery method which will the client use. It'll check for the presence of a file named `policy` under the path

```
https://openpgpkey.example.org/.well-known/openpgpkey/example.org/policy
```
Replace the `example.org` with your domain. 

The file structure for this method will look like this:
```
wkd_root/
└── .well-known/
    └── openpgpkey/
        └── example.org  
            ├── policy
            └── hu  /
                ├── hashed user id 1
                ├── hashed user id 2
                └── ...
```


If the `policy` is found, it'll try to download the public key from:

```
https://openpgpkey.example.org/.well-known/openpgpkey/example.org/hu/<userid_hash>?l=user
```

Now all we have to do is copy the public keys (as binary not ASCII armored) to the `/hu` folder with the hashed user id as the file name (no file extension required). 

**We will be using this method with GitHub pages in this doc.**

### Direct Method  

:::warning[Should have access to the Web Server]

Although it's relatively easy to setup this method when compared to advanced method, you should have the access to the webserver of the domain you have. 

In other words, there should be a live web server that listens to your domain up and running and you can add/edit the files under the webserver's disk. It must also support HTTPS. 

Also there shouldn't be a `openpgpkey` subdomain for the root domain as well. If you're using a wildcard certificate for the root, you need to add an empty TXT record for the `openpgpkey` subdomain. 

:::

This is the fallback method that client will use if it failed to locate the policy using advanced method as we discussed above. In direct method, the client will try to locate the policy under the path:

```
https://example.org/.well-known/openpgpkey/policy
```

The file structure for this method will look like this:  
```
webserver_root/
└── .well-known/
    └── openpgpkey/
        ├── policy
        └── hu  /
            ├── hash_id of key 1
            ├── hash_id of key 2
            └── ...
```


If the `policy` is found, it'll fetch the key from:

```
https://example.org/.well-known/openpgpkey/hu/<hashed_userid>?l=user  
```

In this method, if you use any reverse proxy or anything, you have to ensure it's properly setup. For example, 

- Should **disable** directory listing
- Should have the **right CORS** headers  
- The Content-Type must be `application/octet-stream`  

For example, the Apache config would look something like this:

```config  
<Directory "/.well-known/openpgpkey/hu">
  Options -Indexes
  ForceType application/octet-stream
  Header always set Accesss-Control-Allow-Origin "*"
</Directory>
```

For nginx, 

```config
location /.well-known/openpgpkey/hu/ {
  default_type "application/octet-stream";
  add_header Accesss-Control-Allow-Origin * always;
}
```

## Creating a WKD  

:::info[Managed WKD service]

Although this document covers how to setup a WKD service on your own, if you want to keep things simpler, you can always use [WKD as Service](https://keys.openpgp.org/about/usage#wkd-as-a-service) from the **keys.openpgp.org**. You just have to upload your keys to **keys.openpgp.org** and set a CNAME record on your domain. 

:::

Since we're going to use GitHub pages, let's create an empty folder and initialize git. 

```bash
mkdir github-wkd
cd github-wkd
git init  
```
Since we're going to use the advanced method, we'll need to create the file structure as mentioned above. If you're using and older version of GnuPG (`<2.2.12`), you'll need to set up the folders and files manually as the older versions doesn't ship Web Key Service (WKS) with it. 

Check whether you've `gpg-wks-client` installed. If you have that, you can skip the manuall setup. 

### Older version (no WKS present)  

If you don't have WKS installed, you can export the keys one by one to the specified tree. Let's create the directories first.  

```bash
# Assuming you're in the root of the repository created  
mkdir -p .well-known/openpgpkey/example.org/hu  # Replace example.org with your domain  
touch .well-known/openpgpkey/example.org/policy  
```

Now that the folders have been set, let's fetch the hashed_userid of the primary key.  Execute:

```bash
gpg --with-wkd-hash --fingerprint you@example.org
# OR 
gpg --with-wkd -k you@example.org  
```

Regardless of which command you executed, you'll get the following output:

```
pub   ed25519/0xD0855B1A8FDCD2F9 2024-07-06 [SC]
      Key fingerprint = 7DE1 A30E 363A FD39 ABCC  BDB4 D085 5B1A 8FDC D2F9
uid                   [ultimate] You <you@example.org>
                      tm4s53wnx8fs6zsorm3tihcmu9ghamw1@example.org
sub   cv25519/0x2C44081BC024D0F8 2024-07-06 [E]
```

Notice the hash under the `uid` section? That's the hashed user ID. That should be our file name (strip the part after "@"). So, in this case, our hashed_userid is `tm4s53wnx8fs6zsorm3tihcmu9ghamw1`. 

Now we need to export the public key to the `/hu` folder. For that, 

```bash
# Assuming you're on the root of the repository
gpg --export --no-armor you@example.org > .well-known/openpgpkey/example.org/hu/tm4s53wnx8fs6zsorm3tihcmu9ghamw1
```

This will dump a binary file to the `/hu` path with the hashed_userid as the file name. 

### Newer versions (with WKS present) [Preferred]  

Starting from GnuPG v2.2.12, the WKS is shipped with it by default. We can use the `gpg-wks-client` to export the public key(s) (bulk) and it'll take care of the folder structure automatically.

```bash

# Assuming you're on the root of the repo  
cd .well-known

# Export public keys (bulk)  
gpg --list-options show-only-fpr-mbox -k "@example.org" | gpg-wks-client -v --install-key

```

The first command `gpg` with `--list-options` will output **all the UIDs that on the domain `example.org` along with their fingerprint**. The second one, `gpg-wks-client` will "install" or export the pub keys as binary to the `./openpgpkey` folder following the domain name and such. This is why we ran that command under `.well-known` directory. 

On a successful event, it'll output something like:  

```
gpg-wks-client: using key with user id 'You <you@example.org>'
gpg-wks-client: directory 'openpgpkey/example.org' created
gpg-wks-client: directory 'openpgpkey/example.org/hu' created
gpg-wks-client: policy file 'openpgpkey/example.org/policy' created
gpg-wks-client: key 7DE1A30E363AFD39ABCCBDB4D0855B1A8FDCD2F9 published for 'you@example.org'
```
If you're managing an organization or have multiple keys under same domain, this `gpg-wks-client` can be treated as a "bulk" upload tool. 

Now that we have the required files, let's commit all the keys to git. But before pushing it to GitHub, we also need to add two more files. 

- An empty `.nojekyll` file which will tell Github to not to even try building this repo (when we enable the GitHub Pages). 
- A `CNAME` file which will contain the domain name `openpgpkey.<your_domain>`

Now add these two files and push it to GitHub. 

## Enabling GitHub Pages  

Now that we have all of our files pushed to main repo, it's time to enable the GitHub Pages. Before enabling, we need to verify / configure our custom domain with GitHub. For that, follow [this guide from GitHub](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages). 

After than you can enable GitHub Pages from the repository's settings page. Make sure you have the following options set. 

- Deployment Source: **Deploy from Branch**
- Branch: **master**(or your main branch), and path is set to **/root**
- Add custom domain (`openpgpkey.<your_domain>`)

And that's it. Now you have your own WKD service!. Have fun!

Bye!


## References  

- [OpenPGP Web Key Directory (WKD) [YouTube]](https://www.youtube.com/watch?v=O7JaWH52cRc)
- [A blog I found](https://www.uriports.com/blog/setting-up-openpgp-web-key-directory/)
- [A repo I found which uses GitHub Pages as WKD base](https://github.com/mihalyr/openpgpkey)
- [GnuPG Wiki](https://wiki.gnupg.org/WKDHosting)

