# Demo overview:  
Appformix is used for network devices monitoring.  
Appformix send webhook notifications to SaltStack.   
The webhook notifications provides the device name and other details.  
SaltStack automatically collects junos show commands output on the "faulty" JUNOS device and archives the output on a Git server.    
![Appformix-SaltStack-Junos-Git.png](Appformix-SaltStack-Junos-Git.png)  

# Demo building blocks: 
- Junos devices
- Appformix
- Ubuntu with: 
    - SaltStack
    - Docker
    - Gitlab

# webhooks Overview: 
- A webhook is notification using an HTTP POST. A webhook is sent by a system A to push data (json body as example) to a system B when an event occurred in the system A. Then the system B will decide what to do with these details. 
- Appformix supports webhooks. A notification is generated when the condition of an alarm is observed. You can configure an alarm to post notifications to an external HTTP endpoint. AppFormix will post a JSON payload to the endpoint for each notification.
- SaltStack can listens to webhooks and generate equivalents ZMQ messages to the event bus  
- SaltStack can reacts to webhooks

# Building blocks role: 

## Appformix:  
- Collects data from Junos devices (JTI native telemetry and SNMP)  
- Generates webhooks notifications (HTTP POST with a JSON body) to SaltStack when the condition of an alarm is observed. The JSON body provides the device name and other details

## Junos devices: 
- Several Junos devices. 
- They are monitored by Appformix
- Based on webhook notifications from Appformix, SaltStack collects junos show commands on the "faulty" Junos and archives the collected data to a Gitlab repository  

## Ubuntu
- with Docker and SaltStack installed.  
- A Gitlab docker container is instanciated.  

## Gitlab  
- This SaltStack setup uses a gitlab server for external pillars (variables)
- This SaltStack setup uses the gitlab server as a remote file server 
- Junos show commands output is automatically saved on the Gitlab server

## SaltStack: 
- Salt master, minion, proxy (one proxy process per Junos device), webhook engine.   
- The Salt master listens to webhooks 
- The Salt master generates a ZMQ messages to the event bus when a webhook notification is received. The ZMQ message has a tag and data. The data structure is a dictionary, which contains information about the event.
- The Salt reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions.
- The sls reactor file used in this content does the following: it parses the data from the ZMQ message to extract the network device name. It then ask to the proxy that manages the "faulty" Junos device to execute an sls file.
- The sls file executed by the proxy minion collects junos show commands output and archives the collected data to a Gitlab repository

# Requirements: 
- Install appformix
- Configure appformix for network devices monitoring
- Install Docker 
- Instanciate a Gitlab docker container
- Configure Gitlab
- Install SaltStack

# Appformix  

## Install Appformix. 
This is not covered by this documentation.  

## Configure Appformix for network devices monitoring 

Appformix supports network devices monitoring using SNMP and JTI (Juniper Telemetry Interface) native streaming telemetry.  
- For SNMP, the polling interval is 60s.  
- For JTI streaming telemetry, Appformix automatically configures the network devices. The interval configured on network devices is 60s.  

Here's the [**documentation**](https://www.juniper.net/documentation/en_US/appformix/topics/concept/appformix-ansible-configure-network-device.html)  

In order to configure AppFormix for network devices monitoring, here are the steps:
- manage the 'network devices json configuration' file. This file is used to define the list of devices you want to monitor using Appformix, and the details you want to collect from them.    
- Indicate to the 'Appformix installation Ansible playbook' which 'network devices json configuration file' to use. This is done by setting the variable ```network_device_file_name``` in ```group_vars/all```
- Set the flag to enable appformix network device monitor. This is done by setting the variable ```appformix_network_device_monitoring_enabled``` to ```true``` in ```group_vars/all```
- Enable the Appformix plugins for network devices monitoring. This is done by setting the variable ```appformix_plugins``` in ```group_vars/all```
- re run the 'Appformix installation Ansible playbook'.

Here's how to manage the 'network devices json configuration file' with automation:  
Define the list of devices you want to monitor using Appformix, and the details you want to collect from them:    
```
vi configure_appformix/network_devices.yml
```

Execute the python script [**network_devices.py**](configure_appformix/network_devices.py). It renders the template [**network_devices.j2**](configure_appformix/network_devices.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**network_devices.json**](configure_appformix/network_devices.json).  
```
python configure_appformix/network_devices.py
```
```
more configure_appformix/network_devices.json
```

From your appformix directory, update ```group_vars/all``` file: 
```
cd appformix-2.15.2/
vi group_vars/all
```
to make sure it contains this:
```
network_device_file_name: /path_to/network_devices.json
appformix_network_device_monitoring_enabled: true
appformix_jti_network_device_monitoring_enabled: true
appformix_plugins:
   - plugin_info: 'certified_plugins/jti_network_device_usage.json'
   - plugin_info: 'certified_plugins/snmp_network_device_routing_engine.json'
   - plugin_info: 'certified_plugins/snmp_network_device_usage.json'
```

Then, from your appformix directory, re-run the 'Appformix installation Ansible playbook':
```
cd appformix-2.15.2/
ansible-playbook -i inventory appformix_standalone.yml
```
## Configure the network devices with the SNMP community used by Appformix

You need to configure the network devices with the SNMP community used by Appformix. The script [**snmp.py**](configure_junos/snmp.py) renders the template [**snmp.j2**](configure_junos/snmp.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**snmp.conf**](configure_junos/snmp.conf). This file is then loaded and committed on all network devices used with SNMP monitoring.
 
Requirement: This script uses the junos-eznc python library so you need first to install it.  

```
python configure_junos/snmp.py
configured device 172.30.52.85 with snmp community public
configured device 172.30.52.86 with snmp community public
```
```
more configure_junos/snmp.conf
```

## Configure the network devices for JTI telemetry

For JTI native streaming telemetry, Appformix uses NETCONF to automatically configure the network devices:  
```
lab@vmx-1-vcp> show system commit
0   2018-03-22 16:32:37 UTC by lab via netconf
1   2018-03-22 16:32:33 UTC by lab via netconf
```
```
lab@vmx-1-vcp> show configuration | compare rollback 1
[edit services analytics]
+    sensor Interface_Sensor {
+        server-name appformix-telemetry;
+        export-name appformix;
+        resource /junos/system/linecard/interface/;
+    }
```
```
lab@vmx-1-vcp> show configuration | compare rollback 2
[edit]
+  services {
+      analytics {
+          streaming-server appformix-telemetry {
+              remote-address 172.30.52.157;
+              remote-port 42596;
+          }
+          export-profile appformix {
+              local-address 192.168.1.1;
+              local-port 21112;
+              dscp 20;
+              reporting-rate 60;
+              format gpb;
+              transport udp;
+          }
+          sensor Interface_Sensor {
+              server-name appformix-telemetry;
+              export-name appformix;
+              resource /junos/system/linecard/interface/;
+          }
+      }
+  }

lab@vmx-1-vcp>
```
Run this command to show the installed sensors: 
```
lab@vmx-1-vcp> show agent sensors
```

If Appformix has serveral ip addresses, and you want to configure the network devices to use a different IP address than the one configured by appformix for telemetry server, execute the python script [**telemetry.py**](configure_junos/telemetry.py). 
The python script [**telemetry.py**](configure_junos/telemetry.py) renders the template [**telemetry.j2**](configure_junos/telemetry.j2) using the variables [**network_devices.yml**](configure_appformix/network_devices.yml). The rendered file is [**telemetry.conf**](configure_junos/telemetry.conf). This file is then loaded and committed on all network devices used with JTI telemetry.  

Requirement: This script uses the junos-eznc python library so you need first to install it.  

```
more configure_appformix/network_devices.yml
```
```
python configure_junos/telemetry.py
configured device 172.30.52.155 with telemetry server ip 192.168.1.100
configured device 172.30.52.156 with telemetry server ip 192.168.1.100
```
```
# more configure_junos/telemetry.conf
set services analytics streaming-server appformix-telemetry remote-address 192.168.1.100
```
Verify on your network devices: 
```
lab@vmx-1-vcp> show configuration services analytics streaming-server appformix-telemetry remote-address
remote-address 192.168.1.100;

lab@vmx-1-vcp> show configuration | compare rollback 1
[edit services analytics streaming-server appformix-telemetry]
-    remote-address 172.30.52.157;
+    remote-address 192.168.1.100;

lab@vmx-1-vcp> show system commit
0   2018-03-23 00:34:47 UTC by lab via netconf

```
```
lab@vmx-1-vcp> show agent sensors

Sensor Information :

    Name                                    : Interface_Sensor
    Resource                                : /junos/system/linecard/interface/
    Version                                 : 1.1
    Sensor-id                               : 150000323
    Subscription-ID                         : 562950103421635
    Parent-Sensor-Name                      : Not applicable
    Component(s)                            : PFE

    Server Information :

        Name                                : appformix-telemetry
        Scope-id                            : 0
        Remote-Address                      : 192.168.1.100
        Remote-port                         : 42596
        Transport-protocol                  : UDP

    Profile Information :

        Name                                : appformix
        Reporting-interval                  : 60
        Payload-size                        : 5000
        Address                             : 192.168.1.1
        Port                                : 21112
        Timestamp                           : 1
        Format                              : GPB
        DSCP                                : 20
        Forwarding-class                    : 255

```

# Docker 


Check if Docker is already installed
```
$ docker --version
```

If it was not already installed, install it:
```
$ sudo apt-get update
```
```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```
$ sudo apt-get update
```
```
$ sudo apt-get install docker-ce
```
```
$ sudo docker run hello-world
```
```
$ sudo groupadd docker
```
```
$ sudo usermod -aG docker $USER
```

Exit the ssh session and open an new ssh session and run these commands to verify you installed Docker properly:  
```
$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
```
$ docker --version
Docker version 18.03.1-ce, build 9ee9f40
```

# Gitlab

This SaltStack setup uses a gitlab server for external pillars and as a remote file server.  

## Instanciate a Gitlab docker container


There is a Gitlab docker image available https://hub.docker.com/r/gitlab/gitlab-ce/


Pull the image: 
```
# docker pull gitlab/gitlab-ce
```

Verify: 
```
# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
gitlab/gitlab-ce             latest              09b815498cc6        6 months ago        1.33GB
```

Instanciate a container: 
```
docker run -d --rm --name gitlab -p 3022:22 -p 9080:80 gitlab/gitlab-ce
```
Verify:
```
# docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                PORTS                                                 NAMES
9e8330425d9c        gitlab/gitlab-ce             "/assets/wrapper"        5 months ago        Up 5 days (healthy)   443/tcp, 0.0.0.0:3022->22/tcp, 0.0.0.0:9080->80/tcp   gitlab
```

Wait for Gitlab container status to be ```healthy```. It takes about 2 mns.   
```
$ watch -n 10 'docker ps'
```

Verify you can access to Gitlab GUI: 
Access Gitlab GUI with a browser on port 9080. Gitlab user is ```root```    


## Configure Gitlab

### Create repositories

Create the group ```organization```.    
Create the repositories ```network_parameters``` and ```network_model``` in the group ```organization```.      
The repository ```network_parameters``` is used for SaltStack external pillars.    
The repository ```network_model``` is used as an external files server for SaltStack   

### Add your public key to Gitlab

The Ubuntu host will inteact with the Gitlab server. 

#### Generate ssh keys
```
$ sudo -s
```
```
# ssh-keygen -f /root/.ssh/id_rsa -t rsa -N ''
```
```
# ls /root/.ssh/
id_rsa  id_rsa.pub  
```
#### Add the public key to Gitlab  
Copy the public key:
```
# more /root/.ssh/id_rsa.pub
```
Access Gitlab GUI, and add the public key to ```User Settings``` > ```SSH Keys```

### Update your ssh configuration
```
$ sudo -s
```
```
# touch /root/.ssh/config
```
```
# ls /root/.ssh/
config       id_rsa       id_rsa.pub  
```
```
# vi /root/.ssh/config
```
```
# more /root/.ssh/config
Host gitlab_ip_address
Port 3022
Host *
Port 22
```

### Configure your Git client

```
$ sudo -s
# git config --global user.email "you@example.com"
# git config --global user.name "Your Name"
```

### Verify you can use Git and Gitlab

Git clone the repositories ```network_parameters``` and ```network_model```

# SaltStack 

## Install SaltStack 

- Install master
- Install minion 
- Install requirements for SaltStack Junos proxy

### Install master

Check if SaltStack master is already installed
```
$ sudo -s
```
```
# salt --version
```
```
# salt-master --version
```
if SaltStack master was not already installed, then install it: 
```
$ sudo -s
```
```
# wget -O - https://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2/SALTSTACK-GPG-KEY.pub | sudo apt-key add -
```
Add ```deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main``` in the file ```/etc/apt/sources.list.d/saltstack.list```
```
# touch /etc/apt/sources.list.d/saltstack.list
```
```
# nano /etc/apt/sources.list.d/saltstack.list
```
```
# more /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main
```
```
# sudo apt-get update
```
```
# sudo apt-get install salt-master
```
Verify you installed properly SaltStack master 
```
# salt --version
salt 2018.3.2 (Oxygen)
```
```
# salt-master --version
salt-master 2018.3.2 (Oxygen)
```

### Install Minion

Check if SaltStack minion is already installed
```
# salt-minion --version
```
if SaltStack minion was not already installed, then install it: 
```
$ sudo -s
```
```
# wget -O - https://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2/SALTSTACK-GPG-KEY.pub | sudo apt-key add -
```
Add ```deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main``` in the file ```/etc/apt/sources.list.d/saltstack.list```
```
# touch /etc/apt/sources.list.d/saltstack.list
```
```
# nano /etc/apt/sources.list.d/saltstack.list
```
```
# more /etc/apt/sources.list.d/saltstack.list
deb http://repo.saltstack.com/apt/ubuntu/16.04/amd64/archive/2018.3.2 xenial main
```
```
# sudo apt-get update
```
```
$ sudo apt-get install salt-minion
```
And verify if salt-minion was installed properly installation 
```
# salt-minion --version
salt-minion 2018.3.2 (Oxygen)
```

### Install requirements for SaltStack Junos proxy 

The Salt Junos proxy has some requirements (```junos-eznc``` python library and other dependencies). 

```
# apt-get install python-pip
# pip list
# apt-get --auto-remove --yes remove python-openssl
# pip install pyOpenSSL junos-eznc jxmlease jsnapy
# pip list | grep "pyOpenSSL\|junos-eznc\|jxmlease\|jsnapy"
```

## Configure SaltStack

- Configure SaltStack master
- Configure SaltStack minion 
- Configure SaltStack pillars
- Configure SaltStack proxy 
- Configure SaltStack files server
- Configure SaltStack webhook engine
- Configure SaltStack reactor
- Configure Junos

### Configure SaltStack master

#### SaltStack master configuration file

ssh to the Salt master and copy this [SaltStack master configuration file](master) in the file ```/etc/salt/master```  

So:
- the Salt master is listening webhooks on port 5001. It generates equivalents ZMQ messages to the event bus
- external pillars are in the gitlab repository ```organization/network_parameters```  (master branch)
- Salt uses the gitlab repository ```organization/network_model``` as a remote files server.  

#### Restart the salt-master service

```
# service salt-master restart
```
#### Verify the salt-master status

To see the Salt processes: 
```
# ps -ef | grep salt
```
To check the status, you can run these commands: 
```
# systemctl status salt-master.service
```
```
# service salt-master status
```
#### SaltStack master log

```
# more /var/log/salt/master 
```
```
# tail -f /var/log/salt/master
```

### Configure SaltStack minion 


#### SaltStack minion configuration file

Copy the [minion configuration file](minion) in the file ```/etc/salt/minion```


#### Restart the salt-minion service


```
# service salt-minion restart
```

#### Verify the salt-minion status

To see the Salt processes: 
```
# ps -ef | grep salt
```
To check the status: 
```
# systemctl status salt-minion.service
```
```
# service salt-minion status
```

#### Verify the keys 

You need to accept the minions/proxies public keys on the master.   


To list all public keys:
```
# salt-key -L
```
To accept a specified public key:
```
# salt-key -a minion1 -y
```
Or, to accept all pending keys:
```
# salt-key -A -y
```

#### Verify master <-> minion communication 

Run this command to make sure the minion is up and responding to the master. This is not an ICMP ping. 
```
# salt minion1 test.ping
```
Run this additionnal test  
```
# salt "minion1" cmd.run "pwd"
```


### Configure SaltStack pillars

Pillars are variables     
They are defined in sls files, with a yaml data structure.  
There is a ```top``` file.  
The ```top.sls``` file map minions to sls (pillars) files.  

#### Pillar configuration

Refer to the [master configuration file](master) to know the location for pillars.  
It is the repository ```network_parameters```  
Add at the root of the repository ```network_parameters``` your pillars: 
- [top.sls](top.sls) 
- [data_collection.sls](data_collection.sls)
- [core-rtr-p-01-details.sls](core-rtr-p-01-details.sls) 
- [core-rtr-p-02-details.sls](core-rtr-p-02-details.sls) 


#### Pillars configuration verification

```
$ sudo -s
```
```
# salt-run pillar.show_pillar
```
```
# salt-run pillar.show_pillar core-rtr-p-02
```

### Configure SaltStack proxy 

#### SaltStack proxy configuration file

Copy the [proxy configuration file](proxy) in the file ```/etc/salt/proxy```  

#### Start SaltStack proxy 

You need one salt proxy process per Junos device.  

to start the proxy as a daemon for the device ```core-rtr-p-01```, run this command
```
# sudo salt-proxy -d --proxyid=core-rtr-p-01
```
The proxy daemon ```core-rtr-p-01``` manages the network device ```core-rtr-p-01```.  

to start the proxy as a daemon for the device ```core-rtr-p-02```, run this command
```
# sudo salt-proxy -d --proxyid=core-rtr-p-02
```
The proxy daemon ```core-rtr-p-02``` manages the network device ```core-rtr-p-02```.  


you can run this command to start it with a debug log level: 
```
# sudo salt-proxy -l debug --proxyid=core-rtr-p-02
```

To see the SaltStack processes, run this command: 
```
# ps -ef | grep salt
```

#### Verify the keys

You need to accept the minions/proxies public keys on the master.   


To list all public keys:
```
# salt-key -L
```
To accept a specified public key:
```
# salt-key -a core-rtr-p-01 -y
# salt-key -a core-rtr-p-02 -y

```
Or, to accept all pending keys:
```
# salt-key -A -y
```
#### Verify master <-> proxy communication

Run this command to make sure the proxies are up and responding to the master. This is not an ICMP ping. 
```
# salt 'core-rtr-p-0*' test.ping
```
Run this additionnal test. It is an execution module. The master asks to the proxies ```core-rtr-p-01``` and ```core-rtr-p-02``` to use an execution module
```
# salt 'core-rtr-p-0*' junos.cli "show version"
```


### Configure SaltStack files server

Salt runs a files server to deliver files to minions and proxies.  
The [master configuration file](master) indicates the location for the files server.  
We are using an external files server (gitlab repository ```organization/network_model```) 
Add this content in the gitlab repository ```organization/network_model```: 
- collect_data_and_archive_to_git.sls
- collect_show_commands_example_1.sls
- collect_show_commands_example_2.sls
- All files from [the directory trigger_alarms](trigger_alarms)



#### Test your automation content manually from the master

Example with the proxy ```core-rtr-p-01``` (it manages the network device ```core-rtr-p-01```).   
Run this command on the master to ask to the proxy ```core-rtr-p-01``` to execute the state file [collect_data_and_archive_to_git](collect_data_and_archive_to_git).  
```
salt core-rtr-p-01 state.apply collect_data_and_archive_to_git
```
The data collected by the proxy ```core-rtr-p-01``` is archived in the directory [core-rtr-p-01](core-rtr-p-01)  



### Configure SaltStack webhook engine

Engines are executed in a separate process that is monitored by Salt. If a Salt engine stops, it is restarted automatically.  
Engines can run on both master and minion.  To start an engine, you need to specify engine information in master/minion config file depending on where you want to run the engine. Once the engine configuration is added, start the master and minion normally. The engine should start along with the salt master/minion.   
webhook engine listens to webhook, and generates and pusblishes messages on SaltStack 0MQ bus.  

We already added the webhook engine configuration in the [master configuration file](master)  
So Appformix should his webhook notifications to the master ip address on port 5001. 

```
# more /etc/salt/master
```

### Configure SaltStack reactor

#### Configure reactor configuration file
The reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions.  

To map some events to reactor sls files, copy the [reactor configuration file](reactor.conf) to ```/etc/salt/master.d/reactor.conf```  

This reactor binds webhook from Appformix to ```/srv/reactor/automate_show_commands.sls```  

#### Restart the salt master service
```
# service salt-master restart
```
#### Verify the reactor operationnal state: 
```
# salt-run reactor.list
```
#### Add your reactor sls files
create a ```/srv/reactor/``` directory    
```
# mkdir /srv/reactor/
```
and copy the sls reactor file [automate_show_commands.sls](automate_show_commands.sls) to the directory ```/srv/reactor/```

The reactor [automate_show_commands.sls](automate_show_commands.sls) parses the data from the ZMQ message that has the tags ```salt/engines/hook/appformix_to_saltstack``` to extract the device name, and asks to the proxy that manages the faulty device to apply the state file [collect_data_and_archive_to_git](collect_data_and_archive_to_git)

The state file [collect_data_and_archive_to_git](collect_data_and_archive_to_git) executed by the proxy is located in the ```network_model``` repository.  
It collects show commands and archives the data collected to a git server.  

The list of junos commands to collect is maintained with the variable ```data_collection```  
the variable ```data_collection``` is defined in the pillar [data_collection.sls](data_collection.sls)  
Pillars are in the repository ```network_parameters```  














### SaltStack Git execution module basic demo

ssh to the Salt master.

On the Salt master, list all the keys.
```
salt-key -L
```
These commands are run from the master.   
Most of these commands are using the Git execution module.   
So the master is asking to the minion ```core-rtr-p-01``` to execute these commands.    
```
# salt core-rtr-p-01 git.clone /tmp/local_copy git@github.com:ksator/automated_junos_show_commands_collection_with_appformix_saltstack.git identity="/root/.ssh/id_rsa"
core-rtr-p-01:
    True

# salt core-rtr-p-01 cmd.run "ls /tmp/local_copy"
core-rtr-p-01:
    README.md
    ...

# salt core-rtr-p-01 git.config_set user.email me@example.com cwd=/tmp/local_copy
core-rtr-p-01:
    - me@example.com

# salt core-rtr-p-01 git.config_set user.name ksator cwd=/tmp/local_copy
core-rtr-p-01:
    - ksator
    
# salt core-rtr-p-01 git.config_get user.name cwd=/tmp/local_copy
core-rtr-p-01:
    ksator

# salt core-rtr-p-01 git.pull /tmp/local_copy
core-rtr-p-01:
    Already up-to-date.

# salt core-rtr-p-01 file.touch "/tmp/local_copy/test.txt"
core-rtr-p-01:
    True

# salt core-rtr-p-01 file.write "/tmp/local_copy/test.txt" "hello from SaltStack using git executiom module"
core-rtr-p-01:
    Wrote 1 lines to "/tmp/local_copy/test.txt"

# salt core-rtr-p-01 cmd.run "more /tmp/local_copy/test.txt"
core-rtr-p-01:
    ::::::::::::::
    /tmp/local_copy/test.txt
    ::::::::::::::
    hello from SaltStack using git executiom module

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------
    untracked:
        - test.txt

# salt core-rtr-p-01 git.add /tmp/local_copy /tmp/local_copy/test.txt
core-rtr-p-01:
    add 'test.txt'

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------
    new:
        - test.txt

# salt core-rtr-p-01 git.commit /tmp/local_copy 'The commit message'
core-rtr-p-01:
    [master 60f5943] The commit message
     1 file changed, 1 insertion(+)
     create mode 100644 test.txt

# salt core-rtr-p-01 git.status /tmp/local_copy
core-rtr-p-01:
    ----------

# salt core-rtr-p-01 git.push /tmp/local_copy origin master identity="/root/.ssh/id_rsa"
core-rtr-p-01:
```
The above commands pushed the file [test.txt](test.txt) to this repository  

### Test your Junos proxy daemons

ssh to the Salt master.

On the Salt master, list all the keys. 
```
# salt-key -L
```
Run this command to check if the minions are up and responding to the master. This is not an ICMP ping.
```
# salt -G 'roles:minion' test.ping
```
```
# salt core-rtr-p-01 test.ping
core-rtr-p-01:
    True
```
List the grains: 
```
# salt core-rtr-p-01 grains.ls
...
```
Get the value of the grain nodename. 
```
# salt core-rtr-p-01 grains.item nodename
core-rtr-p-01:
    ----------
    nodename:
        svl-util-01
```
So, the junos proxy daemon ```core-rtr-p-01``` is running on the minion ```svl-util-01```  

The Salt Junos proxy has some requirements (junos-eznc python library and other dependencies).
```
# salt svl-util-01 cmd.run "pip list | grep junos"
svl-util-01:
    junos-eznc (2.1.7)
```
### Junos execution modules

Run this command on the master to ask to a proxy to use a Junos execution module:   
```
# salt core-rtr-p-01 junos.cli "show chassis hardware"
core-rtr-p-01:
    ----------
    message:

        Hardware inventory:
        Item             Version  Part number  Serial number     Description
        Chassis                                VM5AA80D5BB2      VMX
        Midplane
        Routing Engine 0                                         RE-VMX
        CB 0                                                     VMX SCB
        FPC 0                                                    Virtual FPC
          CPU            Rev. 1.0 RIOT-LITE    BUILTIN
          MIC 0                                                  Virtual
            PIC 0                 BUILTIN      BUILTIN           Virtual
    out:
        True
```
### Junos state modules 

The files [collect_show_commands_example_1.sls](collect_show_commands_example_1.sls) and [collect_show_commands_example_2.sls](collect_show_commands_example_2.sls) use a diff syntax but they are equivalents.  

#### Syntax 1

```
# more /srv/salt/collect_junos_show_commands_example_1.sls
show version:
  junos:
    - cli
    - dest: /tmp/show_version.txt
    - format: text
show chassis hardware:
  junos:
    - cli
    - dest: /tmp/show_chassis_hardware.txt
    - format: text
```
Run this command on the master to ask to the proxy core-rtr-p-01 to execute the sls file  [collect_show_commands_example_1.sls](collect_show_commands_example_1.sls).
```
# salt core-rtr-p-01 state.apply collect_show_commands_example_1
```

#### Syntax 2
```
# more /srv/salt/collect_show_commands_example_2.sls
show_version:
  junos.cli:
    - name: show version
    - dest: /tmp/show_version.txt
    - format: text
show_chassis_hardware:
  junos.cli:
    - name: show chassis hardware
    - dest: /tmp/show_chassis_hardware.txt
    - format: text
```
Run this command on the master to ask to the proxy core-rtr-p-01 to execute the sls file  [collect_show_commands_example_2.sls](collect_show_commands_example_2.sls).
```
# salt core-rtr-p-01 state.apply collect_show_commands_example_2
```
### sls file to collect junos show commands and to archive the output to git

This sls file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls) collectes data from junos devices (show commands) and archive the data collected on a git server  

Add this file in the ```junos``` directory of the ```organization/network_model``` repository (```gitfs_remotes```) .  

### Pillars 

Here's an example for the ```top.sls``` file at the root of the gitlab repository ```organization/network_parameters``` (```ext_pillar```)  
```
{% set id = salt['grains.get']('id') %} 
{% set host = salt['grains.get']('host') %} 

base:
  '*':
    - production
   
{% if host == '' %}
  '{{ id }}':
    - {{ id }}
{% endif %}
```

The pillar ```data_collection``` is used by the file [collect_data_and_archive_to_git.sls](collect_data_and_archive_to_git.sls)  
Update the file ```production.sls``` in the repository ```organization/network_parameters``` (```ext_pillar```) to define the pillar ```data_collection``` 
```
data_collection:  
   - command: show interfaces  
   - command: show chassis hardware
   - command: show version   
   - command: show configuration
```

### Test your automation content manually from the master

Example with the proxy ```core-rtr-p-02``` (it manages the network device ```core-rtr-p-02```).   
Run this command on the master to ask to the proxy ```core-rtr-p-01``` to execute it.  
```
salt core-rtr-p-01 state.apply collect_data_and_archive_to_git
```

The data collected by the proxy ```core-rtr-p-01``` is archived in the directory [core-rtr-p-01](core-rtr-p-01)  


###  Update the Salt reactor

Update the Salt reactor file  
The reactor binds sls files to event tags. The reactor has a list of event tags to be matched, and each event tag has a list of reactor SLS files to be run. So these sls files define the SaltStack reactions.  
Update the reactor.  
This reactor binds ```salt/engines/hook/appformix_to_saltstack``` to ```/srv/reactor/automate_show_commands.sls``` 
```
# more /etc/salt/master.d/reactor.conf
reactor:
   - 'salt/engines/hook/appformix_to_saltstack':
       - /srv/reactor/automate_show_commands.sls
```

Restart the Salt master:
```
service salt-master stop
service salt-master start
```

The command ```salt-run reactor.list``` lists currently configured reactors:  
```
salt-run reactor.list
```

Create the sls reactor file ```/srv/reactor/automate_show_commands.sls```.  
It parses the data from the ZMQ message that has the tags ```salt/engines/hook/appformix_to_saltstack``` and extracts the network device name.  
It then ask to the Junos proxy minion that manages the "faulty" device to apply the ```junos/collect_data_and_archive_to_git.sls``` file.  
the ```junos/collect_data_and_archive_to_git.sls``` file executed by the Junos proxy minion collects show commands from the "faulty" device and archive the data collected to a git server. 

```
# more /srv/reactor/automate_show_commands.sls
{% set body_json = data['body']|load_json %}
{% set devicename = body_json['status']['entityId'] %}
automate_show_commands:
  local.state.apply:
    - tgt: "{{ devicename }}"
    - arg:
      - collect_data_and_archive_to_git
```

# Run the demo: 

## Create Appformix webhook notifications.  

You can do it from Appformix GUI, settings, Notification Settings, Notification Services, add service.    
Then:  
service name: appformix_to_saltstack  
URL endpoint: provide the Salt master IP and Salt webhook listerner port (```HTTP://192.168.128.174:5001/appformix_to_saltstack``` as example).  
setup  

## Create Appformix alarms, and map these alarms to the webhook you just created.

You can do it from the Appformix GUI, Alarms, add rule.  
Then, as example:   
Name: in_unicast_packets_core-rtr-p-02,  
Module: Alarms,  
Alarm rule type: Static,  
scope: network devices,  
network device/Aggregate: core-rtr-p-02,  
generate: generate alert,  
For metric: interface_in_unicast_packets,  
When: Average,  
Interval(seconds): 60,  
Is: Above,  
Threshold(Packets/s): 300,  
Severity: Warning,  
notification: custom service,  
services: appformix_to_saltstack,  
save.

## Watch webhook notifications and ZMQ messages  

Run this command on the master to see webhook notifications:
```
# tcpdump port 5001 -XX 
```

Salt provides a runner that displays events in real-time as they are received on the Salt master:  
```
# salt-run state.event pretty=True
```

## Trigger an alarm  to get a webhook notification sent by Appformix to SaltStack 

Either you DIY, or, depending on the alarms you set, you can use one the automation content available in the directory [trigger_alarms](trigger_alarms).  
Here's how to use the automation content available in the directory [trigger_alarms](trigger_alarms).  

### generate traffic between 2 routers 
Add the file [generate_traffic.sls](trigger_alarms/generate_traffic.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  

And run this command on the master:   
```
# salt "core-rtr-p-02" state.apply generate_traffic
```
### Change interface speed on a router

Add the file [change_int_speed.sls](trigger_alarms/change_int_speed.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  
Add the file [speed.set](trigger_alarms/speed.set) to the directory ```template``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).    
Run this command on the master:   
```
# salt "core-rtr-p-02" state.apply change_int_speed
# salt "core-rtr-p-02" junos.cli "show system commit"
# salt "core-rtr-p-02" junos.cli "show configuration | compare rollback 1"
# salt "core-rtr-p-02" junos.cli "show configuration interfaces ge-0/0/1"
```

### Change MTU on a router

Add the file [change_mtu.sls](trigger_alarms/change_mtu.sls) to the directory ```junos``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).  
Add the file [mtu.set](trigger_alarms/mtu.set) to the directory ```template``` of the gitlab repository ```organization/network_model``` (```gitfs_remotes```).    
Run this command on the master:   
```
# salt "core-rtr-p-02" state.apply change_mtu
# salt "core-rtr-p-02" junos.cli "show system commit"
# salt "core-rtr-p-02" junos.cli "show configuration | compare rollback 1"
# salt "core-rtr-p-02" junos.cli "show configuration interfaces ge-0/0/1"
```

## Verify on the git server 

The data collected by the proxy ```core-rtr-p-01```  is archived in the directory [core-rtr-p-01](core-rtr-p-01)  
The data collected by the proxy ```core-rtr-p-02```  is archived in the directory [core-rtr-p-02](core-rtr-p-02)

