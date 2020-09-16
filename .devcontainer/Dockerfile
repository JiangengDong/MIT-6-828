
FROM ubuntu:16.04

SHELL ["/bin/bash", "-c"]
WORKDIR /usr/local/src

# some basic tools
RUN apt-get update && \
    apt-get install -y git build-essential gdb gcc-multilib tmux vim sudo && \
    apt-get autoremove -y && \
    apt-get clean -y && \
    rm -rf /var/lib/apt/lists/*

# install qemu
RUN apt-get update && \
    apt-get install -y libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev libtool zlib1g-dev libfdt-dev && \
    apt-get autoremove -y && \
    apt-get clean -y && \
    rm -rf /var/lib/apt/lists/*
RUN git clone --depth 1 https://github.com/mit-pdos/6.828-qemu.git qemu && \
    cd qemu && \
    ./configure --disable-kvm --target-list="i386-softmmu x86_64-softmmu" && \
    make && \
    make install 

# add a user "oslearner" with password "oslearner"
RUN useradd -m oslearner && \
    echo "oslearner:oslearner" | chpasswd && \
    adduser oslearner sudo
# change user and workdir
USER oslearner
WORKDIR /home/atlas
