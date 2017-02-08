ansible-windows-docker-springboot
======================================================================================
[![Build Status](https://travis-ci.org/jonashackt/ansible-windows-docker-springboot.svg?branch=master)](https://travis-ci.org/jonashackt/ansible-windows-docker-springboot)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-jonashackt-660198.svg)](https://galaxy.ansible.com/jonashackt)

## Example project showing how to provision, deploy and run a Spring Boot app with Docker Native on Windows using Ansible &amp; chocolatey

This is a follow-up to the repository [ansible-windows-springboot](https://github.com/jonashackt/ansible-windows-springboot) and the blog post [Running Spring Boot Apps on Windows with Ansible (codecentric.de)](https://blog.codecentric.de/en/2017/01/ansible-windows-spring-boot/).

Because [Microsoft &amp; Docker Inc. developed a native Docker implementation on Windows](https://blog.docker.com/2016/09/dockerforws2016/) using Hyper-V (or even a thinner layer) which let´s you run tiny little Windows containers inside your Windows box, which are accessible through the Docker API, I wanted to get my hands on them as soon as I heard of it. A list of [example Windows Docker Images is provided here](https://hub.docker.com/r/microsoft/).

Firing up Spring Boot apps with Ansible on Windows using Docker sound´s like the next step after [Running Spring Boot Apps on Windows with Ansible (codecentric.de)](https://blog.codecentric.de/en/2017/01/ansible-windows-spring-boot/).

Every [Prerequisite](https://github.com/jonashackt/ansible-windows-springboot#prerequisites), [preparation step](https://github.com/jonashackt/ansible-windows-springboot#prepare-the-windows-box-for-ansible-communication) and the need to [choose a Spring Boot app](https://github.com/jonashackt/ansible-windows-springboot#choose-an-spring-boot-app-to-deploy) to run stays the same.

Bringing Docker to the table, we should start with the following step:




### Additional Preparation Steps for Windows with native Docker

###### Install Container feature:

```
Enable-WindowsOptionalFeature -Online -FeatureName containers -All
```

Only needed, because Ansible doesn´t support installation of Windows features in Windows 10 non-Server edition, although it is possible with the [Anniversary Edition](https://docs.microsoft.com/en-us/virtualization/windowscontainers/quick-start/quick-start-windows-10). Your Box will restart after that command...

Therefor try this:

```
- win_shell: Enable-WindowsOptionalFeature -Online -FeatureName containers -All
```

####### Install Hyper-V

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```


## Craft a Windows-ready ansible playbook

I did that step already for you :) So let´s run our the playbook restexample-windows.yml:

```
ansible-playbook -i hostsfile restexample-windows.yml --extra-vars "spring_boot_app_jar=../restexamples/target/restexamples-0.0.1-SNAPSHOT.jar spring_boot_app_name=restexample-springboot host=restexample-windows-dev"
```



## Best practices

