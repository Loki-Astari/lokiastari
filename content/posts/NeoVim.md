---
title: NeoVim Config on Apple Silicon
date: 2025-03-05T02:48:30-0800
author: Loki Astari, (C)2025
comments: true
categories:
- C++
- Vim
- NeoVim
- Config
- IDE
series:
- Vim
tags:
- Vim
subtitle: NeoVim IDE
description: NeoVim Config on Apple Silicon
draft: false
cover:
  image: /images/post/post-2.png
  hidden: false
  caption: Photo by ThisisEngineering RAEng
---

# NeoVim

I want to use neovim as my build environment for ***C++***.

Couple of pre-requisites before you even start.

* Install XCode (Download from the AppStore)
* Install the command line tools `xcode-select --install`
* Install brew from [brew.sh](https://brew.sh/)

## Installing
Though you can install **NeoVim** by package from [neovim.io](https://neovim.io/) I don't recommend this. This is because it will install an application built with the Intel CPU instruction set (this runs because of the emulation mode). This is fine for most operations, but when you use **NeoVim** as an IDE, it will build the code for the Intel processor. The better option is to install **NeoVim** from [Brew.sh](https://formulae.brew.sh/formula/neovim#default) as this will install the appropriate version of **NeoVim** for the current CPU.

Since I use the lazy package manager I also install **luarocks** at the same time.


```bash
brew install neovim
brew install luarocks
```

## Config

### Color Themes

You can find a good set of color themes [here](https://dotfyle.com/neovim/colorscheme/top):

### Cursor Visibility

I found that the cursor is not always very visible. I use the following command in my config to make the cursor visible in light and dark themes.

```vim
vim.cmd([[ set guicursor=n-v-c:block,i-ci-ve:ver25,r-cr:hor20,o:hor50,a:blinkwait700-blinkoff400-blinkon250-Cursor/lCursor,sm:block-blinkwait175-blinkoff150-blinkon175 ]])
```

### Tree Sitter

[Tree sitter](https://github.com/nvim-treesitter/nvim-treesitter) is a language parser that is extremely fast.

Once you have installed it (via your package manager) into NeoVim. You can install additional parsers for specific languages with `:TSInstall`

```vim
:TSInstall c
:TSInstall cpp
:TSInstall java
:TSInstall python

```

### Language Server




	

