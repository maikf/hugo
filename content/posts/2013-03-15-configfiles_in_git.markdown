---
layout: post
title:  "configfiles in git"
date:   2013-03-15 22:17:40
categories: Artikel
Aliases:
  - /Artikel/configfiles_in_git
---



<p>
About 4 years ago, I started tracking my configuration files with git. The
advantages of storing configuration files in some repository are numerous:
</p>
<ol>

<li>
You can destroy your configuration/computer and easily revert to a known good
state.
</li>

<li>
You can easily distribute <strong>and update</strong> the same set of
configfiles across multiple machines (especially virtual machines or other test
setups).
</li>

<li>
You can refer other people to your configfiles if you store them in a public
repository.
</li>
</ol>

<p>
This article describes the way I use git to track <a
href="http://code.stapelberg.de/git/configfiles">my configfiles</a>. My
solution is simple and should be easy to understand even if you have not used
git extensively before.
</p>

<p>
<strong>Please note</strong> that I am not trying to convince you to use my
script(s), or my configfiles. Also, I am not going to support this — if you
have any troubles with it, you have to debug it on your own. I am merely
describing what works fine for me, in the hope that it will be useful to
somebody else.
</p>

<h2>initialize.sh</h2>

<p>
Obviously, the first step to get the configfiles repository on a “fresh”
computer is to clone the git repository: <code>git clone
git://code.stapelberg.de/configfiles</code>
</p>

<p>
Afterwards, run the script initialize.sh: <code>cd configfiles &amp;&amp;
./initialize.sh</code>. This script will create a corresponding dotfile for
each file in your configfiles repository, e.g. when you have
“configfiles/foorc”, it will create “~/.foorc” as a symbolic link:
</p>

<pre>
$ ls -l .zshrc
lrwxrwxrwx 1 michael staff 32 2010-09-30 07:57 .zshrc -> /home/michael/configfiles/zshrc
</pre>

<p>
Every already existing file, e.g. your current ~/.zshrc, will be preserved in
~/.configfiles.bak.
</p>

<h3>Files located in subdirectories</h3>

<p>
While this approach works well for a large number of configfiles, there are
exceptions. MPlayer for example stores its configfile in ~/.mplayer/config.
Since folders are reserved for a different purpose in my way of doing things,
the solution for this problem is a file called <code>MAPPING</code>. It
contains a simple lookup table of (filename → path) pairs, e.g. <code>mplayer.config
~/.mplayer/config</code>. initialize.sh takes care of creating the destination
parent folders if necessary.
</p>

<p>
Furthermore, you can use that mechanism to group files together by giving them
a filename with common prefix. As an example, all my emacs-specific configfilse
start with “emacs-”, e.g. “emacs-init.el” and “emacs-zkj-notmuch.el”. I find
this much clearer than just naming the file configfiles/init.el.
</p>

<h3>Host-specific files</h3>

<p>
If you have files which are specific to a certain host, you can store them in a
folder named like your host (use “hostname” to see how the folder needs to be
called precisely). To have a standard ~/.Xmodmap, but overwrite it on a single
machine called midna (with a weird keyboard), use configfiles/Xmodmap and
configfiles/midna/Xmodmap.
</p>

<p>
I have read about other people using branches to have different configfiles for
different hosts. While that may be a more complete solution for some extreme
cases, I find it way too hard to manage.
</p>

<h2>update.sh</h2>

<p>
In order to actually update the configfiles repository, one would normally run
<code>git pull</code>. To make this happen automatically, I call update.sh from
my zshrc:
</p>

<pre>
cfgfiles=$(dirname $(readlink ~/.zshrc))
# If the configfiles are in a git repository, update if it’s older than one hour
find $cfgfiles -maxdepth 1 -name .git -mmin +60 -execdir ./update.sh \;
</pre>

<p>
Whenever I log into a system which is automatically managed, it will update the
configuration files. On the other hand, if I don’t use a system, it will not
spend any bandwith/cpu time on updating.
</p>

<p>
The script update.sh itself is a wrapper around <code>git pull</code>, but has
a few nice properties:
</p>

<ol>
<li>
Only one instance of the script will run, so that you can open multiple
shells/terminal emulators quickly and not have any race conditions.
</li>
<li>
The script uses <code>git stash</code> to preserve your local, uncomitted
changes.
</li>
<li>
Instead of operating inplace (and breaking programs that run precisely when git
replaces a file, e.g. your shell), the script works in a temporary directory.
</li>
<li>
It runs the update in the background so that you don’t have to wait for a
terminal emulator window to open. This implies that the shell process which
triggered the upgrade will not pick up changes to zshrc automatically.
</li>
</ol>

<h3>Detecting and handling errors</h3>

<p>
Because update.sh runs its upgrade in the background, it will not post any
messages about success or failures to stdout/stderr — that might interrupt what
you are currently doing in foreground. Instead, it is entirely quiet in case
everything is okay.
</p>

<p>
When something fails (e.g. the server which hosts your git repository is down),
update.sh will create a file called ERROR in your configfiles directory. My
zshrc is configured to pick this up in its prompt and display it prominently in
red:
</p>

<pre>
setup_prompt() {
    local _cfg_nag

    if [ -f "$(dirname $(readlink ~/.zshrc))/ERROR" ]; then
        _cfg_nag="%F{red}cfg-git-error%f "
    else
        _cfg_nag=""
    fi

    PROMPT="%K{cyan}%F{black}%m%k%f ${fg_green}%~${fg_no_colour} \$(get_git_prompt_info)$_cfg_nag$ "
}

setup_prompt
</pre>

<p>
So whenever something is wrong, you will end up with a prompt like this:
</p>

<img src="/Bilder/zsh-cfg-git-error.png" alt="zsh: cfg-git-error">

<p>
In order to figure out what went wrong, view the file last-update.log in the
configfiles folder. Afterwards, just delete the file ERROR.
</p>

<h2>Committing changes</h2>

<p>
There is nothing special to watch out for when committing your changes. That
is, you could edit ~/.zshrc, then go to the configfiles directory and commit
your changes with <code>git commit -a</code>. Don’t forget to <code>git
push</code> your changes afterwards. Also, <code>git add -p</code> might come
in handy to only add specific parts of a file.
</p>

<h2>Pitfalls</h2>

<p>
In order to help with trouble-shooting, here are a few mistakes I have done in
the past:
</p>

<ul>
<li>
I thought it’d be a clever idea to check out the configfiles repository once in
/etc/configfiles and run initialize.sh for each user. The obvious problem is
that as a user, you don’t have permission to write to /etc, thus you cannot
upgrade. So this idea only works out when you log in as root by default, which
you should avoid.
</li>

<li>
You should not use <code>git stash</code> on your own in the configfiles
repository. update.sh tries to stash any changes and will then just
unconditionally try to pop the latest stash entry after pulling.
</li>
</ul>
