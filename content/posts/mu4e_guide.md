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
- **IMAP Mailbox Synchronizer** - Because this setup is a traditional *offline* email client, we'll need a way to actually get the emails from the server into our maildir. For that we're using [```isync```](https://isync.sourceforge.io/). ```isync``` (also known as ```mbsync``` due to the name of the executable itself) is a fast, highly configurable IMAP synchronization tool. Another popular option is ```offlineimap```, but it has a number of shortcomings compared to ```isync```, as detailed in [Anarcat's excellent benchmark comparison between the two](https://anarc.at/blog/2021-11-21-mbsync-vs-offlineimap/).
- **SMTP Client** - Composing and sending mail from Emacs is probably the main reason anybody would want to set this up, and for that we're using [```msmtp```](https://marlam.de/msmtp/). ```msmtp``` is the tool that actually forwards our local emails to the SMTP server.

These four tools, ```mu4e```, ```mu```, ```isync```, and ```msmtp``` are all that's needed to have a fully functional email client inside of Emacs. In the next sections I'll explain how to configure each of them, providing examples and warning of any pitfalls I encountered along the way. I'm going to be doing this on Ubuntu 24.04, but the only real difference between distros or even operating systems should be package installation, and init system, so as long as you can install packages, and run daemons in your system of choice, these steps should be universal.

## isync
In order to read emails in Emacs, we first have to get them from the server into our maildir. As I mentioned above, while the project is called ```isync```, the actual program is called ```mbsync```, so don't get confused if I use them interchangeably.

{{% steps %}}

### Installation

Install ```isync```. On Debian based systems, it looks like this: ```sudo apt install isync```.

### Configuration

Your entire isync configuration will live in a configuration file, which is found by default at ```~/.mbsyncrc```. If you'd like to put the config file somewhere else, you can pass the location to ```mbsync``` with the ```-c``` flag. Configuring ```isync``` is probably the most tedious part of this entire setup, but once you wrap your brain around the various moving pieces, it's really not too bad. My advice is to start with one account, get it working how you like, and then move on to your other accounts afterward.

An ```isync``` configuration file defines an email account in a few different, interconnected sections. Understanding what each of the sections does, and how they work together will vastly improve your ```isync``` experience, and will make adding new accounts later on much less irritating. Typically, an account is defined in the following sections: *Remote Stores*, *Local Stores*, and *Channels*.

While I could define each of these sections (or *classes* as the ```isync``` documentation refers to them), I'll just let you read the definitions given in the documentation: 

> A Store defines a collection of mailboxes; basically a folder, either local or remote. A Channel connects two Stores, describing the way the two are synchronized.

Finally, we'll also make use of the optional class *Account*. The *Account* class lets us separate a network based *Store's* connection configuration from its file configuration. Essentially, if you have multiple mailboxes on the same server, you can save yourself the typing and apply the same connection configuration to multiple mailboxes.

#### Remote Store Configuration
Without further ado, here's what the configuration for a remote *Store* looks like:
```
IMAPAccount mxroute
Host glacier.mxrouting.net
Port 993
User cole@hohosunbear.com
PassCmd "gpg -dq $HOME/.pass/mxroute_pass.gpg"
SSLType IMAPS
SSLVersions TLSv1.2

IMAPStore mxroute-remote
Account mxroute
```
Let's walk through the configuration of an *Account* and remote *Store*.

```
IMAPAccount mxroute
```
The configuration starts with what ```isync``` calls a "section-starting keyword" in ```IMAPAccount```. With this keyword, ```isync``` knows that we are defining an *Account* class. The parameter "mxroute" is the human readable name that we can refer to the *Account* by later in the configuration. You can name your *Account* whatever you'd like; since I host my mail on the excellent [MXroute](https://mxroute.com/) mail provider, I called the *Account* "mxroute".

```
Host glacier.mxrouting.net
Port 993
```
Naturally, the ```Host``` and ```Port``` keywords tell ```isync``` where it needs to connect to the mail server. 993 is the default port for secure IMAP, and unless you have a special setup, this is probably what you'll be using.

```
User cole@hohosunbear.com
PassCmd "gpg -dq $HOME/.pass/mxroute_pass.gpg"
```
This section is also pretty straightforward. ```User``` is the username you want ```isync``` to log into server as. Here, I'm using the ```PassCmd``` keyword instead of the standard ```Pass``` keyword. If you use ```Pass```, note that your **actual** password will be in your config file, so I'd highly recommend using ```PassCmd``` instead. With ```PassCmd```, you can use any arbitrary shell command to retrieve your password. I've configured mine to retrieve my password from an encrypted file that my keyring can unlock. No cleartext passwords here!

```
SSLType IMAPS
SSLVersions TLSv1.3
```
The final part of the *Account* definition is configuring SSL. Since we're using secure IMAP, ```SSLType``` needs to be set to ```IMAPS```. Last but not least, we need to tell ```isync``` what TLS version to use. You'll have to get this setting from your mail provider, but if it's any good, it will be 1.3. 

And that's it for the *Account* configuration. If you look back at the full excerpt I posted above, you'll notice that there is a blank line between the *Account* section and the *Store* section. This is necessary, as ```isync``` expects class definitions to be terminated either by a blank line or the end of the file, so don't forget it. Anyway, now we just have to define a remote *Store* and tell it to use the *Account* we've just defined. 

```
IMAPStore mxroute-remote
Account mxroute
```
Again, we start the *Store* class definition with the ```IMAPStore``` keyword. And again, the parameter is just a human readable name of your choice. The ```Account``` keyword really shows of the power of the *Account* class, as we don't really have to do any configuration here, we can just pass it the name we gave our *Account* definition, and it happily uses that. If you're only going to use one *Store* per *Account*, you *can* just populate your *Store* definition with the *Account* options, but I don't really see a reason to ever do so. Finally, if your mail server has any special configuration, there are some other ```IMAPStore``` options you can use, but for most setups, what I have will work just fine.

#### Local Store Configuration
Now that we've told ```isync``` how where to look for the mail on your mail server, we need to tell it where to put that mail on your local machine. To do so, we define a local *Store* that's analogous to the remote one we just defined. Here's my definition:
```
MaildirStore mxroute-local
Path ~/Mail/HohoSunBear/
Inbox ~/Mail/HohoSunBear/Inbox
# Represent IMAP subfolders as local subfolders
Subfolders Verbatim
```
Since we *just* defined a remote *Store* we can move a little quicker here. We start the definition with the section-starting keyword ```MaildirStore``` and call it "mxroute-local". We've done this part before.

```
Path ~/Mail/HohoSunBear/
Inbox ~/Mail/HohoSunBear/Inbox
```
The ```Path``` keyword is the root directory of the mailbox's filesystem on your machine. Typically your maildir will look like mine: a ```Mail``` directory that all your mailboxes live in, and then a subdirectory for each of your mailboxes. In this example, we're configuring my Hoho Sun Bear email, so its path is ```~/Mail/HohoSunBear```. Note that the trailing slash is required if you're intending to specify a directory here, which you probably are. The ```Inbox``` keyword works much the same as ```Path``` but it just tells ```isync``` where you want your inbox.

```
Subfolders Verbatim
```
This is what stops us from having to manually define each mail folder. You have several options here, but I just wanted to keep it simple and have my on-disk hierarchy reflect the mail server's hierarchy, so I just used "Verbatim". This is also what the documentation recommends.

Now we've gotten both *Stores* defined, the remote one on the mail server, and the local one on our workstation. All that's left is to define the *Channel* that connects them and we'll have ```isync``` up and running and synchronizing our emails over secure IMAP. 

#### Channel Configuration
The *Channel* class is where ```isync```'s power and flexibility really becomes apparent. Consequently, this is also where you're most likely to find yourself wanting to deviate from my setup. I have what I'd consider sane defaults, but if you find yourself wanting different behaviour, feel free the peruse the documentation and tweak to your heart's content.
```
Channel mxroute
Far :mxroute-remote:
Near :mxroute-local:
Patterns *
Expunge Both
CopyArrivalDate yes
Sync All
Create Near
SyncState *
```
Once again, we start and name our definition with ```Channel mxroute```.

```
Far :mxroute-remote:
Near :mxroute-local:
```
This is how we tell the *Channel* which *Stores* it's actually connecting. In case it's not clear, you can think of these as "source" and "destination" respectively. Mail is pulled from ```Far``` into ```Near```. You may optionally specify a mailbox after the *Store* name like so: ```:mxroute-remote:example-mailbox```, but if you don't provide one, "inbox" is assumed.

```
Patterns *
```
```Patterns``` instructs ```isync``` what you'd actually like to sync between *Stores*. In this case I just used a wildcard since I wanted everything, but there is a ton of customization in case you want to exclude certain mailboxes or pretty much anything else you can think of.

```
Expunge Both
```
This is how you control deletion of emails. Since this is naturally a risky operation, ```isync``` recommends (rightfully so) to set this *after* you've ensured your configuration is otherwise working as intended. With ```Both``` set, messages marked for deletion are deleted on both the ```Near``` and ```Far``` sides of the channel. This is probably what you want, but do some testing before you set it.

```
CopyArrivalDate yes
```
I'm not really sure why one *wouldn't* want this option, but for whatever reason, the default is ```no```. It just ensures the arrival date of messages on the ```Far``` side are propagated to the messages on the ```Near``` side.

```
Sync All
```
Here's the meat of *Channel* as a whole: we're *finally* telling it how to sync the two *Stores*. ```All``` is the default and sets full two-way syncing to between the ```Stores```. Again, this is probably what you want, but there are a bunch of options in case it isn't.

```
Create Both
```
Here we give ```isync``` the power to create mailboxes on either side of the *Channel*. I used ```Near``` initially, since I didn't want mailboxes getting created on the server, but after some testing I've switched to ```Both``` so I don't have to log in to the server directly in order to organize my emails. 

```
SyncState *
```
This is sort of an implementation specific option, and might differ depending on your setup, but put simply, it specifies where ```isync``` should store its metadata files. ```*``` tells it to use a file that is crucially *in* the ```Near``` mailbox itself. If you don't care about that, go ahead and use ```*```, and everything should work as intended. If you have a more advanced setup, you'll have to figure out what you'd like to use here.

### Conclusion

Finally, at long last, we've gotten an end to end mailbox fully configured in ```isync```. This configuration *should* be at least *pretty* close for most standard mail servers that use secure IMAP. Any additional mailboxes and accounts you'd like to use can just go into this same configuration file. Gmail is a common sticking point for ```isync```/IMAP users, as it has some unusual settings required to get it working, so if you'd like some help getting a Gmail account configured in ```isync```, I'll be posting another article specifically on that topic in short order.

Now, you should be able to run ```isync``` with the following command: ```mbsync --all```. This first run can take quite a while depending on how many emails you have on the server, but subsequent invocations are much faster, as only new emails need to be fetched. With that, we've gotten over the first major hurdle of Emacs/text based email: getting the mail. Later on we'll go over how to do this automatically, but for now, congratulate yourself on having completed perhaps the most annoying aspect of this project.

{{% /steps %}}
