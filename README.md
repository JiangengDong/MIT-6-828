# MIT 6.828

This repository is my implementation for [MIT 6.828](https://pdos.csail.mit.edu/6.828/2018/schedule.html)'s homework. 

## Setup the environment 

I use the [ms-vscode-remote.remote-containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension of VSCode so that I can work inside a container where all the dependencies are configured without corrupting my host OS. My [Dockerfile](./.devcontainer/Dockerfile) and [configuration file](./.devcontainer/devcontainer.json) are provided. 

I mount the X11 Unix socket of the host onto the container so that I can make use of the host's X server. This means that running `make qemu` under the [jos](./jos) folder can create a window as if it is running on the host machine. However, I still prefer the `make qemu-nox` version. 

To anyone that want to use my docker configuration: If the GUI cannot work, first check if your host has an X server. If so, you may need to disable the access control of the X server on the host with `xhost +`. You can re-enable the control with `xhost -` later.

## Contents

1. [Lab1](./lab1/README.md)