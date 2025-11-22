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

Install ```isync```. On Debian based systems, it looks like this: ```apt install isync```.

### Configuration

Your entire isync configuration will live in a configuration file, which is found by default at ```~/.mbsyncrc```. If you'd like to put the config file somewhere else, you can pass the location to ```mbsync``` with the ```-c``` flag. Configuring ```isync``` is probably the most tedious part of this entire setup, but once you wrap your brain around the various moving pieces, it's really not too bad. My advice is to start with one account, get it working how you like, and then move on to your other accounts afterward.

An ```isync``` configuration file defines an email account in a few different, interconnected sections. Understanding what each of the sections does, and how they work together will vastly improve your ```isync``` experience, and will make adding new accounts later on much less irritating. Typically, an account is defined in the following sections: *Remote Stores*, *Local Stores*, and *Channels*.

While I could define each of these sections (or *classes* as the ```isync``` documentation refers to them), I'll just let you read the definitions given in the documentation: 

> A Store defines a collection of mailboxes; basically a folder, either local or remote. A Channel connects two Stores, describing the way the two are synchronized.

Finally, we'll also make use of the optional class *Account*. The *Account* class lets us separate a network based *Store's* connection configuration from its file configuration. Essentially, if you have multiple mailboxes on the same server, you can save yourself the typing and apply the same connection configuration to multiple mailboxes.

#### Remote Store
Without further ado, here's what the configuration for a remote *Store* looks like:
```conf
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

```conf
IMAPAccount mxroute
```
The configuration starts with what ```isync``` calls a "section-starting keyword" in ```IMAPAccount```. With this keyword, ```isync``` knows that we are defining an *Account* class. The parameter "mxroute" is the human readable name that we can refer to the *Account* by later in the configuration. You can name your *Account* whatever you'd like; since I host my mail on the excellent [MXroute](https://mxroute.com/) mail provider, I called the *Account* "mxroute".

```conf
Host glacier.mxrouting.net
Port 993
```
Naturally, the ```Host``` and ```Port``` keywords tell ```isync``` where it needs to connect to the mail server. 993 is the default port for secure IMAP, and unless you have a special setup, this is probably what you'll be using.

```conf
User cole@hohosunbear.com
PassCmd "gpg -dq $HOME/.pass/mxroute_pass.gpg"
```
This section is also pretty straightforward. ```User``` is the username you want ```isync``` to log into server as. Here, I'm using the ```PassCmd``` keyword instead of the standard ```Pass``` keyword. If you use ```Pass```, note that your **actual** password will be in your config file, so I'd highly recommend using ```PassCmd``` instead. With ```PassCmd```, you can use any arbitrary shell command to retrieve your password. I've configured mine to retrieve my password from an encrypted file that my keyring can unlock. No cleartext passwords here!

```conf
SSLType IMAPS
SSLVersions TLSv1.3
```
The final part of the *Account* definition is configuring SSL. Since we're using secure IMAP, ```SSLType``` needs to be set to ```IMAPS```. Last but not least, we need to tell ```isync``` what TLS version to use. You'll have to get this setting from your mail provider, but if it's any good, it will be 1.3. 

And that's it for the *Account* configuration. If you look back at the full excerpt I posted above, you'll notice that there is a blank line between the *Account* section and the *Store* section. This is necessary, as ```isync``` expects class definitions to be terminated either by a blank line or the end of the file, so don't forget it. Anyway, now we just have to define a remote *Store* and tell it to use the *Account* we've just defined. 

```conf
IMAPStore mxroute-remote
Account mxroute
```
Again, we start the *Store* class definition with the ```IMAPStore``` keyword. And again, the parameter is just a human readable name of your choice. The ```Account``` keyword really shows of the power of the *Account* class, as we don't really have to do any configuration here, we can just pass it the name we gave our *Account* definition, and it happily uses that. If you're only going to use one *Store* per *Account*, you *can* just populate your *Store* definition with the *Account* options, but I don't really see a reason to ever do so. Finally, if your mail server has any special configuration, there are some other ```IMAPStore``` options you can use, but for most setups, what I have will work just fine.

#### Local Store
Now that we've told ```isync``` how where to look for the mail on your mail server, we need to tell it where to put that mail on your local machine. To do so, we define a local *Store* that's analogous to the remote one we just defined. Here's my definition:
```conf
MaildirStore mxroute-local
Path ~/Mail/HohoSunBear/
Inbox ~/Mail/HohoSunBear/Inbox
# Represent IMAP subfolders as local subfolders
Subfolders Verbatim
```
Since we *just* defined a remote *Store* we can move a little quicker here. We start the definition with the section-starting keyword ```MaildirStore``` and call it "mxroute-local". We've done this part before.

```conf
Path ~/Mail/HohoSunBear/
Inbox ~/Mail/HohoSunBear/Inbox
```
The ```Path``` keyword is the root directory of the mailbox's filesystem on your machine. Typically your maildir will look like mine: a ```Mail``` directory that all your mailboxes live in, and then a subdirectory for each of your mailboxes. In this example, we're configuring my Hoho Sun Bear email, so its path is ```~/Mail/HohoSunBear```. Note that the trailing slash is required if you're intending to specify a directory here, which you probably are. The ```Inbox``` keyword works much the same as ```Path``` but it just tells ```isync``` where you want your inbox.

```conf
Subfolders Verbatim
```
This is what stops us from having to manually define each mail folder. You have several options here, but I just wanted to keep it simple and have my on-disk hierarchy reflect the mail server's hierarchy, so I just used "Verbatim". This is also what the documentation recommends.

Now we've gotten both *Stores* defined, the remote one on the mail server, and the local one on our workstation. All that's left is to define the *Channel* that connects them and we'll have ```isync``` up and running and synchronizing our emails over secure IMAP. 

#### Channel
The *Channel* class is where ```isync```'s power and flexibility really becomes apparent. Consequently, this is also where you're most likely to find yourself wanting to deviate from my setup. I have what I'd consider sane defaults, but if you find yourself wanting different behaviour, feel free the peruse the documentation and tweak to your heart's content.
```conf
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

```conf
Far :mxroute-remote:
Near :mxroute-local:
```
This is how we tell the *Channel* which *Stores* it's actually connecting. In case it's not clear, you can think of these as "source" and "destination" respectively. Mail is pulled from ```Far``` into ```Near```. You may optionally specify a mailbox after the *Store* name like so: ```:mxroute-remote:example-mailbox```, but if you don't provide one, "inbox" is assumed.

```conf
Patterns *
```
```Patterns``` instructs ```isync``` what you'd actually like to sync between *Stores*. In this case I just used a wildcard since I wanted everything, but there is a ton of customization in case you want to exclude certain mailboxes or pretty much anything else you can think of.

```conf
Expunge Both
```
This is how you control deletion of emails. Since this is naturally a risky operation, ```isync``` recommends (rightfully so) to set this *after* you've ensured your configuration is otherwise working as intended. With ```Both``` set, messages marked for deletion are deleted on both the ```Near``` and ```Far``` sides of the channel. This is probably what you want, but do some testing before you set it.

```conf
CopyArrivalDate yes
```
I'm not really sure why one *wouldn't* want this option, but for whatever reason, the default is ```no```. It just ensures the arrival date of messages on the ```Far``` side are propagated to the messages on the ```Near``` side.

```conf
Sync All
```
Here's the meat of *Channel* as a whole: we're *finally* telling it how to sync the two *Stores*. ```All``` is the default and sets full two-way syncing to between the ```Stores```. Again, this is probably what you want, but there are a bunch of options in case it isn't.

```conf
Create Both
```
Here we give ```isync``` the power to create mailboxes on either side of the *Channel*. I used ```Near``` initially, since I didn't want mailboxes getting created on the server, but after some testing I've switched to ```Both``` so I don't have to log in to the server directly in order to organize my emails. 

```conf
SyncState *
```
This is sort of an implementation specific option, and might differ depending on your setup, but put simply, it specifies where ```isync``` should store its metadata files. ```*``` tells it to use a file that is crucially *in* the ```Near``` mailbox itself. If you don't care about that, go ahead and use ```*```, and everything should work as intended. If you have a more advanced setup, you'll have to figure out what you'd like to use here.

### Conclusion

Finally, at long last, we've gotten an end to end mailbox fully configured in ```isync```. This configuration *should* be at least *pretty* close for most standard mail servers that use secure IMAP. Any additional mailboxes and accounts you'd like to use can just go into this same configuration file. Gmail is a common sticking point for ```isync```/IMAP users, as it has some unusual settings required to get it working, so if you'd like some help getting a Gmail account configured in ```isync```, I'll be posting another article specifically on that topic in short order.

Now, you should be able to run ```isync``` with the following command: ```mbsync --all```. This first run can take quite a while depending on how many emails you have on the server, but subsequent invocations are much faster, as only new emails need to be fetched. With that, we've gotten over the first major hurdle of Emacs/text based email: getting the mail. Later on we'll go over how to do this automatically, but for now, congratulate yourself on having completed perhaps the most annoying aspect of this project.

{{% /steps %}}

## mu

Now that we've solved the issue of getting the mail from the server onto the machine, our next task is to make it easy to *do stuff* to the mail we've downloaded. For that our setup uses ```mu```. ```mu``` is an email indexing tool written in C++ that makes operating on Maildir formatted mailboxes easy. ```mu```, as the name would imply, is also the core upon which ```mu4e```, our Emacs major mode, is built on, so getting it up and running is a non-optional step in this process. Lucky for us, this step is quite a bit less involved than ```isync```.

{{% steps %}}

### Installation
Nothing special here, on Debian based systems, just run ```apt install mu4e```.

### Usage
```mu``` doesn't require any configuration if you've got your mail in the Maildir format it needs. If you've followed my ```isync``` tutorial, that should be the case. In order for ```mu``` to index your maildir, you'll first have to initialize its database with the command ```mu init -m <maildir> --my-address=<my-email-address>```. For the ```isync``` configuration we built earlier, the command would look like this: ```mu init -m ~/Mail --my-address=cole@hohosunbear.com```. If you've added multiple email address in the ```isync```configuration, then you can repeat the ```--my-address``` parameter as many times as you need. Next, you can actually initiate the indexing like so: ```mu index```. That's it.

### Conclusion
`mu` is definitely a nice little reprieve from the configuration gauntlet that is ```isync```, and I find it fitting that it's the natural next step. And, now we're really starting to get somewhere: we've got two different tools working together, which is really the name of the game with this project. 

{{% /steps %}}

## msmtp
At this point, we could either continue with the ```mu``` configuration, and start working on ```mu4e``` in Emacs, or we could keep setting up Unix-philosophy-adhering programs that make all this work underneath. Since ```mu4e``` isn't all that useful without the ability to *send* emails, I'm voting to continue on the command line and in config files for just a little bit longer. The fun part is coming, I promise, but before we start writing elisp, let's put our heads down, and get the last big piece cemented into place: ```msmtp```. 

As the name implies, ```msmtp``` is the SMTP client we'll be using to send email from ```mu4e```. Much like ```isync```, it relies on a simple configuration file to tell it where, and how, to connect to your mail server. Thankfully, you should already be in the email configuration file mindset, *and* the ```msmtp``` config file is quite a bit less verbose than that of ```isync```.

{{% steps %}}

### Installation
More of the same: ```apt install msmtp```.

### Configuration
```msmtp```'s config file lives by default at either ```~/.msmtprc``` or ```$XDG_CONFIG_HOME/msmtp/config```. I like having as much of my configuration as possible in ```~/.config``` and I like default configuration file locations, so I'll be writing my config at ```~/.config/msmtp/config```. An ```msmtp``` config file is organized into *accounts*. Each *account* describes the SMTP server and how to use it. Additionally, there are a few global options that can be set in the config file. We'll start with those.

#### Global Options
Here's the global section of the ```msmtp``` config:

```conf
defaults

auth on
tls on
logfile ~/.msmtp.log
```
Since this is a short section, we'll just go over the options together. ```defaults``` tells ```msmtp``` to treat the following options as defaults for all accounts defined later in the config. ```auth on``` enables authentication. Obviously we need to authenticate with the mail server. ```tls on``` should also be self explanatory, but for the sake of exhaustiveness it enables TLS. And finally, ```logfile ~/.msmtp.log``` sets the logfile location for ```msmtp```. Simple enough. Now let's move on to the interesting part: *accounts*.

#### Accounts
Here's the account definition for the same mail server we've been using throughout the tutorial:

```conf
account mxroute
host glacier.mxrouting.net
port 587
from cole@hohosunbear.com
user cole@hohosunbear.com
passwordeval "gpg -dq $HOME/.pass/mxroute_pass.gpg"
```
Looks familiar doesn't it? This should be old news after ```isync```.

```conf
account mxroute
```
Much like ```isync``` we declare an *account* definition with the ```account``` keyword. We pass a human readable name to the account that we want to refer to it as. Keeping it consistent, I'm calling this *account* "mxroute".

```conf
host glacier.mxrouting.net
port 587
```
This is where we tell ```msmtp``` where the it's connecting to is. You'll need to get this information from your mail server. Port 587 is the standard port for secure SMTP mail submission, so unless you are using a non-standard setup, you'll probably be using 587 as well.

```conf
from cole@hohosunbear.com
user cole@hohosunbear.com
passwordeval "gpg -dq $HOME/.pass/mxroute_pass.gpg"
```
Here we tell ```msmtp``` who we are. ```from cole@hohosunbear.com``` sets the *envelope-from* address in the mail. This is the "From" field that recipients will see when they read email you send with this account via ```msmtp```. ```user cole@hohosunbear.com``` is the username ```msmtp``` will use to authenticate with the mail server, and, just like with ```isync```, ```passwordeval "gpg -dq $HOME/.pass/mxroute_pass.gpg"``` passes a shell command that ```msmtp``` can use to get your password. And we're done! Since we don't have to deal with synchronizing local and remote inboxes, once we tell ```msmtp``` where to send and how to send it, we're free to blast emails out as we please.

### Conclusion
Emacs setup excepted, ```msmtp``` is the last of the moving pieces we needed to set up in order to have full email functionality in Emacs. There's a lot going on, but when you start to think about the system as a whole, it's quite elegant. This setup follows the Unix philosophy to a tee; each of the pieces does one job (and does it well), and can be seamlessly swapped out with another, equivalent piece with minimal changes. This extends even to the frontend you'd like to use to read and compose emails. Tired of ```mu4e```?  Switch to ```mutt``` and it will still work as expected. So, now that we've got the bones of the system working, let's move on to hooking it all up and writing emails in Emacs.

{{% /steps %}}

## mu4e
We're finally on to the fun part: working in our Emacs config. While we're not stuck learning and writing obscure configuration file formats anymore, our work is far from done. There's still some configuration to do, only now we get to write it in elisp. While I personally use Doom Emacs, I've written all the elisp in this article to be compatible with a vanilla install. 

{{% steps %}}

### Basic Settings
Here's what a simple ```mu4e``` configuration looks like:

```elisp
(require 'mu4e)

;; Mu4e settings
(setq mu4e-context-policy 'ask-if-none
      mu4e-compose-context-policy 'always-ask
      mu4e-update-interval nil
      mu4e-bookmarks
      '((:name "Unread" :query "flag:unread AND NOT flag:trashed" :key 117)
        (:name "Today's messages" :query "date:today..now" :key 116)
        (:name "Last 7 days" :query "date:7d..now" :hide-unread t :key 119)
        (:name "Messages with images" :query "mime:image/*" :key 112)))
```

The first thing to do is load ```mu4e``` into our Emacs. Depending on your setup and the package manager you're using, this will look a little different. The classic way is like so: ```(require mu4e)```. Once we've got the package loaded, we need to do some basic setup. There are a ton of options, but the few that I've selected should be a decent starting point for most people. Let's go through them one by one.

* ```mu4e-context-policy 'ask-if-none``` - This is a setting that you'll really only need if you decide to configure multiple email accounts. ```mu4e``` delineates accounts (and their requisite configurations) into what it calls *contexts*. It has a mechanism by which it will try to infer the context it should be using, and this setting tells it how to select a context when opening the main menu. The value I've selected, ```ask-if-none```, tells it to simply ask the user if it hasn't gotten a context by some other mechanism.

* ```mu4e-compose-context-policy 'always-ask``` - Similarly, this setting instructs ```mu4e``` how it should determine a context when composing an email. For simplicity's sake, I've chosen to have it always ask me which context to use. Hence, ```always-ask```.

* ```mu4e-update-interval nil``` - `mu4e` has a built-in mechanism to handle retrieving mail, and then indexing it. I'm actually handling this with a different mechanism which we'll get to later, so I've disabled the internal update mechanism. 

* `mu4e-bookmarks` - Because it's Emacs, `mu4e` offers *extensive* customization over its main screen. The `mu4e-bookmarks` variable allows you to specify queries you may run on your maildir ahead of time. The values above are the defaults, but rest assured, the options are unlimited.

At this point, you should already be able to run `mu4e` and browse your downloaded emails. It doesn't need to know anything about `isync` because we're not using the built-in mail fetching functionality. As long as the messages are in the maildir it expects them to be, it will dutifully show them to you. However, because it *does* need to interact with `msmtp` in order to *send* mail, we do have to tell it that.

### message/sendmail Configuration
Sending mail with `mu4e` is, perhaps unsurprisingly, a fairly complicated subject, mostly due to the multitude of different pieces that make up the pipeline, both on the Emacs level, and the OS level. `mu4e` sends email via the Emacs built-in `message-mode`. `message-mode`is a pretty bare bones email composition mode, and was originally intended to be used with the `GNUS`email/article reader...thing. It's funky. Anyway, because we're going to be sending mail with `msmtp`, we're going to have to tell *`message-mode`* to use `msmtp`. Unfortunately, that's a little circuitous. 

```elisp
(setq message-send-mail-function #'message-send-mail-with-sendmail
      send-mail-function #'sendmail-send-it
      sendmail-program (executable-find "msmtp")
      message-sendmail-f-is-evil t
      message-sendmail-extra-arguments '("--read-envelope-from")
      )
```

* `message-send-mail-function #'message-send-mail-with-sendmail` - In order to use a custom program to send mail with `message-mode`, we need to configure `message-mode` to use the *even more bare bones* `mail-mode` (which is in `sendmail.el`). Only then can we configure *`mail-mode`* to use `msmtp`. So here, we're just configuring `message-mode` to send its mail via `mail-mode`.

* `send-mail-function #'sendmail-send-it` - `mail-mode` has several different mechanisms by which it can infer, delegate, or otherwise modify the mail sending pipeline, but since we've already done the configuration work on the backend, we just want it to send the mail with the program its told to, that being `msmtp` shortly.

* `sendmail-program (executable-find "msmtp")` - `mail-mode` by default just invokes the `sendmail` utility that ships with nearly every Linux distribution. But, since we're using `msmtp`, we're going to set `mail-mode`'s executable to that instead. Now, finally, when we send a mail from `mu4e`, which invokes `message-mode`, which invokes ``mail-mode`, `mail-mode` will then invoke `/usr/bin/msmtp` instead of `/usr/sbin/sendmail`. Whew.

* `message-sendmail-f-is-evil t` - Here's a fun one. Due to the esoteric variable name, the documentation is quite a bit more detailed than most of the other ones:
  ```
  Non-nil means don't add "-f username" to the "sendmail" command line.
  
  The "sendmail" program has a useful feature to let you set the
  envelope FROM address via a command line option, "-f".
  Unfortunately, it also has a widely disliked default behavior of
  disclosing your actual user name anyway by inserting an
  unattractive warning in the headers.
  ```
  Yeah, we don't want `message-mode` to be messing with the command line arguments, so we'll definitely want to enable this option.
   
* `message-sendmail-extra-arguments '("--read-envelope-from")` - This variable allows us to specify extra command line arguments that we actually *do* want. Even though we literally just turned off the `-f` flag that's supposed to add the `smtp.mailfrom` (or envelope from) address, we still do want to set it, albeit correctly. The `msmtp` flag `--read-envelope-from` does just that. It sets the `smtp.mailfrom` address based on the header `FROM` address.

{{% /steps %}}
