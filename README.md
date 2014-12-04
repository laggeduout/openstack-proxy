openstack-proxy
===============

There are two main 'programs' here: sstp-proxy, and ssh-jump.
The first is used to proxy incoming SSTP (Microsoft VPN) **OR**
wildcarded web (*.vpn.sandvine.rocks) to an instance, thus obviating
the need for a public IP.

The second (ssh-jump) is used to ssh directly to an instance
from outside the system, again without a public IP.

ssh-jump
--------
This is a program sort of like netcat, but it finds the namespace
of a given host (router) inside neutron, and then copies data
back and forth from there.

You might consider as a start putting (if your cloud host was called 'cloud.domain.name'):

    Host *+cloud
     ProxyCommand ssh -i ~/.ssh/id_nubo_jump_rsa -q jump@cloud.domain.name -- -tenant TENANT -host $(echo %h | sed -e 's?+cloud??g')
and then 

    ssh root@HOST+cloud (where HOST is the name you see as an instance in Nova)

in ~/.ssh/config. Or, something like:

    ssh -o "ProxyCommand ssh -q jump@cloud.domain.name -- -tenant YOU -host YOURHOST" user@cloud

on the command line.

It needs to have a python interperter with cap_sys_admin.
So i did this:
cd ~jump
cp /usr/bin/python2.7 .
setcap cap_sys_admin+ep ./python2.7
and then wrote a wrapper shell script

I also did a 'chsh' to /home/jump/nsnc
and ssh-keygen -t rsa; cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys

This means that now unprivileged users, outside your 'cloud', can ssh directly to
their instances without needing floating-ip or public IP.

sstp-proxy
----------

The syntax is https://<TENANT>.<INSTANCE>.vpn.sandvine.rocks,
or sstp://vpn.sandvine.rocks:9999/tenant/instance 
or sstp://tenant.instance.vpn.sandvine.rocks:9999
(YMMV as to which works best, the first is for browsers, the
second for Windows, the 3rd for Linux/Mac SSTP).

As a pre-req, you need python-prctl installed.

I use this with a VPN installed on Ubuntu 14.04 (softether), using
the following Heat Template subset. Login as cloud@VPN (password cloud).
Here is an example snippet for Heat.

    vpn:
      type: OS::Nova::Server
      properties:
        name: { str_replace: { params: { $stack_name: { get_param: 'OS::stack_name' } }, template: '$stack_name-vpn' } }
        key_name: { get_resource: key }
        image: "trusty"
        flavor: "m1.tiny"
        config_drive: "true"
        networks:
          - network: { get_resource: ctrl_net }
          - network: { get_resource: data_sub_net1 }
        user_data_format: RAW
        user_data: |
          #!/bin/bash
          iptables -F
          sed -i -e '/eth1/d' /etc/network/interfaces
          cat <<EOF >>/etc/network/interfaces
          auto eth1
          iface eth1 inet manual
            up ip link set eth1 up promisc on
            down ip link set eth1 down promisc off
          EOF
          ifup eth1

          cd /var/lib/softether
          stop softether
          rm -f vpn_server_config
          start softether
          cat <<EOF1 > vpn.cmd
          HubCreate vpn /PASSWORD:""
          hub vpn
          SecureNatDisable
          ServerCertRegenerate vk
          SstpEnable yes
          BridgeCreate vpn /DEVICE:eth1 /TAP:no
          UserCreate cloud /GROUP:none /REALNAME:none /NOTE:none
          UserPasswordSet cloud /PASSWORD:cloud
          EOF1
          vpncmd localhost /server /IN:vpn.cmd

