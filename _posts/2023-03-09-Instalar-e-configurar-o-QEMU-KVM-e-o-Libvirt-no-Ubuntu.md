---
title: Instalação do KVM no Linux Ubuntu
author: pa3r1ck
date: 2023-03-09 21:10:00 +0800
categories: [kvm, Ubuntu]
tags: [kvm]
pin: false
---


![image](https://upload.wikimedia.org/wikipedia/commons/thumb/4/45/Qemu_logo.svg/512px-Qemu_logo.svg.png?20210415043304)


## Instalar e configurar o QEMU-KVM e o Libvirt no Ubuntu

O QEMU-KVM e o libvirt são duas ferramentas importantes para virtualização no Linux. O QEMU-KVM é uma plataforma de virtualização de código aberto que permite emular várias arquiteturas de CPU, incluindo Intel e ARM, e executar sistemas operacionais convidados em máquinas virtuais. O libvirt é uma API de gerenciamento de virtualização que fornece uma camada abstrata para interagir com diferentes hypervisors, incluindo QEMU-KVM. Juntos, eles fornecem uma solução robusta e escalável para virtualização em ambientes Linux. Neste tutorial, vamos explorar como instalar e configurar o QEMU-KVM e o libvirt no Ubuntu.

Instalação do software necessário
``` bash
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils libguestfs-tools genisoimage virtinst libosinfo-bin virt-manager dnsmasq
```
Adicionar o utilizador aos Grupos KVM
``` bash
sudo adduser $USER libvirt
```

``` bash
sudo adduser $USER libvirt-qemu
```
Editrar as permições de forma a usar o software com o nosso utilizador sem previlégios de root.
``` bash
sudo addgroup "$(whoami)" libvirt
```
``` bash
sudo addgroup "$(whoami)" kvm
```

Agora é necessário reenicar o nosso sistema operativo para que as mudanças surtam efeito.