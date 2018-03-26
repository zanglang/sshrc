# sshrc

Bring your config files with you

## Description

This is a fork of [Russell91](https://github.com/Russell91)'s awesome [sshrc](https://github.com/Russell91/sshrc) utility. The original goal of this project was to refactor Russell91's script to make it more readable and more modular. Since then, I have also introduced some additional features to the script.

### Improvements

- Help text and man page entry
- Supports more than one base64 encoding utility
- Configurable compression utility (--archiver); bzip2 generally seems to work best in my experience
- Support for defining your own shell wrapper; this can be used, for example, to start zsh instead of bash
- Better MOTD support
- More files can be included in .sshrc.d
  - The original sshrc limits you to 64KB because this is the maximum command size on some machines; not all machines are limited to 64KB though, so this version makes the check configurable (--arg-max)
  - This version allows not uploading the script itself to the remote server (--no-bin); this provides a small amount of extra space
  - The --arg-size flag allows you to see how big the command will be
- Debug flags that give more insight into the inner workings of the script

## Usage

sshrc works just like ssh, but it also sources the ~/.sshrc on your local computer after logging in remotely.

    $ echo "echo welcome" >> ~/.sshrc
    $ sshrc me@myserver
    welcome

    $ echo "alias ..='cd ..'" >> ~/.sshrc
    $ sshrc me@myserver
    $ type ..
    .. is aliased to `cd ..'

You can use this to set environment variables, define functions, and run post-login commands. It's that simple, and it won't impact other users on the server - even if they use sshrc too. This makes sshrc very useful if you share a server with multiple users and can't edit the server's ~/.bashrc without affecting them, or if you have several servers that you don't want to configure independently.

## Installation

    $ wget https://raw.githubusercontent.com/taylorskalyo/sshrc/master/sshrc
    chmod +x sshrc
    sudo mv sshrc /usr/local/bin #or anywhere else on your PATH

## Advanced configuration

Your most import configuration files (e.g. vim, inputrc) may not be bash scripts. Put them in ~/.sshrc.d and sshrc will copy them to a (guaranteed) unique folder in the server's /tmp directory after login. You can find them at `$SSHHOME/.sshrc.d`. You can usually tell programs to load their configuration from the $SSHHOME/.sshrc.d directory by setting the right environment variables. Putting too much data in ~/.sshrc.d will slow down your login times. If the folder contents are > 64kB, the server may block your sshrc attempts.

### Vim

    $ mkdir -p ~/.sshrc.d
    $ echo ':imap <special> jk <Esc>' >> ~/.sshrc.d/.vimrc
    $ cat << 'EOF' >> ~/.sshrc
    export VIMINIT="let \$MYVIMRC='$SSHHOME/.sshrc.d/.vimrc' | source \$MYVIMRC"
    EOF
    $ sshrc me@myserver
    $ vim # jk -> normal mode will work

If you want to load your .vim folder as well, you can 1) put the .vim folder in ~/.sshrc.d, 2) move your .vimrc into the .vim folder, 3) edit the path above to reflect the new .vimrc location, and 4) add the following lines at the top of the moved .vimrc, which will notify vim of the .vim folder location:

    " set default 'runtimepath' (without ~/.vim folders)
    let &runtimepath = printf('%s/vimfiles,%s,%s/vimfiles/after', $VIM, $VIMRUNTIME, $VIM)
    " what is the name of the directory containing this file?
    let s:portable = expand('<sfile>:p:h')
    " add the directory to 'runtimepath'
    let &runtimepath = printf('%s,%s,%s/after', s:portable, &runtimepath, s:portable)

### Tmux

If you use tmux frequently, you can make sshrc work there as well. The following seems complicated, but hopefully it should just work.

    $ cat << 'EOF' >> ~/.sshrc
    alias foo='echo I work with tmux, too'
    
    tmuxrc() {
        local TMUXDIR=/tmp/mytmuxserver
        if ! [ -d $TMUXDIR ]; then
            rm -rf $TMUXDIR
            mkdir -p $TMUXDIR
        fi
        rm -rf $TMUXDIR/.sshrc.d
        cp -r $SSHHOME/.sshrc $SSHHOME/.sshrc.d $TMUXDIR
        SSHHOME=$TMUXDIR /usr/bin/tmux -S $TMUXDIR/tmuxserver $@
    }
    export SHELL=`which bash`
    EOF
    $ sshrc me@myserver
    $ tmuxrc
    $ foo
    I work with tmux, too

The -S option will start a separate tmux server. You can still safely access the vanilla tmux server with `tmux`. Tmux servers can persist for longer than your ssh session, so the above `tmuxrc` function copies your configs to the more permenant /tmp/russelltmuxserver, which won't be deleted when you close your ssh session. Starting tmux with the SHELL environment variable set to bashsshrc will take care of loading your configs with each new terminal. Setting SHELL back to /bin/bash when you're done is important to prevent quirks due to tmux sessions having a non-default SHELL variable.

### Specializing .sshrc to individual servers

You may have different configurations for different servers. I would recommend putting configurations in separate directories and then specifying which one to use with `SSHHOME`:

    SSHHOME=~/.ssh/server1 sshrc server1
    # or
    cd ~/.ssh/server1 && sshrc server1

You can also use the following structure for your ~/.sshrc control flow:

    if [ $(hostname | grep server1 | wc -l) == 1 ]; then
        echo 'server1'
    fi
    if [ $(hostname | grep server2 | wc -l) == 1 ]; then
        echo 'server2'
    fi

### Tips

* I don't recommend trying to throw your entire .vim folder into ~/.sshrc.d. It will more than likely be too big.

* You can avoid duplication of dotfiles using symlinks (e.g. $ cd ~/.sshrc.d && ln -s ../.tmux.conf .tmux.conf/ ).

* For larger configurations, consider copying files to an obscure folder on the server and using ~/.sshrc to automatically source those configurations on login.

* To enable tab completion in zsh, add `compdef sshrc=ssh` to your .zshrc file:
