---
layout: post
title: "Octopress 2.0 on cygwin"
date: 2017-04-12 19:29:24 +0200
comments: true
tags: ["octopress", "misc"]
---

Today I got working Octopress 2.0 on a Cygwin environment. I'm going to share some hacks needed to make it work.

<!-- More -->
## Packages needed

When installing Cygwin, select at least this optional packages:

```
git
ruby
ruby-bundler
ruby-rake
ruby-devel
ruby-pkg-config
ruby-bigdecimal
ruby-json
cc-core
gcc-g++
libffi-devel
make
autoconf
automake
```

We need this to compile the native gems Octopress needs.

## Generate bin stubs

Cygwin comes with a more modern version of rake that the one Octopress requires... so, any rake command execution will give:

```
$ rake preview
rake aborted!
Gem::LoadError: You have already activated rake 12.0.0, but your Gemfile requires rake 10.5.0. Prepending `bundle exec` to your command may solve this.
```

Prepending 'bundle exec' works, but you can also generate binary stubs to run the correct rake version in your Octopress directory:

```
$ gem install rubygems-bundler
$ gem regenerate_binstubs
```

Now, rake command should execute the correct version.

## Hack: chcp stub

When I tried to execute 'rake preview' I got an error like this:

```
$ rake preview
## Set the codepage to 65001 for Windows machines
rake aborted!
Errno::ENOENT: No such file or directory - chcp
```

Well, it seems rake detects it's running on Windows and tries to make a chcp... but chcp is not available and it isn't necessary on Cygwin. We will make a chcp stub to make rake happy:

```
$ cd /usr/local/bin
$ echo "echo Skipping CHCP request on cygwin" > chcp
$ chmod +x chcp
```

## Hack: include $HOME/bin in your PATH

2nd attempt to run 'rake preview' with our chcp stub gives another error

```
$ rake preview
## Set the codepage to 65001 for Windows machines
Starting to watch source with Jekyll and Compass. Starting Rack on port 4000
rake aborted!
Errno::ENOENT: No such file or directory - jekyll
```

When we installed all the Octopress gems, some executable went to $HOME/bin, but this directory is not included in cygwin PATH.

Next step is easy: edit your .bashrc file and add the following line to the end:

```
export PATH="$PATH:$HOME\bin"
```

Now exit from your cygwin console, open another one and try 'rake preview' again. Now it runs, but it doesn't generate new pages...

## 'rake generate' fails

If you try to make a 'rake generate', it will fail with the following error:

```
$ rake generate
## Set the codepage to 65001 for Windows machines
## Generating Site with Jekyll
    write source/stylesheets/screen.css
/home/GorkaE/.gem/ruby/2.3.0/gems/jekyll-2.5.3/lib/jekyll/filters.rb:2:in `require': cannot load such file -- json (LoadError)
```

'rake preview' doesn't fail, but it outputs some errors and it doesn't generate new pages.

In my case, I had a json gem installed, but this was not present in the project Gemfile. I solved adding 
```
gem 'json'
```
to Gemfile. I suspect json was not required when I started this blog, and a Jekyll update broke this (as I could see [here](https://github.com/lede-project/web/issues/35))

## Why all this if I can run Octopress on Windows native ruby?

Well, I prefer cygwin because this way I get a lot of synergies between different UNIX-like tools that make my workflow similar on Unix and Windows... but it's a question of tastes.

See you on the next hack!
