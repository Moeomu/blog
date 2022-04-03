---
title: macOS-pkg file parsing
description: macOS-pkg file parsing & PD16 can't network solution & @ symbol problem of ls
date: 2020-11-24 12:56:00+0800
categories:
    - Analyze
tags:
    - macOS
---

Source: [Moeomu's blog](/posts/macos-pkg-file-analysis/)

## Cause

I couldn't download the ed2k link with motrix, so I was going to download a lite version of Xunlei, and suddenly I found a pkg installer, which piqued my interest.

## After

### Unpacking

- I've always wondered what pkg actually runs and what it does, so I started the first step, unpacking
- Unpacking only requires one command: `pkgutil --expand the pkg.pkg that needs to be unpacked Custom unpacking folder name`.

### View

- I found a Res file with no suffix, so I checked the format with file and found that it was compressed, so I uncompressed it with the unar command and came out with a file of the same name, but with a bigger size
- right click in finder - open way - archive utility - app was extracted directly
- Drag this app into /Application and found that it can be used directly, thus completing the task
- Looked at the other files, there is a Unix executable for opening web pages, probably a promotion, no rogue behavior

## Result

- Finish the task

## Adding

### iOS app installer exploration

- After updating the BigSur system, I found that there is such a thing as `iOS app installer` in the system, but it doesn't work, so I explored it

I explored it![System](https://s3.ax1x.com/2020/11/24/Dt5BZt.png)

- Check the location of this software, and check the type, it's good, generic, means maybe Intel will be supported in the future?

![Locate](https://s3.ax1x.com/2020/11/24/Dt5rIf.png)

- Open Terminal, go to this folder, run the software, error is reported, suggesting that dependency files are needed, as shown in the picture

![2](https://s3.ax1x.com/2020/11/24/Dt5DdP.png)

- Find the location of the dependency file and find that there seems to be nothing, probably because only the ARM version of the dependency program is available

![3](https://s3.ax1x.com/2020/11/24/Dt5wqI.png)

### Parallels Desktop cannot be networked solution

- Still the same problem with BigSur, PD16 is not networked again, so I have no choice but to find a compromise solution
- Run the command `sudo -b /Applications/Parallels\ Desktop.app/Contents/MacOS/prl_client_app` and it will open PD16, so you will be able to connect to the Internet!
- This command means to run the program with administrator privileges, but in fact, BigSur should have tightened the privileges again, causing problems with PD16 cracking
- You can write this command in a .command file on your desktop, so that you can double-click it to open it

### Problem with @ symbol in ls -l

- Recently, I found that some folders/files are marked with the @ symbol, so I wondered what it meant, so I briefly explored it.
- Since this phenomenon is caused by the ls program, I asked the system's documentation and commanded `man ls`.
- It was quick, and it came out with a snap, as shown in the picture

![man-ls](https://s3.ax1x.com/2020/11/24/Dto2bn.png)

```s
-@ Display extended attribute keys and sizes in long (-l) output.
```

- It is very clear that such file/folder has an extended attribute which can be viewed with the `xattr -l` command
- This attribute can also be cleaned up completely with `xattr -c`
