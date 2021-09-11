---
title: 'Preparing a LXC environment via Ansible'
description: 'A short walk-through that demonstrates how to prepare a LXC environment via Ansible.'
date: "2021-09-11"
tags: ["Ansible", "IaC"]
ShowToc: true
cover:
  image: 'erik-van-dijk-uAWRPtZ6n0s-unsplash.jpg'
  alt: 'Preparing LXC environments via Ansible'
---

## Introduction
[Ansible](https://www.ansible.com/) is my go-to IaC tool. Why? Because unlike other tools it just requires SSH or WinRM access to a machine. I prefer this 
method for simplicity rather than creating a master/slave relationship. 

[LXD](https://linuxcontainers.org/lxd/introduction/) is also my go-to container and virtual machine manager. Sure, we can debate all day the pros and cons 
and compare to "Docker". Although, I must state for local development, I will use Docker and Docker Compose, but for production, I prefer LXD, which runs 
an OS as a container and can utilize the benefits of a VM.

In short, technology is subjective, you choose what you like, and I like these tools. Of course, another tool to consider, for development, would be [Vagrant](https://www.vagrantup.com/).
I love Hashicorp! ... I had to mention it. lol.

## Code
Assuming you have Python, Ansible and LXD installed, to quickly create a playbook, I always opt to use [Ansible Generator](https://github.com/kkirsche/ansible-generator). Simply install via
`pip install -U ansible-generator` and run `ansible-generate -p playbook_name`. Add a role as well `ansible-generate -r role1`. 
In both cases name your project and role to what you see fit.

Let's create a hosts file. In a future article I will expand on this; however here we just put a name as these "hosts" will all be local.

```yaml
# inventory/hosts
proxy
```

I rename `playbook.yaml` to `site.yaml` and fill with the following code:

```yaml
# site.yaml
---
- hosts: localhost
  # run this task in the host
  tags: servers
  connection: local
  tasks:
    - name: create containers
      loop: "{{ groups['all'] }}"
      community.general.lxd_container:
        name: "{{ item }}"
        state: started
        source:
          type: image
          mode: pull
          server: https://images.linuxcontainers.org
          alias: ubuntu/focal/amd64
        profiles: ["default"]
        wait_for_ipv4_addresses: true
        timeout: 600
        devices:
          # configure network interface
          eth0:
            type: nic
            nictype: bridged
            parent: lxdbr0
            # get ip address from inventory
            # ipv4.address: "{{ hostvars[item].ip_address }}"
        # uncomment if you installed lxd using snap
        url: unix:/var/snap/lxd/common/lxd/unix.socket

    - name: add port maps
      community.general.lxd_container:
        name: proxy
        devices:
          map_port_2095:
            type: proxy
            listen: tcp:0.0.0.0:2095
            connect: tcp:127.0.0.1:80
          map_port_2096:
            type: proxy
            listen: tcp:0.0.0.0:2096
            connect: tcp:127.0.0.1:443
          map_port_8080:
            type: proxy
            listen: tcp:0.0.0.0:8080
            connect: tcp:127.0.0.1:8080

# NOT NEEDED
# WILL EXPAND ON THIS IN FUTURE ARTICLE
# EXAMPLE FOR INTERACTING WITH THE CONTAINER
#- hosts: proxy
#  tags: proxy
#  roles:
#    - consul_client
#    - proxy
```

The idea in this `site.yaml` is to pull Ubuntu Focal from https://us.images.linuxcontainers.org/. You may also change this to another image if you want. I am also 
using the default profile I created when initializing LXD. As a side note, I also suggest using LVM and create your own thin pool. Managing environments with LVM is much easier
and super simple to expand drives. Some good articles on the subject can be found [HERE](https://www.pither.com/simon/blog/2018/09/28/lxd-lvm-thinpool-setup) and [HERE](https://askubuntu.com/questions/1222407/setup-lxd-storage-thin-pool-on-an-existing-lvm-volume-group-of-the-host).

Several pors have also been forwarded. This is useful if you want to utilize proxies like Traefik, Nginx, etc. In the example above I utilize Cloudflare's DNS proxying on ports
2095 and 2096. I also make port 8080 available for the Traefik API. Again, in a future article I will create a role for [Traefik](https://traefik.io/).

We need to install the community collection, and then we can run ansible playbook to create the LXD environment:

```bash
ansible-galaxy collection install community.general
ansible-playbook -i inventory/hosts site.yml --tags="servers"
```

Further information can be found at [Ansible](https://docs.ansible.com/ansible/latest/collections/community/general/lxd_container_module.html).

You should be able to see your environment, `lxc list`, and to enter it `lxc exec proxy -- sudo /bin/bash`.

## Final Words
Ansible is a super simple tool to use. It has a big community and many collections to help 
interact with many things. LXD/LXC is one of many containerization technologies available, 
but it is simple to use and offers a full-blown OS.
