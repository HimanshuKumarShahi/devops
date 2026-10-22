FROM ubuntu:26.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
    nano \
    git \
    iproute2 \
    iputils-ping \
    net-tools \
    htop \
    tree \
    openssh-server \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/run/sshd


RUN useradd -rm -d /home/devops -s /bin/bash -g root -G sudo devops

RUN mkdir -p /home/devops/.ssh

COPY lab_key.pub /home/devops/.ssh/authorized_keys

RUN chown -R devops:root /home/devops/.ssh \
    && chmod 700 /home/devops/.ssh \
    && chmod 600 /home/devops/.ssh/authorized_keys

RUN sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

WORKDIR /workspace

CMD ["/usr/sbin/sshd", "-D"]