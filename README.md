ansible-windows-docker-springboot
======================================================================================
[![Build Status](https://travis-ci.org/jonashackt/ansible-windows-docker-springboot.svg?branch=master)](https://travis-ci.org/jonashackt/ansible-windows-docker-springboot)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-jonashackt-660198.svg)](https://galaxy.ansible.com/jonashackt)

## Example project showing how to provision, deploy and run Spring Boot apps inside Docker Windows Containers on Windows Host using Packer, Powershell, Vagrant & Ansible

This is a follow-up to the repository [ansible-windows-springboot](https://github.com/jonashackt/ansible-windows-springboot) and the blog post [Running Spring Boot Apps on Windows with Ansible (codecentric.de)](https://blog.codecentric.de/en/2017/01/ansible-windows-spring-boot/). There are some corresponding follow up blog posts available:

* [Docker-Windows-Container mit Ansible managen (1/2)](https://www.heise.de/developer/artikel/Docker-Windows-Container-mit-Ansible-managen-1-2-3824736.html) on [heise developer](https://www.heise.de/developer/) (german only)
* [Running Spring Boot Apps on Docker Windows Containers with Ansible: A Complete Guide incl Packer, Vagrant & Powershell](https://blog.codecentric.de/en/2017/04/ansible-docker-windows-containers-spring-boot/)
* [Scaling Spring Boot Apps on Docker Windows Containers with Ansible: A Complete Guide incl Spring Cloud Netflix and Docker Compose](https://blog.codecentric.de/en/2017/05/ansible-docker-windows-containers-scaling-spring-cloud-netflix-docker-compose/)

This repository uses the following example Spring Boot / Cloud applications for provisioning: [cxf-spring-cloud-netflix-docker](https://github.com/jonashackt/cxf-spring-cloud-netflix-docker)

## Before you start...

Because [Microsoft &amp; Docker Inc. developed a native Docker implementation on Windows](https://blog.docker.com/2016/09/dockerforws2016/) using Hyper-V (or even a thinner layer) which let´s you run tiny little Windows containers inside your Windows box, which are accessible through the Docker API, I wanted to get my hands on them as soon as I heard of it. A list of [example Windows Docker Images is provided here](https://hub.docker.com/r/microsoft/).

Firing up Spring Boot apps with Ansible on Windows using Docker sound´s like the next step after [Running Spring Boot Apps on Windows with Ansible (codecentric.de)](https://blog.codecentric.de/en/2017/01/ansible-windows-spring-boot/).

> Before we start: The most important point here is to start with a __correct Build Number__ of Windows 10 (1607, Anniversary Update)/Windows Server 2016. It took me days to figure that out, but it won´t work with for example 10.0.14393.67 - but will with 10.0.14393.206! I ran over [the advice on this howto](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-10) to fast, because I thought "Windows 10 Anniversary should be enough, don´t bother me with build numbers". But take care! The final `docker run` won´t work (but all the other steps before, which made it so hard to understand)... Here are two examples of the output of a `(Get-ItemProperty -Path c:\windows\system32\hal.dll).VersionInfo.FileVersion` on a Powershell:

__Good build number:__

![Windows_build_number_Docker_working](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/Windows_build_number_Docker_working.png)

__Bad build number:__

![Windows_build_number_Docker_failing](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/Windows_build_number_Docker_failing.png)

Because of the minimal required build of Windows 10 or Server 2016, we sadly can´t use the easy to download and easy to handle [Vagrant box with Windows 10 from the Microsoft Edge developer site](https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/#downloads). So we have to look after an alternative box to start from. My goal was to start from a original Microsoft Image and show a 100% replicable way how to get to a running Vagrant box. Because besides the Microsoft Edge boxes (which have don´t have the correct build number for now) there aren´t any official from Microsoft around in [Vagrant Atlas](https://atlas.hashicorp.com/boxes/search?utf8=%E2%9C%93&sort=&provider=&q=windows+10). And hey - we´re dealing with Windows! I don´t want to have someone installing things on a VM I don´t know... 


## Preperation: Find a Windows Box - the Evalutation ISO

After a bit of research, you´ll find another way to evaluate a current Windows Version: The [Windows 10 Enterprise Evalutation ISO](https://www.microsoft.com/de-de/evalcenter/evaluate-windows-10-enterprise) or the [Windows Server 2016 Evalutation ISO](https://www.microsoft.com/de-de/evalcenter/evaluate-windows-server-2016). Both the Windows 2016 Server and the 10 Enterprise come with a 180 Days Evaluation licence (you have to register a live-ID for that).

> __DISCLAIMER:__ There are [two Windows Container Types](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/) : __Windows Server Containers__ (aka isolation level "process" or shared Windows kernel) and __Hyper-V-Containers__ (aka isolation level "hyper-v"). Windows 10 only supports the latter one. But Hyper-V-Containers seem not the thing you´re used to, when it comes to the Docker core concepts - because Docker relies on Process-Level isolation and __does not__ use a Hypervisor. So with that knowledge I would strongly encourage you to go with Windows Server 2016 and leave Windows 10 behind. At first glance it seems somehow "easier" to start with the "smaller" Windows 10. But don´t do that! I can also back this advice with lot´s of ours (when not days) trying to get things to work for myself or with customers - and finally switching to Windows Server and everything was just fine!

So if you really want to go with Windows 10 anyway, it shouldn´t be that much work to write your own Packer template and use the other ISO instead. Here we´ll stay with Windows Server 2016.


## Step 0 - How to build Ansible-ready Vagrant box from a Windows ISO ([step0-packer-windows-vagrantbox](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step0-packer-windows-vagrantbox))

The problem with an ISO - it´s not a nice Vagrant box we can fireup easily for development. But hey! There´s something for us: [packer.io](https://packer.io/). This smart tool is able to produce machine images in every flavour - also as a Vagrant box ;) And [from the docs](https://www.packer.io/docs/post-processors/vagrant.html): "[Packer] ... is in fact how the official boxes distributed by Vagrant are created." On a Mac you can install it with:

`brew install packer` 

We also install Windows completely [unattended](https://social.technet.microsoft.com/wiki/contents/articles/36609.windows-server-2016-unattended-installation.aspx) - which means, we don´t have to click on a single installation screen ;) And we configure it already completely for compatibility with Ansible. Which means several things:

* configure WinRM (aka Powershell remoting) correctly (including Firewall settings)
* install VirtualBox Guest tools (just for better usability)
* configure Ansible connectivity 

The WinRM connectivity is configured through the [Autounattend.xml[(https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/packer/Autounattend.xml). At the end we run the configure-ansible.ps1 - which will call the https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1. But this is done mostly for habing a better feeling, because WinRM should be configured already sufficiently. 

If you like to dig deeper into the myriads of configuration options, have a look into Stefan Scherers GitHub repositories, e.g. https://github.com/StefanScherer/docker-windows-box - where I learned everything I had to - and also borrowed the mentioned [Autounattend.xml[(https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/packer/Autounattend.xml) from. You can also create one yourself from ground up - but you´ll need a running Windows instance and then install the [Windows Assessment and Deployment Kit (Windows ADK)](https://developer.microsoft.com/de-de/windows/hardware/windows-assessment-deployment-kit).


#### Build your Windows Server 2016 Vagrant box

Download the [14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO](https://www.microsoft.com/de-de/evalcenter/evaluate-windows-server-2016) and place it into the __/packer__ folder.

Inside the `step0-packer-windows-vagrantbox` directory start the build with this command:

```
packer build -var iso_url=14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO -var iso_checksum=70721288bbcdfe3239d8f8c0fae55f1f windows_server_2016_docker.json
```

Now get yourself a coffee. This will take some time ;)

#### Add the box and run it


After successful packer build, you can init the Vagrant box (and receive a Vagrantfile):
```
vagrant init windows_2016_docker_virtualbox.box 
```

Now fire up your Windows Server 2016 box:

```
vagrant up
```


## Step 1 - Prepare your Windows Box to run Docker Windows Containers with Ansible ([step1-prepare-docker-windows](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step1-prepare-docker-windows))

If you don´t want to go with the discribed way of using packer to build your own Vagrant box and start with your own custom Windows Server 2016 machine right away - no problem! Just be sure to [prepare your machine correctly for Ansible](https://github.com/jonashackt/ansible-windows-springboot#prepare-the-windows-box-for-ansible-communication).

Now let´s check the Ansible connectivity. `cd..` into the root folder `ansible-windows-docker-springboot`:

```
ansible ansible-windows-docker-springboot-dev -i hostsfile -m win_ping
```

Getting a __SUCCESS__ responde, we can start to prepare our Windows box to run Windows Docker Containers. Let´s run the preparation playbook: 

```
ansible-playbook -i hostsfile prepare-docker-windows.yml --extra-vars "host=ansible-windows-docker-springboot-dev"
```

This does those things for you: 

* Checking, if you have the correct minimum build version of Windows
* Install the necessary Windows Features `containers` and `Hyper-V` (this is done Windows Version agnostic - so it will work with Windows 10 AND Server 2016 - which is quite unique, becaue Microsoft itself always distinquishes between these versions)
* Reboot your Windows Box, if necessary
* Install the current Docker version (via [chocolatey docker package](https://chocolatey.org/packages/docker). And although the package claims to only install the client, it also provides the Docker Server (which means this is 100% identical with the [step 2. Install Docker in Microsoft´s tutorial](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-10)).)
* Register and Start the Docker Windows service
* Installing docker-compose (this is only needed for multiple containers)
* Running a first Windows container inside your Windows box (via `docker run microsoft/dotnet-samples:dotnetapp-nanoserver`)
* Building the `springboot-oraclejre-nanoserver` Docker image to run our Spring Boot Apps later on


If Docker on Windows with Windows Docker Containers is fully configured, you should see something like this (which definitely means, Docker is running perfectly fine on your Windows box!):

```
TASK [Docker is ready on your Box and waiting for your Containers :)] **********
ok: [127.0.0.1] => {
    "msg": [
        "", 
        "        Dotnet-bot: Welcome to using .NET Core!", 
        "    __________________", 
        "                      \\", 
        "                       \\", 
        "                          ....", 
        "                          ....'", 
        "                           ....", 
        "                        ..........", 
        "                    .............'..'..", 
        "                 ................'..'.....", 
        "               .......'..........'..'..'....", 
        "              ........'..........'..'..'.....", 
        "             .'....'..'..........'..'.......'.", 
        "             .'..................'...   ......", 
        "             .  ......'.........         .....", 
        "             .                           ......", 
        "            ..    .            ..        ......", 
        "           ....       .                 .......", 
        "           ......  .......          ............", 
        "            ................  ......................", 
        "            ........................'................", 
        "           ......................'..'......    .......", 
        "        .........................'..'.....       .......", 
        "     ........    ..'.............'..'....      ..........", 
        "   ..'..'...      ...............'.......      ..........", 
        "  ...'......     ...... ..........  ......         .......", 
        " ...........   .......              ........        ......", 
        ".......        '...'.'.              '.'.'.'         ....", 
        ".......       .....'..               ..'.....", 
        "   ..       ..........               ..'........", 
        "          ............               ..............", 
        "         .............               '..............", 
        "        ...........'..              .'.'............", 
        "       ...............              .'.'.............", 
        "      .............'..               ..'..'...........", 
        "      ...............                 .'..............", 
        "       .........                        ..............", 
        "        .....", 
        "", 
        "", 
        "**Environment**", 
        "Platform: .NET Core 1.0", 
        "OS: Microsoft Windows 10.0.14393 ", 
        ""
    ]
}

```


## Step 2 - How to run a simple Spring Boot App inside a Docker Windows Container with Ansible ([step2-single-spring-boot-app](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step2-single-spring-boot-app))


Everything needed here is inside [step2-single-spring-boot-app](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step2-single-spring-boot-app). Be sure to have cloned and (Maven-) build the example simple Spring Boot app [weatherbackend](https://github.com/jonashackt/spring-cloud-netflix-docker/tree/master/weatherbackend). Let´s cd into `step2-single-spring-boot-app` and run the playbook:

```
ansible-playbook -i hostsfile ansible-windows-docker-springboot.yml --extra-vars "host=ansible-windows-docker-springboot-dev app_name=weatherbackend jar_input_path=../../cxf-spring-cloud-netflix-docker/weatherbackend/target/weatherbackend-0.0.1-SNAPSHOT.jar"
```

This should run a single Spring Boot app inside a Docker Windows Container on your Windows box.

![spring-boot-example-running-docker-windows-containers](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/spring-boot-example-running-docker-windows-containers.png)

## Step 3 - How to scale multiple Spring Boot Apps inside a Docker Windows Containers with Ansible, docker-compose and Spring Cloud Netflix ([step4-multiple-spring-boot-apps-docker-compose](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step4-multiple-spring-boot-apps-docker-compose))


Everything needed here is inside [step3-multiple-spring-boot-apps-docker-compose](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step3-multiple-spring-boot-apps-docker-compose). Be sure to have cloned and (Maven-) build the complete Spring Cloud example apps [cxf-spring-cloud-netflix-docker](https://github.com/jonashackt/spring-cloud-netflix-docker).  Let´s cd into `step3-multiple-spring-boot-apps-docker-compose` and run the playbook:

```
ansible-playbook -i hostsfile ansible-windows-docker-springboot.yml --extra-vars "host=ansible-windows-docker-springboot-dev"
```

This will fire up multiple containers running Spring Boot Apps inside Docker Windows Containers on your Windows box. They will leverage the power of Spring Cloud Netflix with Zuul as a Proxy and Eureka as Service Registry (which dynamically tells Zuul, which Apps to route).

But with [docker-compose](https://docs.docker.com/compose/) you are now able to __easily fire up your hole application with one command__ (`docker-compose`) and deployment with Ansible also gets much faster through that (all the Docker containers are build at once and startet in parallel).
Additionally, it is now possible to easily scale your Containers. If you want to scale the weatherbackend from 1 to 5 containers for example, just do the following inside __c:\spring-boot\__ :

```
docker-compose scale weatherbackend=5
```

A few seconds later (depending on the power of your machine), you should be able to see them all in Eureka:

![spring-cloud-example-running-docker-windows-containers-docker-compose](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/spring-cloud-example-running-docker-windows-containers-docker-compose.png)



## Best practices

##### Using Powershell on Host to Connect to Container

```
docker ps -a 
```
Look up your containers´ id, then do
```
docker exec -ti YourContainerIdHere powershell
```

##### Check if your Spring Boot app is running inside the Container

```
iwr http://localhost:8080/swagger-ui.html -UseBasicParsing
```

##### Set Proxy with Ansible, if you have a corporate firewall

See https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon:

```
  - name: Set Proxy for docker pull to work (http)
    win_environment:
      state: present
      name: HTTP_PROXY
      value: http://username:password@proxy:port/
      level: machine

  - name: Set Proxy for docker pull to work (https)
    win_environment:
      state: present
      name: HTTPS_PROXY
      value: http://username:password@proxy:port/
      level: machine
```

## Known Issues

If something doesnt work as expected, see this guide here https://docs.microsoft.com/en-us/virtualization/windowscontainers/troubleshooting

Especially this command here is useful to check, whether something isn´t working as expected:
```
Invoke-WebRequest https://aka.ms/Debug-ContainerHost.ps1 -UseBasicParsing | Invoke-Expression
```

And network seems to be a really tricky part with all this non-localhost, Hyper-V network-stuff ...

#### Network

Good overview:

![windows-docker-network-architecture](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/windows-docker-network-architecture.png)
(from https://blogs.technet.microsoft.com/virtualization/2016/05/05/windows-container-networking/)

###### Useful commands

Show Docker Networks
```
docker network ls
```

Inspect one of these networks
```
docker network inspect networkNameHere
```

A good (e.g. working) starting configuration for the Windows Docker Network shows something like this

![docker-network-correct-nat-config](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/docker-network-correct-nat-config.png)

If the "IPAM" section shows an empty Subnet & Gateway, you may have the problem, that your NAT wont work and you can´t connect to your Docker-Containers from the Windows Docker Host itself (see Caveats and Gotchas section on https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-networking)

###### localhost to forward to Windows Containers isn´t working as expected

[On Windows it isn´t possible to do what you know from Linux](https://blog.sixeyed.com/published-ports-on-windows-containers-dont-do-loopback/): Run a `docker run -d -p 80:80 microsoft/iis` and go to `http://localhost` won´t work sadly! But before I hear you scream: "Hey, why is that `-p 80:80` thingy for - if that simple thing isn´t working?!" Well, if you come from outside the Windows Docker Host Maschine and try this IP, it will work - so everything will work, except of your localhost-Tests :D

> But this is only temporarily --> The Windows Docker team is on it and the fix will be released soon as a Windows Update - see [github Issue 458](https://github.com/docker/for-win/issues/458)

###### Network docs

Helpful knowledge! Docker Windows Networks work slightly different to Linux ones (localhost!) https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-networking

https://blogs.technet.microsoft.com/virtualization/2016/05/05/windows-container-networking/

https://4sysops.com/archives/windows-container-networking-part-1-configuring-nat/

## Resources

##### Microsoft & Docker Inc docs

[Windows Containers Documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/index)

[Configure Docker on Windows](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-docker/configure-docker-daemon)

https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-server

https://docs.docker.com/docker-for-windows/troubleshoot/

https://docs.docker.com/docker-for-windows/#docker-settings

https://www.docker.com/microsoft

https://docs.com/taylorb/8408/dockercon-2016-dockerizing-windows-server?c=4TNCe4

https://blogs.technet.microsoft.com/virtualization/2016/10/18/use-docker-compose-and-service-discovery-on-windows-to-scale-out-your-multi-service-container-application/

https://github.com/Microsoft/Virtualization-Documentation/tree/live/windows-container-samples

https://blogs.technet.microsoft.com/virtualization/2017/02/09/overlay-network-driver-with-support-for-docker-swarm-mode-now-available-to-windows-insiders-on-windows-10/

Newest Insider builds: https://insider.windows.com/ or here https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewadvanced


##### Good resources

https://blog.docker.com/2016/09/build-your-first-docker-windows-server-container/

[Walktrough Windows Docker Containers](https://github.com/artisticcheese/artisticcheesecontainer/wiki)

[Video: John Starks’ black belt session about Windows Server & Docker at DockerCon ‘16](https://www.youtube.com/watch?v=85nCF5S8Qok)

https://blog.sixeyed.com/windows-containers-and-docker-5-things-you-need-to-know/

https://github.com/StefanScherer/dockerfiles-windows

https://github.com/joefitzgerald/packer-windows

https://github.com/StefanScherer/docker-windows-box

Install docker-compose on Windows Server 2016: https://github.com/docker/for-win/issues/448#issuecomment-276328342

If Service discovery doen´t work reliable: http://stackoverflow.com/questions/43041297/docker-dns-for-service-discovery-to-resolve-windows-container%C2%B4s-address-by-name/43041298#43041298



# Docker Container Orchestration with Linux & Windows mixed OS setup

Example steps showing how to provision and run Spring Boot Apps with Docker Swarm &amp; Docker in mixed mode on Linux AND Windows (Docker Windows Containers!)

## Why more?

We went quite fare with that setup - and broke up most boundaries inside our heads, what´s possible with Windows. But there´s one step left: leaving the one machine our services are running on and do a step further to go for a multi-machine setup, incl. blue-green-deployments/no-time-out-deployments and kind of "bring-my-hole-app-UP" (regardles, on which server it is running)

## Kubernetes or Docker Swarm?

Everything seems to point to Kubernetes - biggest mediashare, most google searches, most blog posts and so on. But there´s one thing that __today__ bring´s me on the Docker Swarm path: And that´s the __Docker Windows Container Support__ in the current feature set implemented. As of Kubernetes 1.6 Windows Server 2016 (which is capable of running Windows Server Containers) there´s a basic support of Docker Windows Containers in Kubernetes - with two main limitations:

* Networksubsystem HNS isn´t really Kubernetes-ready - so you have to manually put Routingtables together
* Only one Docker Container per Pod is supported right now

Both things mean, that you litterally can´t leverage the benefits of Kubernetes as a Container Orchestration tool with Docker Windows Containers right now. Things might change soon though, if Microsoft releases it´s Version 1709 Windows Server 2016 and Kubernetes 1.8 goes live. But both isn´t now, so we first of all have to go with the competitor Docker Swarm, which should be also a good thing to start - and we´ll later switch to Kubernetes.


## Step 4 - A Multi-machine Windows- & Linux- mixed OS Vagrant setup for Docker Swarm ([step4-windows-linux-multimachine-vagrant-docker-swarm-setup](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step4-windows-linux-multimachine-vagrant-docker-swarm-setup))

There are basically two options to achieve a completely comprehensible setup: running more than one virtual machine on your local machine or go into the cloud. To decide which way to go, I had to rethink about what I wanted to show with this project. My goal is to show a setup of an Docker Orchestration tool to scale Docker Containers on both Windows and Linux - without messing with the specialities of one of the many cloud providers. Not to mention the financial perspective. So for the first setup, I wanted to go with a few virtual machines that run on my laptop.

As I really got to love Vagrant as a tool to handle my Virtual machines, why not do it with that again? And thank´s to a colleague of mine´s hint, I found the [Vagrant multi-machine docs](https://www.vagrantup.com/docs/multi-machine/).


Inside the `step0-packer-windows-vagrantbox` directory start the build for another Windows box (that does not provide a provider config, which wouldn´t work within a Vagrant multimachine setup) with this command:

```
packer build -var iso_url=14393.0.161119-1705.RS1_REFRESH_SERVER_EVAL_X64FRE_EN-US.ISO -var iso_checksum=70721288bbcdfe3239d8f8c0fae55f1f -var template_url=vagrantfile-windows_2016-multimachine.template -var box_output_prefix=windows_2016_docker_multimachine windows_server_2016_docker.json
```

Add new Windows 2016 Vagrant box:
```
vagrant box add --name windows_2016_multimachine windows_2016_docker_multimachine_virtualbox.box
```

Now switch over to `step4-windows-linux-multimachine-vagrant` directory. Here´s the Vagrantfile defining our local Cloud infrastructure. It defines 4 machines to show the many possible solutions in a hybrid Docker Swarm containing Windows and Linux boxes: Manager nodes both as Windows and Linux machines and Worker nodes, also both as Windows and Linux machines:

* masterlinux01
* workerlinux01
* masterwindows01
* workerwindows01


PICTURE!

Within a Vagrant multimachine setup, you define your separate machines with the `config.vm.define` keyword. Inside those define blocks we simply configure our individual machine. Let´s have a look onto the `workerlinux`:

```
    # One Worker Node with Linux
    config.vm.define "workerlinux" do |workerlinux|
        workerlinux.vm.box = "ubuntu/trusty64"
        workerlinux.vm.hostname = "workerlinux01"
        workerlinux.ssh.insert_key = false
        # Forwarding the port for Ansible explicitely to not run into Vagrants 'Port Collisions and Correction'
        # see https://www.vagrantup.com/docs/networking/forwarded_ports.html, which would lead to problems with Ansible later
        workerlinux.vm.network "forwarded_port", guest: 22, host: 2232, host_ip: "127.0.0.1", id: "ssh"

        # As to https://www.vagrantup.com/docs/multi-machine/ & https://www.vagrantup.com/docs/networking/private_network.html
        # we need to configure a private network, so that our machines are able to talk to one another
        workerlinux.vm.network "private_network", ip: "172.16.2.11"

        workerlinux.vm.provider :virtualbox do |virtualbox|
            virtualbox.name = "WorkerLinuxUbuntu"
            virtualbox.gui = true
            virtualbox.memory = 2048
            virtualbox.cpus = 2
            virtualbox.customize ["modifyvm", :id, "--ioapic", "on"]
            virtualbox.customize ["modifyvm", :id, "--vram", "16"]
        end
    end
...
```

The first configuration statements are usual ones like configuring the Vagrant box to use or the VM´s hostname. But the fowarded port configuration is made explicit, because we need to rely on the exact port later in our Ansible scripts. And this wouldn´t be possible with Vagrants default [Port Correction feature](https://www.vagrantup.com/docs/networking/forwarded_ports.html). Because you couldn´t use a port on your host machine more then once, Vagrant would automatically set it to a random value - and we weren´t able to access our boxes later with Ansible.

To define and override the SSH port of a preconfigured Vagrant box, we need to know the `id` which is used to define it in the base box. On Linux boxes this is `ssh` - and on Windows this is `winrm-ssl`.


###### Host-only Network configuration

The next tricky part is the network configuration between the Vagrant boxes. As they need to talk to each other and also to the Host, the so called [Host-only networking](http://www.virtualbox.org/manual/ch06.html#network_hostonly) should be the way to go here (there´s a really good [overview in this post](https://www.thomas-krenn.com/de/wiki/Netzwerkkonfiguration_in_VirtualBox#Host-only_networking), sorry german only). This is easily established using [Vagrants Private Networks configuration](https://www.vagrantup.com/docs/networking/private_network.html).

And as we want to access our boxes with a static IP, we leverage the Vagrant configuration around [Vagrant private networking](https://www.vagrantup.com/docs/networking/private_network.html). All that´s needed here, is to have such a line inside every Vagrant box definition of our multi-machine setup:

```
masterlinux.vm.network "private_network", ip: "172.16.2.10"
```

Same for Windows boxes, Vagrant will tell VirtualBox to create a new separate network (mostly `vboxnet1` or similar), put a second virtual network device into every box and assign with the static IP, we configured in our Vagrantfile. That´s pretty much everything, except for Windows Server :) 







#### Network configuration between Vagrant Boxes and the Host

As Ansible is a really nice tool, that let´s you use the same host in multiple groups - and merges the group_vars from all of those according to that one host - it isn´t a good idea to use a structure like that in your inventory file:

```
[masterwindows]
127.0.0.1

[masterlinux]
127.0.0.1

[workerwindows]
127.0.0.1
```

And try to use different corresponding group_vars entries... Because, you don´t know, which variables will be present!

See https://github.com/ansible/ansible/issues/9065


###### Different hostnames 

Current workaround: configure ~/hosts

```
127.0.0.1 masterlinux01
127.0.0.1 workerlinux01
127.0.0.1 masterwindows01
127.0.0.1 workerwindows01
```

do a:

```
vagrant up
```

Now we´re ready to play. And nevermind, if you want to have a break or your notebook is running hot - just type a `vagrant halt`. And the whole zoo of machines will be stopped for you :)








###### Windows Server firewall blocks Ping & later needed Container network traffic

As you may noticed, there´s an extra for Windows Server 2016. Because we want our machines to be accessible from each other, we have to allow the very basic command everybody start´s with: the ping. That one [is blocked by the Windows firewall as a default](https://www.rootusers.com/how-to-enable-ping-in-windows-server-2016-firewall/) and we have to open that up with the following [Powershell command](https://technet.microsoft.com/de-de/library/dd734783(v=ws.10).aspx#BKMK_3_add) - obviously wrapped inside a Ansible task:

```
  - name: Allow Ping requests on Windows nodes (which is by default disabled in Windows Server 2016)
    win_shell: "netsh advfirewall firewall add rule name='ICMP Allow incoming V4 echo request' protocol=icmpv4:8,any dir=in action=allow"
    when: inventory_hostname in groups['workerwindows']
```

Additionally, and this part is mentioned pretty much at the end of the docker docs if you want to fire up a Swarm, the later established routing network needs access to several ports, [as the docs state](https://docs.docker.com/engine/swarm/ingress/). __AND__ "you need to have the following ports open between the swarm nodes before you enable swarm mode". So we need to do that __before__ even initializing our Swarm!

tbd

But what if we were able to change the /etc/hosts on our Host machine with every `vagrant up`? (https://stackoverflow.com/questions/16624905/adding-etc-hosts-entry-to-host-machine-on-vagrant-up) That´s possible with the https://github.com/cogitatio/vagrant-hostsupdater, install it with:

```
vagrant plugin install vagrant-hostsupdater
```

###### Prepare Docker engines on all Nodes

> __working Ansible SSH config__: you´ll maybe need to install sshpass (e.g. via `brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb` (as `brew install sshpass` won´t work, see https://stackoverflow.com/questions/32255660/how-to-install-sshpass-on-mac:

Before (see https://stackoverflow.com/questions/34718079/add-host-to-known-hosts-file-without-prompt/34721229)

```
export ANSIBLE_HOST_KEY_CHECKING=False
```

Now run the following Ansible playbook to prepare your Nodes with a running Docker Engine:

```
ansible-playbook -i hostsfile prepare-docker-nodes.yml
```

```
unset ANSIBLE_HOST_KEY_CHECKING
```

###### Allowing http based local Docker Registries

But as the [limitations section](https://docs.docker.com/compose/swarm/#limitations) states, we can´t follow our old approach to build our Docker images on the Docker host anymore - because that way we were forced to build those images on all the Swarm´s nodes again and again, which leads to heavy overhead. We should therefore build the Applications Docker image only once and push it into a local Docker registry. But before that, we´ll need such a local registry. This topic is also covered in the [Docker docs](https://docs.docker.com/registry/deploying/#run-the-registry-as-a-service).

The easiest start here is to use a [plain http registry](https://docs.docker.com/registry/insecure/#deploy-a-plain-http-registry). It´s no problem to run this setup in isolated environments, such as your on-premise servers. Just be sure to update to https with TLS certificates, if you´re going into the cloud or if you want to provide your registry to other users outside this Swarm. 

So let´s go. First of all, we need to allow our Docker Engines on all our hosts to interact with an http-only Docker Registry. Therefore we create a `daemon.json` file with the following entry on all of our nodes:

```
{
  "insecure-registries" : ["172.16.2.12:5000"]
}
```

The file has to reside in `/etc/docker/daemon.json` on Linux and on `C:\ProgramData\docker\config\daemon.json` on Windows.






#### Initializing a Docker Swarm

```
ansible-playbook -i hostsfile initialize-docker-swarm.yml
```

Obtaining the worker join-token from the Windows master node isn´t a big problem with Ansible:

```
  - name: Obtain worker join-token from Windows master node
    win_shell: "docker swarm join-token worker -q"
    register: token_result
    ignore_errors: yes
    when: inventory_hostname in groups['masterwindows']
```

But syncing the join-token to the other hosts is a bit tricky, since variables or facts are just defined per Host in Ansible. But there´s help! We only need to use the doc´ info about [Magic Variables, and How To Access Information About Other Hosts](http://docs.ansible.com/ansible/latest/playbooks_variables.html#magic-variables-and-how-to-access-information-about-other-hosts). First of all we access the return variable `token_result` from the Windows master Host `{{ hostvars['masterwindows01']['token_result'] }}` - remember to use the exact Host name here, the group name won´t be enough. The second step is the extraction of the join-token out of the result variable with the help of the set_fact Module. The following two Ansible tasks demonstrate the solution:

```
  - name: Syncing the worker join-token result to the other hosts
    set_fact:
      token_result_host_sync: "{{ hostvars['masterwindows01']['token_result'] }}" #"{{token_result.stdout.splitlines()[0]}}"

  - name: Extracting and saving worker join-token in variable for joining other nodes later
    set_fact:
      worker_jointoken: "{{token_result_host_sync.stdout_lines}}"

```

###### Providing a Docker Registry

As state already in the previous section, we configured every Docker Engine on every Swarm node to enable http only Docker Registry access. Now let´s start our Docker Swarm Registry Service [as mentioned in the docs](https://docs.docker.com/registry/deploying/#run-the-registry-as-a-service). BUT: Currently the docs are wrong - [we´ve got it fixed already here](https://github.com/docker/docker.github.io/pull/4465): 

```
  - name: Specify to run Docker Registry on Linux Manager node
    shell: "docker node update --label-add registry=true masterlinux01"
    ignore_errors: yes
    when: inventory_hostname == "masterlinux01"

  - name: Create directory for later volume bind-mount into the Docker Registry service on Linux Manager node, if it doesn´t exist
    file:
      path: /mnt/registry
      state: directory
      mode: 0755
    when: inventory_hostname == "masterlinux01


  - name: Run Docker Registry on Linux Manager node as Docker Swarm service
    shell: "docker service create --name swarm-registry --label registry=true --mount type=bind,src=/mnt/registry,dst=/var/lib/registry -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 -p 5000:5000 --replicas 1 registry:2"
    ignore_errors: yes
    when: inventory_hostname == "masterlinux01"

```

As the [docs do propose a bind-mount](https://docs.docker.com/registry/deploying/#run-the-registry-as-a-service), we have to add `type=bind` into our `--mount` configuration parameter. AND: We have to create the directory `/mnt/registry` beforehand, as the [docs about "Give a service access to volumes or bind mounts" are stating](https://docs.docker.com/engine/swarm/services/#give-a-service-access-to-volumes-or-bind-mounts). But it seems, that not all the docs are up-to-date here, see https://github.com/docker/docker.github.io/pull/4641.


###### Visualize the Swarm

Docker´s own Swarm visualizer doesn´t look that neat, so I read about a comparison with Portainer: https://stackshare.io/stackups/docker-swarm-visualizer-vs-portainer Seems to be way more prettier! And it say´s in it´s GitHub readme: "can be deployed as Linux container or a Windows native container". So let´s integrate it into our setup: 

https://github.com/portainer/portainer & https://portainer.readthedocs.io/en/latest/deployment.html

We therefore integrated Portainer into the initialization process of our Swarm:

```
- name: Create directory for later volume mount into Portainer service on Linux Manager node, if it doesn´t exist
  file:
    path: /mnt/portainer
    state: directory
    mode: 0755
  when: inventory_hostname in groups['linux']

- name: Run Portainer Docker and Docker Swarm Visualizer on Linux Manager node as Swarm service
  shell: "docker service create --name portainer --publish 9000:9000 --constraint 'node.role == manager' --constraint 'node.labels.os==linux' --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock --mount type=bind,src=/mnt/portainer,dst=/data portainer/portainer -H unix:///var/run/docker.sock"
  ignore_errors: yes
  when: inventory_hostname == "masterlinux01"
```

This will deploy a Portainer instance onto our Linux Manager nodes (masterlinux01, cause we only have one Linux Manager node) and connect it directly to the Swarm.

But there´s one thing, that could lead to frustration: Use a current Browser to access Portainer UI inside your Windows Boxes! It doesn´t work inside the pre-installed IE! Just head to http://172.16.2.10:9000.

![first-docker-swarm-services-running-portainer](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/first-docker-swarm-services-running-portainer.png)


###### Checking swarm status

Just do a `docker info` on one (or all) of the boxes.

Now that we also added Docker labels (like this `docker node update --label-add os=linux masterlinux01`) to each of our nodes so that we can differentiate the OS dependend services later on and also created a Docker Swarm overlay network with `docker network create --driver=overlay mixed_swarm_net`, our Ansible playbook should finally give some output like this:

```
TASK [Swarm initialized...] *****************************************************************************************************************************
skipping: [workerwindows01]
skipping: [masterlinux01]
ok: [masterwindows01] => {
    "msg": [
        "The status of the Swarm now is:", 
        [
            "ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS", 
            "ar2mci5utfwov44x42fihgmtf     masterlinux01       Ready               Active              Reachable", 
            "qdcarj7mzjl37txmsijgrvxt4     workerwindows01     Ready               Active              ", 
            "sqirk9itzxlytf5blteg9no7w *   masterwindows01     Ready               Active              Leader", 
            "vz8ruili76n8fslo2vo35go3b     workerlinux01       Ready               Active              "
        ]
    ]
}
skipping: [workerlinux01]
```

This means that our Docker Swarm cluster is ready for service deployment!


## Step 5 - Deploy multiple Spring Boot Apps on mixed-OS Docker Windows- & Linux Swarm with Ansible ([step5-deploy-multiple-spring-boot-apps-to-mixed-os-docker-swarm](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step5-deploy-multiple-spring-boot-apps-to-mixed-os-docker-swarm))

As Microsoft states in the [Swarm docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/swarm-mode), Docker Swarm Services can be easily deployed to the Swarm with the `docker service create` command and afterwards scaled with `docker service scale`. There´s a huge amount on configuration parameters you can use [with docker service create](https://docs.docker.com/engine/swarm/services/).

__BUT:__ That approach reminds us of those first days with Docker not using Docker Compose. So it would be really nice to have something like Compose also for our Swarm deployment. And it´s really that simple - [just use Compose with Docker Stack Deploy for that :)](https://docs.docker.com/engine/swarm/stack-deploy/):

> "Docker Compose and Docker Swarm aim to have full integration, meaning you can point a Compose app at a Swarm cluster and have it all just work as if you were using a single Docker host."

So let´s directly break a architectural rule we build up in the preceding work: Don´t extensively template the docker-compose.yml any more, because this would lead to a rather complex setup process in Ansible. And remember: we´ve only did this, because we wanted to be free in our choice of Container orchestration technology - but now we choose Docker Swarm, we can savely forget this approach.

Back to the concrete docker-compose.yml file. Let´s use the [newest 3.3 version](https://docs.docker.com/compose/compose-file/compose-versioning/#version-33) here, so that we can leverage the most out of Swarm´s functionality, which is broadened with each Compose (file) version.


#### Pre-Deploy preparations: Build Docker images of all Spring Boot apps and push them to Docker Swarm registry

cd into [step5-deploy-multiple-spring-boot-apps-to-mixed-os-docker-swarm](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step5-deploy-multiple-spring-boot-apps-to-mixed-os-docker-swarm) and run:

```
ansible-playbook -i hostsfile build-and-deploy-apps-2-swarm.yml
```

From https://docs.docker.com/get-started/part5/#introduction:
> "A stack is a group of interrelated services that share dependencies, and can be orchestrated and scaled together. A single stack is capable of defining and coordinating the functionality of an entire application."

Think of a Stack as like what Compose is for Docker - grouping multiple Docker Swarm services together with the help of a docker-stack.yml (which looks like a docker-compose.yml file and uses nearly the same syntax (Stack has 'deploy' over Compose)). An example `docker-stack.yml` looks like:

```
tbadded
```

Don´t try to search for "Docker Stack command reference", just head over to the Docker Compose docs and you should find, what you need: https://docs.docker.com/compose/compose-file/#deploy Because Docker Swarm makes use of Docker Compose files, the Swarm capabilities of Stack are only just a section (`deploy`) in the Compose docs.


We should see our applications in Portainer now:

![portainer-swarm-visualizer](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/portainer-swarm-visualizer.png)

#### Accessing Spring Boot applications deployed in the Swarm

Fore more indepth information how Docker Swarm works, have a look at https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/

![how-swarm-services-work](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/how-swarm-services-work.png)

To get to know, where your App is accessible in the Swarm, there are some commands you can use. On a manager node do a

```
docker service ls
```

to see all the deployed Docker Swarm services. It should output something like:

![docker-service-ls](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/docker-service-ls.png)

Now pick one of your Services to inspect and do a

```
docker service ps clearsky_weatherbackend
```

This should show us, on which node the Swarm manager is running the Docker Swarm task including our App´s container:

![docker-service-ps-app](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/docker-service-ps-app.png)

It´s `workerlinux01` in this example.

If you´re unsure, which Port is mapped to the Docker node workerlinux01, you could run a:

```
docker service inspect --pretty clearsky_weatherbackend
```

This should give you more insights into this app, including the mapped Port 30001:

![docker-service-inspect-app](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/docker-service-inspect-app.png)

With all this information, you could check out your first Docker Swarm deployed App. Just log into `workerlinux01` and call your App, e.g. with a `curl http://localhost:30001/swagger-ui.html` - as the [weatherbackend](https://github.com/jonashackt/cxf-spring-cloud-netflix-docker) is usind [Springfox](https://github.com/springfox/springfox) together with Swagger to show all of it´s REST endpoints:

![curl-linux-container](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/docker-service-inspect-app.png)

As Windows doesn´t support localhost loopback, we have to add one more step, to access an App which is deployed into a Windows native Docker Container: We need to know the Container´s IP:


__BUT:__ We´re not in Docker Engine´s standard mode anymore, we´re in Swarm mode. So the Ports aren´t mapped to the Host we define, but to the Docker Swarm as a hole. How does this work? This is done through Docker Swarm Routing Mesh:

> "When you publish a service port, the swarm routing mesh makes the service accessible at the target port on every node regardless if there is a task for the service running on the node."

https://docs.docker.com/engine/swarm/ingress/

![ingress-routing-mesh](https://github.com/jonashackt/ansible-windows-docker-springboot/blob/master/ingress-routing-mesh.png)

Note: this is only possible with Windows Server 2016, if you opened the needed ports before initializing the Swarm (this is already done in [step4-windows-linux-multimachine-vagrant-docker-swarm-setup](https://github.com/jonashackt/ansible-windows-docker-springboot/tree/master/step4-windows-linux-multimachine-vagrant-docker-swarm-setup) for you ;) ).






# Links

#### General comparison of Docker Container Orchestrators

mindshare: https://platform9.com/blog/kubernetes-docker-swarm-compared/

marketshare: https://blog.netsil.com/kubernetes-vs-docker-swarm-vs-dc-os-may-2017-orchestrator-shootout-fdc59c28ec16

https://www.loomsystems.com/blog/single-post/2017/06/19/kubernetes-vs-docker-swarm-vs-apache-mesos-container-orchestration-comparison

#### Windows Server

https://blogs.technet.microsoft.com/hybridcloud/

https://blogs.technet.microsoft.com/hybridcloud/2017/05/10/windows-server-for-developers-news-from-microsoft-build-2017/

Windows Server Pre-Release (Insider): https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewserver

Current state discription: https://blogs.windows.com/windowsexperience/2017/07/13/announcing-windows-server-insider-preview-build-16237/#tx4mFJzTSMIjl2gX.97 --> coming version 1709 of Windows Server 2016 will have better Kubernetes support with no more manual tinkering with routing tables (better HNS)

#### Docker Swarm

Docker Swarm Windows Docs: https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/swarm-mode

Windows Server 2016 Overlay Networking Support (Windows & Linux mixed mode): https://blogs.technet.microsoft.com/virtualization/2017/04/18/ws2016-overlay-network-driver/

Windows & Linux mixed Video: https://www.youtube.com/watch?v=ZfMV5JmkWCY

https://docs.docker.com/engine/swarm/

https://docs.docker.com/engine/swarm/key-concepts/

https://docs.docker.com/engine/swarm/services/

docker service create CLI reference: https://docs.docker.com/engine/reference/commandline/service_create/

https://docs.docker.com/engine/swarm/stack-deploy/

https://codefresh.io/blog/deploy-docker-compose-v3-swarm-mode-cluster/

#### Kubernetes

Docker Windows Containers & Kubernetes: https://blogs.technet.microsoft.com/networking/2017/04/04/windows-networking-for-kubernetes/

Kubernetes Networking on Windows: https://www.youtube.com/watch?v=P-D8x2DndIA&t=6s&list=PL69nYSiGNLP2OH9InCcNkWNu2bl-gmIU4&index=1

http://www.serverwatch.com/server-news/why-kubernetes-sucks-and-how-to-fix-it.html

http://blog.kubernetes.io/2016/12/windows-server-support-kubernetes.html

https://www.youtube.com/watch?v=Tbrckccvxwg

https://kubernetes.io/docs/getting-started-guides/windows/

https://github.com/kubernetes/features/issues/116

minikube howto: http://www.sqlservercentral.com/blogs/the-database-avenger/2017/06/13/orchestrating-sql-server-with-kubernetes/

https://groups.google.com/forum/#!forum/kubernetes-sig-windows

http://blog.kubernetes.io/2017/08/kompose-helps-developers-move-docker.html?m=1

http://blog.kubernetes.io/2017/09/windows-networking-at-parity-with-linux.html



