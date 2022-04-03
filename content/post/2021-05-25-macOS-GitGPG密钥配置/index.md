---
title: macOS-GitGPG Key Configuration
description: GitGPG key configuration for macOS and enable signature verification
date: 2021-05-25 07:55:00 +0800
categories:
    - Config
tags:
    - macOS
    - Git
    - GPG
---

Source: [Moeomu's Blog](/posts/macos-gitgpg-key-configuration/)

## About GitHub GPG Key Verification

### Enabling Commit Signing

GitHub has a new "Vigilance Mode" that requires a GPG key to sign a certified commit before it will show "Verity".

- First create a GPG key ([GitHub Official Docs](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/) generating-a-new-gpg-key) has details on how to do this, so I won't go over it again)
- List the GPG key's signature: `gpg -K --keyid-format LONG`, which will **keyid** record
- Tell git to use this GPG key: `git config user.signingkey your_keyid`
- The local git username and email need to be the same as the ones filled in when the GPG key was generated: `git config user.name name`, `git config user.email email email`
- Enable commit signing for local git: `git config commit.gpgsign true`
- Add the -S option to the Commit signature: `git commit -S -m message`

### Fatal error occurred - unable to commit-macOS

#### The problem is as follows, reproduced in macOS 11.3.1

```text
error: gpg failed to sign the data
fatal: failed to write commit object
```

#### solution in macOS

- Update & Install

```shell
brew upgrade gnupg  # This has a make step which takes a while
brew link --overwrite gnupg
brew install pinentry-mac
echo "pinentry-program /usr/local/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
killall gpg-agent

git config --global gpg.program gpg  # perhaps you had this already? On linux maybe gpg2
git config --global commit.gpgsign true  # if you want to sign every commit
```

- Sign again
- Check the status of the commit: `git log --show-signature -1`
