# Terminal Multiplexer (tmux) How To Guide
 How To Setup TMUX for persistent terminal windows across SSH sessions.

## Terminal Multiplexer
SSH sessions interact with an active terminal window. When SSH disconnects that terminal session stops and any current action such as an executing script will terminate and possibly fail. This can be frustrating, for example if you begin compiling something that takes several hours only to have your client machine go to sleep or lose communications and terminate the session before completion. The terminal multiplexer (tmux) program provides an excellent workaround for this. 

 The Linux _tmux_ program provides persistent terminal windows for users after the starting terminal close. It connects your SSH session to a terminal client/server on the host that stays running even if the connection to the host ends. Additionally, it supports multiple windows and panes from a single SSH session and separeate sessions for individual users. I find this particularly useful for remote SSH sessions on headless nodes such as a Raspberry Pi.

## Install and run tmux
On A Raspberry Pi (or debian baed Linux)

    sudo apt-get install tmux

Once started a tmux session will run indefinitely. To reconnect after logging out and back in run:

    tmux attach

Or if multiple sessions are running run the following:

    tmux attach-session –t <session_name>

To detach from the session

    <Prefix-hot-key> D

And to kill a session 

    tmux kill-session [-t <target_name>]

## Autostart Setup
The following files automatically start tmux sessions on boot for each user having a defined _.tmux.init_ file
The process uses a systemd service to run the _/usr/local/bin/tmuxuser_ setup script. Define the files below:

### /etc/systemd/system/tmux.service
    [Unit]
    Documentation=https://tmuxguide.readthedocs.io/en/latest/tmux/tmux.html
    Description= tmux initialization per user based on /home/$USER/.tmux.init

    [Service]
    Type=forking
    RemainAfterExit=yes
    ExecStart=/usr/local/bin/tmuxuser start
    ExecStop=/usr/local/bin/tmuxuser stop

    [Install]
    WantedBy=multi-user.target

After defining the service update the daemon with

    sudo systemctl daemon-reload

Then enable it to run at boot by

    sudo systemctl enable tmux

Define the _/usr/local/bin/tmuxuser_ script that the service calls.

### /usr/local/bin/tmuxuser script
    #!/bin/bash
    # called as root by tmux.service on boot...
    # script to setup terminal multiplexer (tmux) with session named for each user with a defined /home/$USER/.tmux.init file

    # parameters…
    SCRIPT=/usr/bin/tmux

    # find users  with .tmux.init
    declare -a who
    for u in $( ls /home ); do
    if [ -e /home/$u/.tmux.init ]; then
        who+=($u)
    fi
    done

    # start or stop session per user...
    case "$1" in
        start)
            for u in ${who[@]}; do
                echo "Starting tmux with /home/$u/.tmux.init"
                su $u -c "$SCRIPT -c /home/$u/.tmux.init"
            done
            sleep 1
            $0 status
        ;;
        stop)
            echo "Shutting down tmux for $USER..."
            su $USER -c "$SCRIPT kill-server"
            $0 status
        ;;
        status)
            STATUS=$(ps aux | sed -n -e 1p -e "/tmux/I"p | grep -v $0 | grep -v sed)
            echo "$STATUS"
        ;;
    esac

Then for each user that wishes to launch a session define the following files. Note: These represent example files only that may be customized as desired. Also note lines may wrap in display and be sure lines terminate with Linux LF characters only and not MS Windows CR-LF sequences.

The _/home/$USER/.tmux.init_ file defines the tmux session and the individual windows within for the user. Customize per user preferences based on the following example. See tmux documentation for session, window, and pane details.

### ~/.tmux.init
    #!/bin/bash

    # preloaded session...
    /usr/bin/tmux new-session -d -s $USER

    # define a set of windows and launch specific commands
    # to make windows remain if initial command terminates append a "bash -i" command

    # Homebrew web server
    /usr/bin/tmux new-window -c /home/$USER/bin -n DIY '/usr/local/bin/node diy.js ../restricted/config; bash -i'
    #/usr/bin/tmux new-window -c /home/$USER/bin -n DIY '/usr/local/bin/forever node diy.js ../restricted/config; bash -i'

    # NodeRED server
    #/usr/bin/tmux new-window -c /home/$USER/bin -n node-red 'node-red; bash -i'

    # interactive node shell
    /usr/bin/tmux new-window -c /home/$USER/bin -n node++ '/usr/local/bin/node -r ./Extensions2JS; bash -i'

    # sqlite restricted bash shell
    /usr/bin/tmux new-window -c /home/$USER/restricted -n restricted

    # general purpose bash shell
    /usr/bin/tmux new-window -c /home/$USER/bin -n root

    # general purpose bash shell
    /usr/bin/tmux new-window -c /usr/local/bin -n bash@/u/bin

    # general purpose bash shell
    /usr/bin/tmux new-window -c /home/$USER -n bash@home

    # interactive python window
    #/usr/bin/tmux new-window -c /home/$USER/bin -n python@bin '/usr/bin/python; bash -i'

NOTE: Later versions will require executable permissions for _.tmux.init_.

The _/home/$USER/.tmux.conf_ file defines tmux behavior per each user's personal preferences. Customize as desire based on the following. See tmux documentation for configuration details.

### ~/.tmux.conf
    # tmux configuration 20140522...

    # change the prefix hot-key to more useable CTRL-A
    set -g prefix C-a
    unbind C-b
    bind C-a send-prefix

    # binding for the config file
    bind r source-file ~/.tmux.conf

    # window look and feel...
    set -g set-titles on
    set -g status-fg black
    set -g status-bg white
    set -g status-interval 5

    # enable mouse to allow window scrolling...
    set -g mouse on

    # toggle mouse-mode with prefix m or prefix M...
    bind m \
    set -g mode-mouse on \;\
    set -g mouse-resize-pane on \;\
    set -g mouse-select-pane on \;\
    set -g mouse-select-pane on \;\
    set -g mouse-select-window on \;\
    display 'Mouse: ON'
    bind M \
    set -g mode-mouse off \;\
    set -g mouse-resize-pane off \;\
    set -g mouse-select-pane off \;\
    set -g mouse-select-pane off \;\
    set -g mouse-select-window off \;\
    display 'Mouse: OFF'

Define the following _gotmux_ (i.e. go tmux or got mux) convenience script to open a user session after login

### /usr/local/bin/gotmux
    #!/bin/bash
    # start user specific tmux session

    who=${1-$USER}
    echo "WHO: '$who'"

    if [ "$who" == "$USER" ]; then
        tmux attach-session -t $who
    else
        su $who -c "tmux attach-session -t $who"
    fi


## Operation
Once all scripts have been defined reboot or run

    sudo systemctl start tmux

After connecting to an SSH session run simply run 

    gotmux

to connect to your session. 

Use number keys assigned to each window to jump around within a session. To disconnect and return to the login shell use _hot-key-prefix D_ keystroke. Make sure the gotmux script has executable permissions.
