# algo_arch

## Introduction and disclaimers
This is a sort of sketchy guide on how I got Arch Linux working as a client for Algo VPN. This guide does not aim to be comprehensive, both because I have not thoroughly tested the process with a variety of configurations and because while it will get the client working, it still is not fully automated. It also may be incomplete in places as I figured it out while messing around and eating my lunch between classes. I intend to repeat the process so I can thoroughly document all issues I ran into and exactly what I did to resolve them at a later date.

I would also add that I am merely a lowly undergraduate student in mathematics and not an infosec expert (and am new to the whole github thing), so if there is anything glaringly, obviously terrible about what I have done (particulalrly from a security perspective), please point it out.

## My setup

I both performed the deployment from Arch and set up the same Arch box as a client, a fully updated installation as of April 6, running the grsec kernel + PaX. I did not have to set any additional PaX exemptions during either the deployment for client configuration process, although your mileage may vary.

In terms of the Algo configuration for the deployment, I opted-in to:

1. On-demand for iOS/macOS clients on cellular
2. On-demand for iOS/macOS clients on Wifi
3. Additional security hardening 
4. Retaining CA certificate so I can run add-users

And opted-out of:

1. Win10 clients/using RSA
2. DNS ablocking
3. Per-user ssh

I also am using DigitalOcean as my provider in this case.

## Deploying from Arch

There were a few minor hiccups here, but otherwise seems to have worked mostly as standard. As usual, unzip `algo-master` into a convenient location. This is where a few minor differences involving Arch emerged. First, the default/system python in Arch is 3. Also, for those coming from other distributions, it is worth noting that Arch tries unless absolutely necessary to avoid splitting packages into `foo` and `foo-devel`. Thus, when installing dependencies, I made sure to specifically install the `python2` versions of packages:

```
libssl libffi python2-pip python2-setuptools python2-virtualenv
```

It is also very important to note that at this stage I installed StrongSwan manually. While in my modified Ansible playbooks for Arch I did include/retain the dependency checks for strongswan, in Arch strongswan is an AUR package and not a binary in the main repos, so I suggest that at this stage you install strongswan from the AUR using whatever your preferred method for installing AUR packages is.

Again, following the standard deployment procedure, we are going to be working in a virtualenv, but we must again ensure that we're using python2. Make sure that in our Ansible inventory `ansible_python_interpreter=python2`, but just to be safe I also explicitly specified using python2 in setting up the environment: 

```
python2 -m virtualenv env && source env/bin/activate && python2 -m pip install -r requirements.txt
```

From here everything seemed to work as standard.

## Arch client configuration

This is where things got trickier. First, as I mentioned above, make sure StrongSwan is installed. Algo already includes Ansible playbooks for client configuration on Debian, Ubuntu, Fedora, and CentOS/RHEL, so I based what I did off these. 

The main playbook for client configuration is `algo-master/deploy_client.yml`. Set your `vpn_user` to the desired user from your config, and `server_ip` to, well, the IP of your VPS. Since I was configuring as a client the same machine from which I did the deployment, locally, set name to `localhost` and simply comment out `ansible_ssh_user`.

The next step was to add an entry for Arch to the `get-os-info` part of the playbook:

```
- name Arch | Install prerequisites
  raw: >
    test -x /usr/bin/python2.7 ||
    sudo pacman -S python2
  changed_when: false
  when: "'ARCH' in distribution.stdout"
```
  
This is admittedly extremely hacky/poorly done, but this check was part of the process and I found the path of least resistance to just be to add such an entry. 
  
Going further in the "chain" of playbooks, `algo-master/roles/client/tasks/main.yml` did not seem to require any further modification, in part I suspect because we made sure to manually install StrongSwan. However, I'm sure this could be modified in a way to properly automate the process.

Next, `algo-master/roles/client/tasks/systems/main.yml` seems to pick which distro-specific playbook to reference depending on the ansible distribution, so I simply added an entry for Arch:

```
- include: Arch.yml
  when: ansible_distribution == 'Archlinux'
```
  
Then finally it was a matter of creating the `Arch.yml` itself, alongside `Ubuntu.yml` etc.:

```---

- set_fact:
    prerequisites: []
    configs_prefix: /etc
```
I followed the lead of the Debian playbook here. Again, we are assuming dependencies are already installed, and as for the various ipsec and strongswan configs, Arch seems to just store them in `/etc` by default (so e.g. `/etc/ipsec.conf`, `/etc/ipsec.secrets`, etc.)

Now this is the part I'm least confident in/seems most likely I don't understand what I'm doing, but it seemed to me that the existing client configuration playbooks expected to work with client ipsec config files of the form `ipsec.*.conf` rather than `ipsec_*.conf`

So in the desired config directory for the server, I cp'd my `ipsec_user.conf` and `ipsec_user.secrets` to `ipsec.user.conf` andf `ipsec.user.secrets`. The client deploy playbook seemed to work correctly after this, placing the aforementioned files in `/etc`, modifying the main `ipsec.conf` and related files to reference them appropriately, etc. No modifications seemed to be necessary in terms of the correct placements of the certificates etc.

As an aside, this may stem from my lack of understanding of ipsec, but it seems that `ipsec.conf` can work either by directly adding a connection to the file itself or by having it include files of the form `ipsec.*.conf` which contain the connection configs, which seems to be how it works in the Linux client deployment playbooks.

For good measure, at this point I reset ipset and made it reread configs (so `rereadsecrets` etc.) but I was able to establish a connection and confirm it was working correctly after this.

As should be clear, this is still hardly all automated. All my method did was configure my Arch machine as a client, but it does not e.g. auto-configure connecting automatically or install strongswan. As of now, this method still requires that you deal with your actual ipsec connections in your preferred way, e.g. by manually invoking `ipsec up`.
 



