title: "apt-get hold version"
date: 2015-06-16 22:56:47
description: 在ubuntu上进行update的时候，有的时候会不想升级某些包，这时可以用dpkg来hold住包，不让它升级。
categories:
    - linux
tags:
    - apt-get
    - ubuntu
---

If you want some specific package not to be processed（keep the current version with the current status whatever that is）, you can hold it.

## hold

```bash
echo "package_name hold" | dpkg --set-selections
```

## unhold

```bash
echo "package_name install" | dpkg --set-selecions
```

# References

1. [The Debian GNU/Linux FAQ
Chapter 7 - Basics of the Debian package management system](http://www.debian.org/doc/manuals/debian-faq/ch-pkg_basics.en.html)
