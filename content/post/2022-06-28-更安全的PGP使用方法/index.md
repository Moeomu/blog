---
title: Safer use of PGP
description: Separate master key and subkey
date: 2022-06-28 08:58:01+0800
categories:
    - Security
tags:
    - GnuPG
    - GPG
    - PGP
    - Information Security
    - End-to-end encryption
---

Source of this article: [MoeomuBlog](/posts/safer-use-of-pgp/)

## Introduction of GnuPG

### Official Introduction

> Source: [GnuPG](https://www.gnupg.org/)

GnuPG is a complete and free implementation of the OpenPGP standard as defined by RFC4880 (also known as PGP). GnuPG allows you to encrypt and sign your data and communications; it features a versatile key management system, along with access modules for all kinds of public key directories. GnuPG, also known as GPG, is a command line tool with features for easy integration with other applications. A wealth of frontend applications and libraries are available. GnuPG also provides support for S/MIME and Secure Shell (ssh).

Since its introduction in 1997, GnuPG is Free Software (meaning that it respects your freedom). It can be freely used, modified and distributed under the terms of the GNU General Public License .

> Arguing that you don't care about the right to privacy because you have nothing to hide is no different from saying you don't care about free speech because you have nothing to say.  
> – Edward Snowden

Using encryption helps to protect your privacy and the privacy of the people you communicate with. Encryption makes life difficult for bulk surveillance systems. GnuPG is one of the tools that Snowden used to uncover the secrets of the NSA.

Please visit the [Email Self-Defense site]((https://emailselfdefense.fsf.org/en/)) to learn how and why you should use GnuPG for your electronic communication.

### So

#### Q1: Why should I use GnuPG?

- Because it is free software and can be used, modified and distributed freely.
- By using it, your communication information will be encrypted and secure.
- PGP can guarantee that a message is sent by someone you trust, and no one else can decrypt it except you two, and this message is transmitted without any modification of even one punctuation or one byte in between.
- A PGP-signed message can be transmitted over any untrusted channel and cannot be tampered with or decrypted by anyone in the middle.

#### Q2: I'm obviously not afraid of censorship. What other reasons do you have to recommend GnuPG?

- Your privacy is up to you, and most people don't want to live under surveillance.
- If even mild criticism is not allowed, silence will be perceived as ill-intentioned. If silence were no longer allowed, it would be a crime to praise inadequate efforts. If only one voice is allowed to exist, then the only voice that exists is a lie.
- If it were more unfortunate to live in a totalitarian state, encrypted communications would protect yourself and the friends you care about.

## Installation

### Windows

> Gpg4win is recommended, it has a complete package and an easy to use GUI.

- Download the version for your computer from [Gpg4win](https://gpg4win.org/download.html). In general, if you have no special needs, just click the green download button.

### macOS

> GPGSuite is recommended

- Download the version for your computer from [GPGSuite](https://gpgtools.org). In general, if you have no special needs, just click the red download button.

> Reminder: GPGMail in the GPGSuite suite is paid software, so if you need to use it to automatically encrypt emails, then you need to purchase GPGMail. but rest assured that the encrypted signature of text and files is completely free.

### Linux

> GnuPG is recommended

- Most Linux distributions come with GnuPG pre-installed and you do not need to install any software. However, it is recommended that you install a GUI management interface `GPA`, as it is more user-friendly.

## Generating keys

> Please experiment with this section first, and then generate your main key after you become proficient.

### Generate a master key

- Enter the command `gpg -expert --full-gen-key`
- Select key type: default
- Enter the key length: 4096
- Enter the key expiration time: `2y` (for two years, you can enter 0 to never expire, you can change the key expiration time at any time), enter y to confirm.
- Enter your name, this is optional, you can enter your screen name or even leave it out.
- Enter your email, which is optional and can even be left out. If you want to use email encryption, then you must enter it.
- Enter a note, which can be left blank.
- Enter O to confirm the information is correct, then you need to enter a key password to start the key generation.

### Generate revocation certificate

> If you lose your master key (or it is taken), you can use a revocation certificate to prove that it is no longer in use. If you do not have a revocation certificate, then you must notify your friends one by one.

- Enter the command `gpg --gen-revoke -ao revoke.pgp email@email.com #uid or key id`

### View subkeys

> The master key should not be used on a daily basis, you should use the generated subkeys on a daily basis. The advantage of this is that if you leak a subkey, you can immediately revoke it without having to deprecate the entire key pair.

- List all public keys and subkeys: `-list-keys` or `gpg -k`
- List all keys, subkeys: `-list-secret-keys` or `gpg -K`
- To see the key fingerprint information, you can add the `-fingerprint` parameter, such as `gpg --list-secret-keys --fingerprint`
- If you want to see the key ID information, you can add the `-keyid-format long` parameter, such as `gpg --list-secret-keys --keyid-format long`

## Exporting keys

> Since subkeys are to be used for daily use, the master key should not exist in the same place, GPG cannot delete the key, it can only export and pour in only the subkeys needed.
>
> Note: The exclamation mark after the key ID cannot be missing, otherwise all keys will be exported

### Export public key

- Export public key: `gpg -ao public-key --export master-key-id`

### Export the master key

- Enter the command `gpg -ao master-key --export-secret-key master-key ID!`

### Export subkeys

- Enter the command `gpg -ao sub-key --export-secret-subkeys Sub-key ID!`

## Import keys

- Enter the command `gpg --import filename`

## Communicate securely with your friends

- Now that you're done, you'll have a hard time leaving PGP after you get into the habit of using it daily

## Publish to the public key server

... To be continued...

## References

- Ulyc. (2021, January 13). 2021年，用更现代的方法使用PGP（上）. C的博客. <https://ulyc.github.io/2021/01/13/2021年-用更现代的方法使用PGP-上/>
- Ulyc. (2021, January 13). 2021年，用更现代的方法使用PGP（中）. C的博客. <https://ulyc.github.io/2021/01/13/2021年-用更现代的方法使用PGP-中/>
- Ulyc. (2021, January 13). 2021年，用更现代的方法使用PGP（下）. C的博客. <https://ulyc.github.io/2021/01/13/2021年-用更现代的方法使用PGP-下/>
