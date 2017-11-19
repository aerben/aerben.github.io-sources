---
title: Thanks, GitHub!
date: 2017-11-16 19:00:00
tags: github
desc: GitHub is full of wonders. It just launched a new interesting feature that tries to make the world a bit safer.
---

GitHub is full of wonders. I just started to play around with some Hexo themes and plugins for my new blog. Then when I pushed some changes to the GitHub Pages repo, I receive an ominous mail.

Usually, the mails that I receive on GitHub contain notifications about Pull Requests or Issues that I follow. But this time, things are different.

![](/images/thanks-github/mail-content.png)

Well, as it turns out, hexo depends on the no longer maintained [swig](https://github.com/paularmstrong/swig) template engine, which itself depends on an outdated and [vulnerable version](https://www.cvedetails.com/vulnerability-list/vendor_id-16037/product_id-35638/version_id-206779/Uglifyjs-Project-Uglifyjs-2.4.23.html) of uglifyjs.
Nice of GitHub to tell me. I would never have guessed that GitHub scans all source code in GitHub Pages-repositories for possible vulnerabilities. I guess it's more or less in their own interest as well as in mine.