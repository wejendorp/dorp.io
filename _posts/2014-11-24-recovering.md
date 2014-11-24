---
layout: post
title:  "Recovering my Mac: creating my install script"
date:   2014-05-20 08:54:00
categories: blog
---

I recently had my Macbook stolen, and thought I'd seize the opportunity to redo
my development setup.

So I went with homebrew and cask for my apps, "OSX for hackers" and stow for the
rest.


  * Blug

<!--more-->

## A simple list of apps
Realizing that practically all my applications and commandline tools are available
via [homebrew](http://brew.sh/) and [cask](http://caskroom.io/), I knew I had to
create a single manifest for all my apps.

### Preliminaries
At the time of writing, homebrew is installed with

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Cask is installed via brew:

```bash
brew install caskroom/cask/brew-cask
```


**Note**: You might need to let brew prepare your environment before continuing.

```bash
brew doctor
```

And just for good measure, make sure you got the latest brew and cask fomulae:

```bash
# update formulae before brewing!
brew update
brew cask update
```



### First couple of apps
The first things first. As a rule of thumb, binaries are found through `brew`,
and apps are found through `brew cask`. Let's get the basic apps out the way.
I like to use the newest browsers, so I'm going to "unlock" (tap) the
alternate versions repository, to get chrome canary and friends:

```bash
# browsers
brew tap caskroom/versions
brew cask install google-chrome-canary firefox-nightly
# An editor, and a better terminal
brew cask install atom iterm2
# etc...
brew cask install ...
```

See the rest of my cask-apps in my dotfiles repo.. LINK IT UP!

Most apps are available through cask. At the time of writing, about 2100 apps
can be installed in this way from the primary repository alone. So I always
check if there's a cask available for any tool I'm about to install.

### 

Installing binaries with brew

installing apps with cask

Language-related tooling..

First thing to figure out was dotfiles.. awesome-dotfiles.
GNU Stow..
dotfiles repo or dropbox folder..
http://brandon.invergo.net/news/2012-05-26-using-gnu-stow-to-manage-your-dotfiles.html

Migrating to zprezto from ohmyzsh...
Custom theme..

OSX for hackers
Customizing


The primary resources
