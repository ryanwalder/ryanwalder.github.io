---
layout: post
title: Using Vagrant for SaltStack States Development
permalink: vagrant-saltstack-development
comments: true
---

I won't get too deep on the subject of why Vagrant is a great tool for development, Darren Beale has a great [writeup](http://24ways.org/2014/what-is-vagrant-and-why-should-i-care/) on the subject, while it's more development based it still holds true for developing your Salt states. The TL;DR is that [Vagrant](https://www.vagrantup.com) gives us the ability to spin up machines then tear them down as needed, this is great for writing config management states as you can plug on through getting a system configured, tear the machine down then reprovision it from scratch.

## Vagrant and lxc

I use a slightly modified version of Vagrant in that I use LXC containers for my development machines rather than VirtualBox machines, this gives me a faster startup time for my machines and they use a lot less system resources while still giving me a full system to work on.

## Installing Vagrant & Plugins
I use Ubuntu 14.04 as my day to day work machine so these commands targeted to it, if you use another distro or run a later version things may differ slightly, we need a newer version of Vagrant than is in the standard Ubuntu repo's so we'll be grabbing it directly from the [site](https://www.vagrantup.com/downloads.html):

    wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.2_x86_64.deb
    sudo dpkg -i vagrant_1.7.2_x86_64.deb

Now we have vagrant installed we need a few extra packages to get LXC working as our provider:

    sudo apt-get update
    sudo apt-get install sudo apt-get install lxc lxc-templates redir cgroup-lite git-core

Now to install a couple of Vagrant plugins to allow us to get going:

    vagrant plugin install vagrant-lxc
    vagrant plugin install salty-vagrant-grains
    vagrant plugin install vagrant-cachier

vagrant-cachier is not required but it can [speed up](http://fgrehm.viewdocs.io/vagrant-cachier/benchmarks) the time it takes to spin up boxes by caching commonly used packages, I use it on a per machine basis in this setup which beans caches will survive across destroying a named machine (say "saltmaster") which means destroying and spinning up machines gets a bit faster.

## Setting up Vagrant
We need to grab the repo with the Vagrantfile in, this contains everything we need to develop salt states in Vagrant including a Salt Master and the ability to define minions within a YAML file which makes defining new machines a breeze

    git clone https://github.com/ryanwalder/saltstack-lxc-vagrant
    cd saltstack-lxc-vagrant

For the purpose of this post we'll be using the example states/pillars provided in the example folder of the repo, this will setup 4 nginx boxes behind a hapoxy box.

We need to set a few environmental variables which point to our salt states and salt pillar directories, these folders will be mounted into the container at /srv/salt and /srv/pillar which are then in turn used by the Salt Master.

    export VLS_STATES="example/states"
    export VLS_PILLAR="example/pillar"

Now we just need to copy the example minions.yaml into place and we can get going:

    cp minions.yaml.example minions.yaml

First we will fire up the Salt Master, as we are using LXC as our provider we need to specify this:

    vagrant up saltmaster --provider=lxc

If like me you like minimising keystrokes we can set Vagrant to use LXC by default with:

    export VAGRANT_DEFAULT_PROVIDER=lxc

After a few minutes you'll have a fully working Salt Master ready to provision systems you can check this out by jumping onto the machine, one on you can check out the shares states & pillar folders.

    vagrant ssh saltmaster
    ls /srv/salt
    ls /srv/pillar

As you can see the folders in `example/states` and `example/pillar` are shared from the host which means we can edit files on our local machine from the comfort of your own environment and have them immediately available on the Salt Master ready to deploy.

## Firing up the minions

Check the status of out vagrant setup you can use `vagrant status` which will give something like the below:

    Current machine states:

    saltmaster                running (lxc)
    nginx-001                 not created (lxc)
    nginx-002                 not created (lxc)
    nginx-003                 not created (lxc)
    nginx-004                 not created (lxc)
    haproxy                   not created (lxc)

    This environment represents multiple VMs. The VMs are all listed
    above with their current state. For more information about a specific
    VM, run `vagrant status NAME`.

The saltmaster is already running which means rather than spinning up each box individually we can just bring them all up at once with `vagrant up` rather than `vagrant up nginx-001`, this only works once the saltmaster box is up otherwise you'll have a race condition between the minions and the master where if they come up before the master things can get out of step which need the minions restarting, which is no fun.

As you can see from the `minions.yaml` config file the nginx boxes will run a highstate on being provisioned but the haproxy boxes does not so you'll need to jump on the box and run a highstate manually.

    vagrant ssh haproxy
    sudo salt-call state.highstate

You should see Salt install and configure haproxy so now you can go to `10.0.3.15` where you should be able to refresh the page and see haproxy rotate through the first 3 nginx boxes we previously setup, you can also see that the 3 nodes are talking to haproxy by going to `10.0.3.15:1936` (username/password: someuser/password).

## Working on salt states

Now we need to update haproxy to use the 4th nginx node, while we could add the 4th node to the haproxy config manually but the better way would be to use [Salt Mine](http://docs.saltstack.com/en/latest/topics/mine/), as you can see in `example/pillar/nginx/init.sls` we are already exporting the eth1 IP address of the nginx nodes via the salt mine so we can use this in the haproxy config. I've included a jinja templated config which does this so you just need to edit the `example/states/haproxy/init.sls` state file in the git repo to include ".jinja" to the end of the file.managed source line:

``` yaml
haproxy:
  pkg:
    - latest
  file:
    - managed
    - name: /etc/haproxy/haproxy.cfg
    - source: salt://haproxy/files/haproxy.cfg.jinja
    - template: jinja
    - user: root
    - group: root
    - mode: 644
    - require:
      - pkg: haproxy
    - watch_in:
      - service: haproxy
  service:
    - running
    - enable: True
    - require:
      - file: enable-haproxy-service
```

This will now use the templated file for setting up the haproxy config so we just need to run a highstate on the haproxy box again:

    vagrant ssh haproxy
    sudo salt-call state.highstate

Once Salt has done it's thing you can now check it has been added to the haproxy config by checking out the web frontend at `10.0.3.15:1936` again and you should now see all 4 minions being load balanced, no editing files remotely, no having to pull anything onto the master just a really quick edit to your states and they are available to deploy to the minions immediately, this gives us that quick feedback loop we need when building our states with the ability to start from scratch whenever we need with only a few commands.

Now we have the states in their "final" setup it's good to get in the habit of destroying the minions and re-provisioning them from scratch to make sure the states work from on fresh systems, this doesn't affect the examples as much as full state development where you'll be running states multiple times before nailing things down which can lead to states which only work after multiple runs due to the ordering of the states.

## Notes

Vagrant keeps track of the containers it's using based on the config you're using so if you remove a box from the minions.yaml that is still running vagrant will not keep track of it any more so it's best practice to make liberal use of `vagrant status` as you go along to keep track of what's running and make sure to `vagrant destroy` boxes before taking them out of the `minions.yaml`. You can always check what containers you have running at any given time by using `sudo lxc-ls` and if you need to kill off errant containers you forgot to destroy before removing them from the YAMl file you can destroy them directly with lxc directly:

    sudo lxc-stop --name container-name
    sudo lxc-destroy --name container-name

And if you just want to nuke everything from orbit:

    for i in $(sudo lxc-ls); do sudo lxc-stop --name $i; sudo lxc-destroy --name $i; done
