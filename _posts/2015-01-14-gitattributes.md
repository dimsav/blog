---
layout: post
title:  "Use gitattributes to keep your tests out of other people's production"
date:   2015-01-14
author: hannes
categories: [package development]
---

A quick tip here, for all PHP developers out there who author packages registered on [packagist.org](https://packagist.org).
Add a `.gitattributes` file in the root of your github repository.

The `.gitattributes` file allows you to configure several things, but the most important thing related to composer is
the ability to not include certain files and folders into your release.

What I mean by that is the following. When composer installs your package with (that is with `--prefer-dist`) it will pull down a zip
file from github or whatever `cvs` you're using, and extract it. With the `.gitattributes` file you can exclude files and folders
to be included in that zip file by adding a line `/folder/file export-ignore`.

When people are using your package, they might not be interested to download your `/tests` folder, or your `.travis.yml` file.
Probably all they need is your `src` folder and your `composer.json` file.

Here's a sample `.gitattributes` file which shows you a lot of files and folders you might want to exclude from your dist-releases:

<script src="https://gist.github.com/hannesvdvreken/89bec532ffd59637156f.js"></script>

Even your `.gitattributes` file. How meta.

### Useful other stuff
There's other useful stuff you can control with the `.gitattributes` file. These things are not related to composer though.

You can let git convert every single end-of-line in text based files to linefeeds (LF). This removes carriage returns (convert CRLF to LF) and makes
sure you don't get weird diffs where the only difference is an invisible character. Enable this by adding `* text=auto` to your `.gitattributes` file.
If you want to specify a different end-of-line character on a file type basis, check the [docs](http://git-scm.com/docs/gitattributes#_end_of_line_conversion).

You can also control how [diffs are displayed](http://git-scm.com/docs/gitattributes#_generating_diff_text)
on a per file type basis using the `diff` keyword.

### References:

- More information on the .gitattributes file in the [git documentation](http://git-scm.com/docs/gitattributes)
- This discussion on reddit: [I don't need your tests in my production.](https://www.reddit.com/r/PHP/comments/2jzp6k/i_dont_need_your_tests_in_my_production)
