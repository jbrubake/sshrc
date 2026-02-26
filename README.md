# sshrc

`sshrc` works just like ssh, but also copies the contents of your local
`$SSHRC_HOME` directory to the remote server and sources the remote copy of
`$SSHRC_HOME/sshrc` in lieu of a remote `~/.bashrc file`. `$SSHRC_HOME` defaults
to `$XDG_CONFIG_HOME/sshrc`.

## Usage

    local$ echo "echo welcome" > ~.config/sshrc/sshrc
    local$ sshrc server
    welcome
    server$

    local$ echo "alias ..='cd ..'" > ~/.config/sshrc/sshrc
    local$ sshrc server
    server$ type ..
    .. is aliased to `cd ..'

You can use this to set environment variables, define functions, and run
post-login commands, allowing you to use the same configuration on multiple
remote servers without needing to configure them individually.

`sshrc` creates a unique remote copy of your local `sshrc` configuration for
each login.  This makes `sshrc` useful if you share a remote account with
multiple users as each user can login to the same remote account and have a copy
of their local `sshrc` configuration without conflict.

By default the initial shell spawned by `sshrc` does not source login shell
files (`/etc/profile` and `~/.bash_profile`). Use the `-j` option to load these
files.

## VIM

Create a `$SSHRC_HOME/vimrc` file and add an alias to your `sshrc` file to have
vim use this file as its configuration:

    # On the remote server $SSHRC contains the path to the remote copy of your
    # sshrc configuration
    local$ echo "alias vim='vim -u $SSHRC/vimrc'" >> ~/.config/sshrc/vimrc
    local$ sshrc server
    server$ type vim
    vim is aliased to `vim -u $SSHRC/vimrc'

## tmuxrc

Normally, using tmux in an `sshrc` session uses the remote `~/.tmux.conf` and
spawns shells that read your remote shell files. `sshrc` can optionally create a
tmuxrc framework that reads `$SSHRC_HOME/tmux.conf` if found and spawns shells
that read your `sshrc` file.

Simply use the `-u` option when starting `sshrc` and run tmux normally. If you
do not have a `$SSHRC_HOME/tmux.conf` file the local `~/.tmux.conf` will be read
instead.

**NOTE:** Do not set the tmux `default-command` option as this is set by tmuxrc
to spawn shells that read your `sshrc` file.

tmuxrc sessions will persist across multiple invocations of `sshrc`.

On initial login, `sshrc` will copy your `sshrc` files to a persistent tmuxrc
directory so that any detached tmux sessions can persist after the `sshrc`
session exits while still having access to your `sshrc` setup. These files are
not overwritten by a subsequent `sshrc` session unless there are no running
tmuxrc sessions.

### Multi-user & Multi-location tmuxrc

When `sshrc` creates a tmuxrc framework it uses a unique ID that is generated
and stored on the local machine. Any `sshrc` sessions using the `-u` option will
use this ID in order to connect to the same tmuxrc session.

If you want to connect to your own tmuxrc session from a different machine, use
the `-U [ID]` option where ID is the unique ID from the local system that
created the tmuxrc session. Use the `-d` option to display the local unique ID
before moving to the second local system.

    # From one system:
    box-1$ sshrc -u server
    server$ tmux
    # detach the session
    server$ sshrc -d
    ABCDEF

    # From another system:
    box-2$ sshrc -U ABCDEF server
    server$ tmux attach
    # you are attached to the previously detached session

If multiple users use `sshrc` to log into the same remote account and want to
share a tmuxrc session the `-U` option can be used but when any user ends their
`sshrc` session while there are no existing tmux sessions, the tmuxrc framework
will be deleted automatically.

Instead, one user should start tmuxrc normally and pass their unique ID to the
other users. This can be obtained before starting the `sshrc` session with the
`-d` option or with the remote `tmux_id` command:

    bob@local$ sshrc -d
    ABCDEF
    bob@local$ sshrc -u server
    bob@server$ tmux_id
    ABCDEF
    bob@server$ tmux
    # inside a tmux session to share

Other users can then connect using `sshrc` **without** the `-u` option and use
the remote `tmuxrc` command to connect:

    alice@local$ sshrc server
    alice@server$ tmuxrc ABCDEF [tmux options and commands]
    # inside the shared tmux session

# Advanced Details

When an `sshrc` session begins, a unique directory is created in `/tmp` and the
variable `$SSHRC` is set to the path to that directory. The contents of your
local `$SSHRC` are then copied to the remote `$SSHRC` directory.

Inside a tmuxrc session, `$SSHRC` is set to the path to the persistent tmuxrc
directory.

You will need to configure necessary environment variables or aliases for
programs other than bash or tmux to use files copied to `$SSHRC` (see the Vim
example above).

Putting too much data in `$SSHRC_HOME` will slow down your login times and if
the contents of `$SSHRC_HOME` are too large, the remote server may deny your
login attempts. Although the maximum allowable command length on modern systems
is generally 

Because the length of a command line is limited, `sshrc` refuses to attempt to
login if it attempts to upload > 1024 KiB.  This maximum size can be changed
with the `-Z` option.

Use the -z option to determine the total size of data that `sshrc` will attempt to
upload.  This includes the size of the `sshrc` bootstrap script which, along with
your `$SSHRC_HOME` directory is compressed using gzip(1).

## Message of the Day

By default `sshrc` will attempt to print the remote message of the day and print
the last login time. This can be suppressed by creating a remote `~/.hushlogin`
file (per `login(1)`) or be creating a `$SSHRC_HOME/.hushlogin` file.

## Tab Completion

To enable tab completion in bash, add `complete -F _comp_cmd_ssh sshrc` to your
`.bashrc` file.

