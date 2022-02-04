---
title: "Open Rubocop's results in VIM's quickfix"
description: "Open all Rubocop offenses in VIM's quickfix to easily navigate through them and address them."
tags: ["Productivity", "VIM"]
author: Sri Vishnu Totakura
layout: post
---

Once in a while, Rubocop introduces a new Cop and I would like to enable that
in our projects.
It usually finds many offenses that I can easily fix but the challenge is always
to keep the list in the terminal window and open the files one by one in my
editor.

VIM has [Quickfix](https://freshman.tech/vim-quickfix-and-location-list/)
lists that helps in cases like this.

The following is what I do to load the Rubocop results into VIM:

```console
$ bundle exec rubocop . -o rubocop-offenses-list
$ vim -q rubocop-offenses-list
```

The `rubocop-offenses-list` contains some unnecessary information like the
progress and etc., but that doesn't break the VIM's Quickfix list.

### Other resources
  * Learn more about how to navigate the results in Quickfix: [See this article](https://freshman.tech/vim-quickfix-and-location-list/)
  * Rubocop lists editor integrations and checkout their [VIM integration](https://docs.rubocop.org/rubocop/integration_with_other_tools.html#vim).



