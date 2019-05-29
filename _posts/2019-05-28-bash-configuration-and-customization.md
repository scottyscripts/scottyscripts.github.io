---
layout: post
title:  "Bash Configuration and Customization"
date: 2019-05-28
---

## What the Shell?

`Bash` is the default shell for many operating systems including MacOS, Ubuntu, Kali, etc. It can even be used on Windows! (Finally!)

I think it's extremely important to have an understanding of how your shell is working. This post will give a high level description of how `Bash` can be configured and customized.

Keep reading if you want to learn how to effectively manage your shell environment, customize your shell's appearance, and save massive amounts of time.

![Crash Bash](/assets/img/crash_bash.png)

## Getting Started

In this post, we will take a look at the following files.
- `.bash_profile`
- `.bashrc`
- `.bash_aliases`

First off, take a look in your home directory to see if they exist.

```shell
cd ~
ls -a
```

If you don't see the above 3 files...

1. verify that you are running `Bash`
```shell
echo $SHELL
# => /bin/bash
```


2. create them!
```shell
touch ~/.bash_profile ~/.bashrc ~/.bash_aliases
```

## Scripts Everywhere

![Scripts Everywhere](/assets/img/scripts_everywhere.jpg)

Note that all of these dotfiles are really just plain ol' `Bash` scripts that are either run automatically when `Bash` is started or manually sourced.

This means that all these files should contain valid `Bash` syntax (unless you like errors).

## .bash_profile

This script is automatically executed for login shells. This means that the code it contains will run any time a shell is created by logging in.

### Um... What's a Login Shell?

1. Shell spawned by switching to a new user (`su someuser`)

2. Shell spawned using ssh (`ssh someuser@anothermachine`)

3. Any shell from `MacOS`'s Terminal application. (Terminal on Mac OS spins up new shells as login shells.)

## .bashrc

This code in this script is automatically executed for interactive non-login shells. If you are already logged into your machine and open a new terminal window, it will be a non-login shell (__unless you're using MacOS which spawns login shell by default__).

## .bash_aliases

This script is used to seperate concerns and contain all `Bash` aliases in a single file.

An `alias` is a shortcut for a command to use in our shell.
For example, if I find myself constantly typing `git status`, I can create an `alias` in `Bash` by doing the following.
```shell
alias gs='git status'
```
Now I will only have to type `gs` and the command `git status` will run.

These aliases can be defined in `~/.bashrc` or `~/.bash_profile`, but I recommend utilizing `~/.bash_aliases` to keep them in a dedicated file.

### Remember To Source It

Just putting your aliases in this file will not make them available to you in your shell. The file will need to be manually sourced. In your `.bash_profile` or `.bashrc` (depending on the type of shell that will be run) add the following

```shell
if [[ -f ~/.bash_aliases ]]; then
  source ~/.bash_aliases
fi
```
OR
```shell
test -f ~/.bash_aliases && . ~/.bash_aliases
```

They both mean the same thing:
"If the file ~/.bash_aliases exists, `source` it"
- We want to `source` the script instead of running it so it will run in the same shell process.

## Example Time

I'll share some examples of how I utilize these files.

### .bashrc / .bash_profile Examples

1. I source some files related to [asdf](https://github.com/asdf-vm/asdf)
    ```bash
      . $HOME/.asdf/asdf.sh

      . $HOME/.asdf/completions/asdf.bash
    ```
  After installing some programs such as asdf, the postinstall message will tell you to add additional config to your `.bashrc` / `.bash_profile`.


2. I modify my `PATH` to add programs I've installed that don't live in default `PATH` destinations.
  (If you are not sure about what the `PATH` environment variable does, I highly recommend hitting the Googles after reading this. I may write a blog post about `PATH` at some point in the future)

    ```bash
    export PATH=$PATH:/opt/metasploit-framework/bin
    ```

3. I define some ENVs for colored output in my terminal.

    ```bash
    export RED='\033[0;31m'
    export GREEN='\033[0;32m'
    export BLUE='\033[0;34m'
    export NC='\033[0m'
    ```

    Now in my shell (on MacOS) or scripts I can type

    ```bash
    echo -e "Elements are ${RED}fire${NC} ${BLUE}water${NC} and ${GREEN}grass${NC}"
    ```

    for some colorized output.

4. I modify `Bash` specific variables like `PS1` for a more custom command line prompt. (This works for MacOS)

    ```bash
    function parse_git_branch {
      ref=$(git symbolic-ref HEAD 2> /dev/null) || return
      echo "("${ref#refs/heads/}")"
    }

    export PS1="${BLUE}\w${NC} \$(parse_git_branch)\n\$"
    ```

    Now, instead of my command line prompt looking like

    ```bash
    Scotts-Computer-Name:mydirectory myusername$
    ```

    I see

    ```bash
    ~/path_to_cwd (name-of-git-branch)
    $
    ```

### .bash_aliases Examples

I LOVE me some `Bash` aliases. I'm always trying to limit my keystrokes to avoid Carpal Tunnel Syndrome / I'm lazy.

Heres some examples from my `.bash_aliases` file (I like to put functions there too). I can only confirm that these work on MacOS.

```bash
# list size of each directory within your current dir
alias dirmem="du -hd1"
# print the current git branch
alias branch="git symbolic-ref --short -q HEAD"
# git push the current branch to origin
alias gpo="git push origin $(branch)"
# When I was teaching new developers, I never wanted to type this command so I made this alias
alias byefelisha="rm -rf"
# women are equal to men! Wanted to access man pages with equality
alias woman='man'
# if you use postgres for MacOS, I'm sure you've run into this one before
alias pgfix='rm /usr/local/var/postgres/postmaster.pid'

# get the time in UTC
alias utc="date -u | awk '{ print \$4 }'"

# open chrome from command line
alias chrome="/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome"

# pass path to CSV file as an argument and see how many lines CSV file is
function csv_count() {
  awk '{n+=1} END {print n}' $1
}

# replace YourOrgName and pass any search term as argument to this function
# paste the URL copied to your clipboard into address bar
# to search your Github Org's repos for a specific term in the code
function ghsearch() {
 echo "https://github.com/search?q=org%3AYourOrgName+$1&type=Code"
 echo "https://github.com/search?q=org%3AYourOrgName+$1&type=Code" | pbcopy
}

# counts number of files in a given directory / its subdirectories
function count_files(){
  if [ $# -eq 0 ]; then
    echo "Usage: count all files recursively in directory."
    echo "Pass directory path as first argument."
  else
    find $1 -type f | wc -l
  fi
}

# pass ssid of a network you have connected to as first argument to return its password
# I hate looking for piece of paper with my wifi password when people come over
function wifi_passwd(){
  security find-generic-password -D "AirPort network password" -a "${1}" -g | grep "password:"
}
```

I have countless more aliases and functions on my machines, but I tried to select a decent variety.

## Conclusion

Hopefully this post helps you understand more about `Bash` configuration and customization. I encourage everyone to continue to learn more about `Bash` and to get creative customizing your shell. It will save you so much time in the long run and help you to be more efficient.

I constantly hear that it's a good idea to come out of our shells, but that isn't always possible when you're a programmer.
