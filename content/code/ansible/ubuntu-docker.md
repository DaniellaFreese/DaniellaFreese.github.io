---
title: "Ansible - How to Install Docker and Connect to the Docker Daemon Immediately"
date: 2022-09-25
draft: false

categories: ["ansible"]
tags: ["ansible", "docker", "ubuntu"]
toc: false
summary: "**Situation:** I need to install docker and then the user immediately be able to reach the docker daemon all in the same playbook."
author: "djfreese"
---

**Situation:** I need to install docker and then the user immediately be able to reach the docker daemon with ansible.

The install steps for docker are: install processes, start/enable process, add your user to the docker group for permission to access the docker daemon, then lastly, logout and log back in to so the group membership is re-evaluated.

When building a demo with ansible, I don't want to have a separate playbook just to install Docker, I wanted to be able to Install Docker and immediately the user be able to run docker commands.

I was struggling with finding an ansible/non-hacky way of doing it, and found that the meta module was really convenient. With the meta module I could reset the connection as a task and immediately proceed to run any docker commands as the nonroot user. Here's the sample task below.

```go
    -  name: Reset the Connection for "{{ ansible_user }}" to access Docker Daemon
       ansible.builtin.meta: reset_connection
```

The task itself is pretty simple. With the fully example from installing docker (in this case on ubuntu) followed by using the meta module to reset the connection is shown below.
 
``` go

    - name: Installing the Ubuntu Pre-requisites
      block: 
        - name: Update Apt Cache
          ansible.builtin.apt:
            update_cache: yes
            cache_valid_time: 3600

        - name: Install Pre-requisite Pkgs
          ansible.builtin.apt:
            pkg:
            - ca-certificates
            - curl
            - gnupg
            - lsb-release
            - openssl
            - python3-pip

        - name: Add signing key
          ansible.builtin.apt_key:
            url: "https://download.docker.com/linux/ubuntu/gpg"
            state: present

        - name: Add repository into sources list
          ansible.builtin.apt_repository:
            repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
            state: present
            filename: docker

        - name: Install Docker
          ansible.builtin.apt:
            pkg: 
            - docker-ce
            - docker-ce-cli
            - containerd.io
            state: latest

        - name: Start Docker Process
            ansible.builtin.service:
            name: docker
            state: started
            enabled: yes

        - name: Add "{{ ansible_user }}" to docker group
          ansible.builtin.user:
            name: "{{ ansible_user }}"
            groups: docker
            append: yes

        - name: Install Pip docker-py pkg
          ansible.builtin.pip:
            name: docker
        become: true

    -  name: Reset the Connection for "{{ ansible_user }}" to access Docker Daemon
       ansible.builtin.meta: reset_connection

```
