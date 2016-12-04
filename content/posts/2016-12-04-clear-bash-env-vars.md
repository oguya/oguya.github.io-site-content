---
date: '2016-12-04'
tags:
- bash
- shell
- linux
title: "Temporarily Clearing Bash Environment Variables"
---


At times I'd just like to run a script or a program without using my current or pre-existing shell environment variables. Running `bash -l` doesn't really help because it just starts a new bash shell login with the same environment variables from your `~/.bashrc` or `profile.d` system-wide settings. Well, the other day I discovered prefixing your program/script with `env` temporarily clears your environment variables so that your script just runs with totally no variables!
<!--more-->


## Displaying Current Environment Variables

From the terminal, `env` or `printenv` will show you your current environment variables. For example:

```
$ env
XDG_VTNR=2
CUDA_INC_PATH=/usr/include/cuda
XDG_SESSION_ID=4
HOSTNAME=vast
PYENV_ROOT=/home/james/.pyenv
ANDROID_HOME=/home/james/src/android-sdk-linux
XDG_MENU_PREFIX=gnome-
SHELL=/bin/bash
```

## Temporarily Clearing Bash Environment Variables

Now, to run a program/script without any environment variable(s), prefix your command with `env -i`.

- The syntax is as follows:

        $ env -i /path/to/your/program arg1 arg2 ...

- For example:

        $ env -i ls -l

The `-i` option tells `env` command to completely ignore the environment

To set an environment variable, just initialize the variable with a value immediately after the `env` command.

- The syntax is as follows:

        env var=value /path/to/your/program arg1 arg2 ...

- For example:

        $ env CUDA_INC_PATH=/tmp/src/cuda /usr/bin/nvidia-smi
