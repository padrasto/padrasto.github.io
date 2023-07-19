---
title: Instalação do Proxmox num Laptop
author: pa3r1ck
date: 2023-07-19 20:10:00 +0800
categories: [Proxmox]
tags: [Proxmox]
pin: false
---


![image](https://smarthomescene.com/wp-content/uploads/2023/03/proxmox-home-assistant-smarthomescene.jpg.webp)


## Breve resumo 

O Proxmox é uma plataforma de virtualização de código aberto que combina virtualização de servidores e gerenciamento de contêineres em uma única solução poderosa. Ele é projetado para facilitar a criação, implantação e gerenciamento de máquinas virtuais (VMs) e contêineres em um ambiente de data center.

Quando instalado num Laptop, o Proxmox vai fazer o particionamento do disco de uma forma que vai deixar muito pouco espaço que realmente possa ser usado. 

Por esse motivo vamos precisar fazer umas magias a fim de recuperar esse espaço para as nossas Vms.

Primeiro temos de reclamar o espaço apagando o volume "local-lvm".

De seguida abrimos o terminal do Proxmox e digitamos os três seguintes comandos.


```bash 
 lvremove /dev/pve/data
```

 
```bash 
 lvresize -l +100%FREE /dev/pve/root
``` 

```bash 
 resize2fs /dev/mapper/pve-root
```

Com o procedimento a cima completo teremos acesso a todo o espaço em disco para as nossas Vms.

Como iremos utilizar um Laptop, vamos querer fechar a tampa e desligar o ecrã sem que o nosso servidor se desligue.
Para isso teremos que fazer alguns ajustes no nosso Proxmox.

Mais uma vez recorrendo ao terminal do mesmo para editar o seguinte arquivo;


```bash 
 nano /etc/systemd/logind.conf
```

No final o resultado que queremos é este;

```bash 
 [Login]
#NAutoVTs=6
#ReserveVT=6
#KillUserProcesses=no
#KillOnlyUsers=
#KillExcludeUsers=root
#InhibitDelayMaxSec=5
#UserStopDelaySec=10
#HandlePowerKey=poweroff
#HandlePowerKeyLongPress=ignore
#HandleRebootKey=reboot
#HandleRebootKeyLongPress=poweroff
#HandleSuspendKey=suspend
#HandleSuspendKeyLongPress=hibernate
#HandleHibernateKey=hibernate
#HandleHibernateKeyLongPress=ignore
HandleLidSwitch=ignore
#HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore
#PowerKeyIgnoreInhibited=no
#SuspendKeyIgnoreInhibited=no
#HibernateKeyIgnoreInhibited=no
#LidSwitchIgnoreInhibited=yes
#RebootKeyIgnoreInhibited=no
```

Reiniciamos o serviço para as mudanças terem efeito;

```bash 
 systemctl restart systemd-logind.service
```


Agora iremos defenir um tempo para o ecrã desligar, editando o arquivo;

```bash 
 nano /etc/default/grub
```
Editamos a linha 

```bash
GRUB_CMDLINE_LINUX=""
```
Para que fique assim;

```bash 
GRUB_CMDLINE_LINUX="consoleblank=300"
```
Por fim atualizamos o arquivo

```bash
sudo apt update-grub
```


Agora temos o nosso servidor totalmente pronto para uso.
