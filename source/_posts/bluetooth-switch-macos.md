---
title: Switching bluetooth on and off on macOS via Spotlight
author: Alexander Erben
date: 2017-09-19 12:29:56
tags: mac
desc: My new headphones keep connecting to my MacBook even though it is in stand-by mode. But I hacked my way around this issue.
---

I recently bought new [noise cancelling headphones](https://www.google.de/search?q=bose+quietcomfort+35) to deal with the turbulent nature of the open space office I work in. They are able to connect to multiple devices at once. This is an awesome feature which allows me to seamlessly switch from listening to music on my mobile phone while commuting, and then taking calls in Slack after arriving at work.

The only downside: the headphones connect to my MacBook even if it is in stand-by mode when I carry it in my backpack. This usually wouldn't be much of an issue, maybe except from the battery drain. But my MacBook keeps blocking the mobile phone from streaming music over bluetooth to my headphones for whatever reason. It usually just does that when a Slack or Skype call starts, but when in stand-by, it appears to be in voice call mode all the time. This might be a bug in either macOS or the headphone's firmware - I couldn't find out what's responsible for this behaviour.

But I found a stupid hack around this issue that works well enough for me. I decided that it would be convenient enough for me if I were able to switch bluetooth on my Mac on and off with a shortcut. That way I can enable bluetooth before I take calls, because the headphones automatically connect to the Mac when bluetooth connection is available. As an added bonus, disabling bluetooth while I don't need it conserves battery life.

Sadly, macOS has no built-in shortcut to enable or disable bluetooth. But using Spotlight, it is still possible to find a feasible solution. On macOS, it is possible to place shell scripts in the `/Applications`-folder and make them discoverable by Spotlight when the file ends with the `.command`-extension. 
The only thing I needed was a command line util to disable and enable bluetooth. [`blueutil`](http://www.frederikseiffert.de/blueutil/) does this job very well.
So here is my solution:

```
$ /Applications cat blueoff.command
#!/bin/sh
blueutil off

$ /Applications cat blueon.command
#!/bin/sh
blueutil on
```

Et voilà, I can now disable and enable bluetooth by simply typing `blueoff` and `blueon` in Spotlight. The next step would be to run these scripts before and after going to stand-by. Until now, I haven't found any build-in hook in macOS to execute scripts that way. If you find one, feel free to send me a message.