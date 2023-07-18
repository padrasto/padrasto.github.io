---
title: Configuração inicial de um servidor Ubuntu
author: pa3r1ck
date: 2023-02-26 21:10:00 +0800
categories: [ssh, Ubuntu]
tags: [ssh]
pin: false
---


## Com autenticação de chave pública 


![image](https://images.freeimages.com/fic/images/icons/2796/metro_uinvert_dock/320/os_ubuntu_mirror.png)

A segurança de um servidor está entre as primeiras coisas que necessitamos de fazer para que as aplicações que neles utilizamos estejam assentes numa base segura.

Para tal uma medida extremamente imprtante é desablitar a autenticação por password assim como a criação de utilizador que possa administrar o servidor mas sem ser root.

Para tal vamos primeiro fazer login no nosso servidor como root.
``` bash
ssh root@ip do servidor
```
Criamos um utilizador neste caso chamado "user" com o seguite comando
``` bash
adduser user
```
De seguida vamos conceder previlégios administrativos a este utilizador adicionando-o ao grupo "sudo".
``` bash
usermod -aG sudo user
```
A próxima etapa para proteger seu servidor é configurar a autenticação de chave pública para seu novo utilizador. A configuração aumentará a segurança do seu servidor, exigindo uma chave SSH privada para fazer o login.


Se você ainda não tiver um par de chaves SSH, que consiste em uma chave pública e uma privada, será necessário gerar uma. Se você já tem uma chave que deseja usar, pule para a etapa Copiar a chave pública.

Para gerar um novo par de chaves, digite o seguinte comando no terminal de sua máquina local (ou seja, seu computador e pressione enter.
``` bash
ssh-keygen
```

Com o comando anterior gerámos duas chaves criptográficas, uma chave privada e uma chave pública que vamos copiar para o nosso servidor.

Para imprimir a chave pública na máquina local é necessário executar o seguinte comando.
``` bash
cat ~/.ssh/id_rsa.pub
``` 
Selecione a chave pública e copie-a para a área de transferência.
Para habilitar o uso da chave SSH para autenticação como o novo utilizador remoto, você deve adicionar a chave pública a um arquivo especial no diretório inicial do utilizador.
No servidor e como utilizador root  digite o seguinte comando para alternar temporariamente para o utilizador antreormente criado.
``` bash
su - user
```
Agora que estamos no diretório do novo utilizador criamos um diretório chamado .ssh que é um dirétorio oculto e mudamos as premissões do mesmo.
``` bash
mkdir ~/.ssh
```
``` bash
chmod 700 ~/.ssh
```
Agora dentro do diretório .ssh criamos um arquivo chamado "authorized_keys" com o editor de texto "nano".
``` bash
nano ~/.ssh/authorized_keys
```
Salvamos o arquivo e alteramos as permições de forma a garantir a integridade do mesmo.
``` bash
chmod 600 ~/.ssh/authorized_keys
```
Depois de todas as alterações feitas voltamos ao utilizador "root".
``` bash
exit
```
Precisamos de editar mais um arquivo para concluir com sucesso a nossa implementação de segurança. Utilizaremos o "nano" para isso.
``` bash
nano /etc/ssh/sshd_config
```
Localizamos a linha que diz "PasswordAuthentication" descomentamos o símbolo #, e no final o resultado tem de ser este.
``` bash
PasswordAuthentication no
```

Aqui estão duas outras configurações que são importantes para autenticação apenas de chave e são definidas por padrão, mas se por algum motivo assim não for devem ficar exatamente assim.
``` bash
PubkeyAuthentication yes
ChallengeResponseAuthentication no
``` 
Findado este processo salvamos o arquivo e reeniciamos o serviço ssh para que as mudanças possam sortir efeito.
``` bash
systemctl reload sshd
```
Assim concluimos a nossa configuração inicial de segurança e passaremos apenas a usar chave ssh para nos ligarmos ao servidor que em termos de segurança é muito mais eficaz do que o método de password.

