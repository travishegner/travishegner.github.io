---
title: 'Running Multiple Pass Repositories'
author: travis.hegner
layout: post
---
I've been using [Pass: The standard unix password manager][1] to store and manage all of my passwords for quite a while now. It is essentially a shell script wrapper around many other standard \*nix tools, but it makes the task of password management a lot easier to do. It basically stores all of your passwords (and additional info, if you so choose) as flat text files encrypted with GPG. It has tasks for viewing, editing, and generating new passwords very easily. One of it's best features is that it can treat your password store as a git repository, so that historical passwords are saved, and it's easier to sync your whole password database between multiple machines.

I've <del>finally</del> recently convinced my co-workers that we should use the pass system to share all of our work passwords. This would allow us to stop using the same password for everything, while simultaneously allowing us to easily keep each other comprised of the current password for each device. I quickly found myself in a dilemma though, because I wanted to keep my personal pass repository, as well as my work one, on the same machine.

Pass defaults to using `~/.password-store` for it's storage directory. When executing any of the `pass` commands, it uses that directory to look for your encrypted files, and do tab completion. I really wanted to keep both repositories available, so what could I do?

Digging through [man page][2], I discovered that I can set an environment variable to control the context of the password store directory. Helpful, yes, but kind of a pain to have to switch the environment variable back and forth in order to get the info from the correct repository. So I decided to set up some custom functions to allow me to query either password repository on a whim.

{% highlight bash %}
tpass() {
  PASSWORD_STORE_DIR=/home/thegner/git_repos/passdb pass "$@"
}

#for working autocomplete:
compdef \_tpass tpass
\_tpass() {
  PASSWORD_STORE_DIR=/home/thegner/git_repos/passdb \_pass
}
{% endhighlight %}

With those two functions added to your `.zshrc`, you can easily query a second password repository as you see fit. When I want to query my second password repository, I simply use the `tpass` command in place of the `pass` command. The second function even provides a working auto-complete for commands and filenames on the second repository. You could add as many password repositories as you desire by adding these two functions for each one with unique names.

[1]: http://www.passwordstore.org/
[2]: http://git.zx2c4.com/password-store/about/
