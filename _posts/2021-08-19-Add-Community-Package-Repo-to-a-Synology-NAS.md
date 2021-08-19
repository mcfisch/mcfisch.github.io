---
title: "Add Community Package Repo to a Synology NAS"
date: 2021-08-19 10:00 -0700
categories: [Synology]
tags: [synology]
excerpt: "This shows how to make the Syno Community package repository available to a Synology NAS."
# toc: true
classes: wide
---

This shows how to add the Syno Community packages to the list of package sources in a Synology NAS. The extra packages come in handy if one for instance needs a tool, that Synology only provides as part of a bigger package (i.e. `git`), or one they don't provide at all.

## Syno Community

The Syno Community package repository is an OpenSource project that provides software packages which aren't provided or supported by Synology, for installation on a Synology device. If you want to contribute to the project visit their Github page where instructions for contributors are provided as well as a link for donations.

[SynoCommunity/spksrc on Github](https://github.com/SynoCommunity/spksrc)

## Adding the repo as a package source

The following steps are a quote directly from synocommunity.com, as it describes the process as easy as it gets:

>Step 1<br>
>Log into your NAS as administrator and go to Main Menu → Package Center → Settings and set Trust Level to Synology Inc. and trusted publishers.
><br><br>
>Step 2<br>
>In the Package Sources tab, click Add, type SynoCommunity as Name and https://packages.synocommunity.com/ as Location and then press OK to validate.
><br><br>
>Step 3<br>
>Go back to the Package Center and enjoy SynoCommunity's packages in the Community tab.
{: .notice}

## In case of a problem

### Check the URL

By the time of writing this post the URL of the Syno Community package repository was https://packages.synocommunity.com/. To verify this is still correct visit the [Syno Community page](https://synocommunity.com) and check the instructions near the bottom. If the site becomes unavailable search the internet for `add synocommunity to synology` or so.

>**Note:**<br>
>The repo URL can't be visited by a regular browser and will show an error!
{: .notice}

### Verify the trust level

Make sure the trustlevel is set correctly as instructed in `Step 1` above. Otherwise the Diskstation won't accept the package source.

## Manual package download and installation

In case the community repo is down you can try finding the desired package somewhere else. For instance there is a fork of the community repo where the account owner builds releases of some of the packages: [publicarray/spksrc on Github](https://github.com/publicarray/spksrc/releases). There are other sources available online as well. 

To install a package manually copy the `spk` file to your Synology device and install it by running:

```bash
sudo synopkg install <path to .spk file>
```

>**Warning:**<br>
>Apply the usual caution when installing packages found somewhere in the depth of the interweb!
{: .notice--warning}

`synopkg` also accepts other parameters, e.g. to start or stop a service provided by a package. Run `synopkg help` to see a comprehensive list.
