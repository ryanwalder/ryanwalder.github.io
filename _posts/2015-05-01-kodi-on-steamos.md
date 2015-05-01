---
layout: post
title: Getting Kodi running on SteamOS
permalink: kodi-on-steamos
comments: true
---

This is a quick guide on how to get Kodi (formerly xbmc) running on [SteamOS](http://http://store.steampowered.com/steamos), this method is a little hacky but it gets the job done and you'll be watching your media in no time. As I use this a NUC under my telly for my gaming/viewing habits I have no need for the gnome desktop environment that gets installed so I replace it with Kodi, this means I just switch to "desktop mode" from within Steam to fire up Kodi and exit Kodi to get back to Steam, it's not perfect but it works and serves all my needs.

## Setup sources

First you'll need to ssh onto your Steam box (or drop to a console), add in the Kodi apt sources file, and install Kodi

    echo "deb http://mirrors.xbmc.org/apt/steamos alchemist main" | sudo tee /etc/apt/sources.list.d/xbmc.list
    wget -O - http://mirrors.xbmc.org/apt/steamos/steam@xbmc.org.gpg.key | sudo apt-key add -
    sudo apt-get update
    sudo apt-get install -y kodi

## Changing the desktop to Kodi

Now Kodi is installed we just need to switch the desktop environment Steam switches to

    sudo sed -i 's/gnome\-session/kodi\-standalone/g' /usr/share/xsessions/gnome.desktop
    sudo reboot

All done, you can now fire up Kodi by exiting to the "desktop" from Steam and you're off to the races.
