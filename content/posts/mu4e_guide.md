---
title: mu4e Setup Guide
cascade:
    type: blog
---

## Introduction
If you're anything like me, the idea of doing *everything* in Emacs is always a fun challenge, if not entirely practical. Email is a task that fulfills both of those qualifiers, as modern email isn't particularly well suited to a text-only program, and it has the added benefit of often being a huge pain to get working. I went through the rewarding, at times arduous process of getting full email functionality up and running in my Emacs, and if you'd like to do the same, read on. Finally, note that while Emacs and mu4e is my some-assembly-required email client of choice, the steps and tools detailed in this article will be fairly similar to the other options such as Mutt or notmuch.

## Overview
Email clients in general are typically built on top of several different tools that all need to play nicely together, and Emacs is no different. Here we'll go over the various moving pieces we'll be setting up.

- **Client** - For our actual client/frontend we'll obviously be using [```mu4e```](https://github.com/emacsmirror/mu4e). ```mu4e``` is a set of Emacs bindings built on top of the ```mu``` email indexing tool.
- **Indexer** - Since mu4e is just a frontend for ```mu```, we'll also need, of course, [```mu```](https://www.djcbsoftware.nl/code/mu/) itself. ```mu``` is what will actually index and operate on our downloaded emails under the hood.
- **IMAP Mailbox Synchonizer** - Because this setup is a traditional *offline* email client, we'll need a way to actually get the emails from the server into our maildir. For that we're using [```isync```](https://isync.sourceforge.io/). ```isync``` (also known as ```mbsync``` due to the name of the executable itself) is a fast, highly configurable IMAP sychronization tool. Another popular option is ```offlineimap```, but it has a number of shortcomings compared to ```isync```, as detailed in [Anarcat's excellent benchmark comparison between the two](https://anarc.at/blog/2021-11-21-mbsync-vs-offlineimap/).
- **SMTP Client** - Composing and sending mail from Emacs is probably the main reason anybody would want to set this up, and for that we're using [```msmtp```](https://marlam.de/msmtp/). ```msmtp``` is the tool that actually forwards our local emails to the SMTP server.

These four tools, ```mu4e```, ```mu```, ```isync```, and ```msmtp``` are all that's needed to have a fully functional email client inside of Emacs. In the next sections I'll explain how to configure each of them, providing examples and warning of any pitfalls I encountered along the way. I'm going to be doing this on Ubuntu 24.04, but the only real difference between distros or even operating systems should be package installation, and init system, so as long as you can install packages, and run daemons in your system of choice, these steps should be universal.

## isync Configuration
In order to read emails in Emacs, we first have to get them from the server into our maildir. As I mentioned above, while the project is called ```isync```, the actual program is called ```mbsync```, so don't get confused if I use them interchangeably.

{{% steps %}}

### Step 1 - Install

Install ```isync```. On Debian based systems, it looks like this: ```sudo apt install isync```.

### Step 2 - Configure

Your entire isync configuration will live in a configuration file, which is found by default at ```~/.mbsyncrc```. If you'd like to put the config file somewhere else, you can pass the location to ```mbsync``` with the ```-c``` flag.

{{% /steps %}}
