---
layout: post
title: "re-build vim with homebrew"
date: 2013-02-17 17:41
comments: true
categories: anything
---

It seems Mac default vim doesn't have clipboard support.
So I reinstalled it by following [https://gist.github.com/dcosson/3686437](https://gist.github.com/dcosson/3686437).

<!-- more -->

``` sh
    hg clone https://vim.googlecode.com/hg/ vim
    cd vim
    hg update vim73
    make clean
    ./configure --prefix=/usr/local --enable-multibyte --enable-xim --enable-fontset --enable-rubyinterp --enable-perlinterp --enable-pythoninterp --with-features=huge --disable-selinux
    make
    sudo make install
```

The thing is after 'make' I got some linking error for ruby. (sorry no record left.)
To src/auto/config.mk, need to add at LDFLAGS : 

``` sh
    -L/Users/<user_name>/.rvm/rubies/<ruby_version>/lib/
```

I also got an error while 'sudo make install' for runtime/tools/efm_perl.pl, which was claimed as an issue here:
[http://code.google.com/p/vim/issues/detail?id=63](http://code.google.com/p/vim/issues/detail?id=63)
You just need to fix the garbling for copyright.

``` sh
    # Copyright (c) 2001 by Joerg Ziefle <joerg.ziefle@gmx.de>
```

And now successfully got compiled vim.

