---
layout: post
title:  "Migrating to tmux as a terminator user"
date:   2017-07-02
categories: Software, Linux
---

I've been a [Terminator](https://gnometerminator.blogspot.com/p/introduction.html)
user for a long time, and have nothing but praise for the project. The key
feature that Terminator offered to me was the simple configurability of terminal
emulators like the default Gnome or XFCE terminals, but also allowed for usage
of split panes.  I've seen much praise for [tmux](https://github.com/tmux/tmux/wiki)
in the past, but after trying it several times in the past I've always had
really large issues with the default keybindings (for real, how are % and "
representative of vertical and horizontal splits?).  However, I've finally become
annoyed enough with my `ssh` workflow that I decided to give it another shot.
After a couple days my workflow feels quite natural.  Two main factors that
pushed me over the edge have been

  1. maintaining sessions and being able to attach and detach at will
  2. being able to split shells within the same session (no more having to
     re-ssh in)

These are the key portions of my basic configuration.

# Overall niceties

Before getting too into keybindings, there were some basic things that needed to
be taken care of.  First, we set mouse support for when others need to do the
point and click thing.  Then, fixing some coloring, and adding the lovely
[tmuxline](https://github.com/edkolev/tmuxline.vim) to match my `vim` setup.
Finally, I noticed that there was some delay between switching modes in `vim`,
so I had to set an escape timeout to make things fluid. The relevant lines are

{% highlight shell%}
  set -g mouse on
  # don't rename windows automatically
  set-option -g allow-rename off
  # enable true color support
  set -ga terminal-overrides ',*:Tc'
  set -g default-terminal "tmux-256color"
  # load in the pretty tmuxline
  if-shell "test -f ~/.tmuxline" "source ~/.tmuxline"
  # fix escape for the sake of vim
  set -sg escape-time 0
{% endhighlight %}

# Motion Keybinds

Then, we come to the real issue: navigation and pane management. As said, I was
a previous Terminator user and grew accustomed to the keybinds for splitting and
resizing.  These don't require any prefix key-chord to be used, so I set them up
the same way for tmux, and things seem to work well enough.  On top of that, I
use [Spacemacs](https://github.com/syl20bnr/spacemacs) due to evil- and
org-mode, so I wanted to have the prefix situation be similar.  At that point it
seemed like using `ctrl-space` was a much more natural choice than the default
of `ctrl-b`.  I also reassigned the prefix-level splitting operations to be
similar to `SPC \` and `SPC -` from Spacemacs.

{% highlight shell%}
  # clear bindings
  unbind C-b
  unbind '"'
  unbind %
  # nicer prefix
  set -g prefix C-Space
  bind Space send-prefix
  # splitting like spacemacs
  bind / split-window -h
  bind - split-window -v
  # do like terminator
  bind -n C-E split-window -h
  bind -n C-S-Left resize-pane -L 3
  bind -n C-S-Right resize-pane -R 3
  bind -n C-S-Up resize-pane -U 3
  bind -n C-S-Down resize-pane -D 3
  bind -n C-O split-window -v
  # move panes without prefix
  bind -n M-h select-pane -L
  bind -n M-l select-pane -R
  bind -n M-k select-pane -U
  bind -n M-j select-pane -D
  bind r source-file ~/.tmux.conf
{% endhighlight %}

# Resulting situation

It's always helpful for me to see what other people's configurations look like
in practice so here's a screenshot of what my current setup looks like

![](../../../../../imgs/tmux_example.png)

All of the configuration for my setup can be foundin my [dotfiles
repository](https://github.com/arbennett/dotfiles).


