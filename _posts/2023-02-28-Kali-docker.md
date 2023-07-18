---
title: Kali Linux no Docker com Xfce
author: pa3r1ck
date: 2023-02-28 23:10:00 +0800
categories: [Kali-Linux,Docker]
tags: [kali-linux,docker,xfce]
pin: false
---

![image](https://www.kali.org/blog/official-kali-linux-docker-images/images/kali-linux-docker-images.jpg)

## Instalação do Kali Linux com interface gráfica usando o Docker

Usar o docker para correr o Kali Linux pode parecer descabido á partida mas tem as suas vantagens. 
Uma das pricipais é a rapidez do docker em relação a máquinas virtuais.

Tomemos como exemplo o seguinte senário, necessitamos de uma máquina Kali para usar uma ferramenta específica como por exemplo o nmap.

Vejamos como é simples usar o docker para esse fim. Para tal só necessitamos de uns segundos e quase nenhum espaço de armazenamento na nossa máquina anfitriã. Vamos então para um exemplo prático.
Obtemos a imagem oficial do Kali-Linux com o seguinte comando.
``` bash
docker pull kalilinux/kali-rolling
```
Em poucos segundos temos uma imagem base do Kali-Linux que por defeito não vem com nenhuma ferramenta instalada. Agora veremos como utilizar essa mesma imagem.

Listamos as imagens disponivéis na nossa máquina da seguinte forma.
``` bash
docker images
```
De seguida criamos um contentor com a imagem base do Kali-Linux.
``` bash
docker run -it kalilinux/kali-rolling
```
Isto irá gerar uma imagem base do Kali-linux e o parâmetro -it é para podermos ter interação com o contentor que acabamos de criar. Ou seja somos imediatamente e após a criação do contentor para o terminal do kali-linux onde podemos instalar as nossas ferramentas, neste caso o nmap.
Vamos então proceder á instalação.
``` bash
apt update && apt install nmap
```
Em poucos segundos temos a nossa máquina Kali-Linux a rodar dentro do docker e o nmap instalado.
Podemos instalar todo o tipo de ferramentas como por exemplo
``` bash
apt install kali-tools-top10
```
O único problema é que quando saimos desse mesmo terminal o contentor que acabamos de criar desaparece e voltamos á imagem base. Para contronar esse obstáculo podemos fazer um "commit" do contentor anteriormente criado. Para isso é só abrir outra janela no terminal e digitar o seguite.
``` bash
docker ps 
```
Para listar os contentores disponivéis e em seguida,
``` bash
docker commit " ID do Contentor" "e o nome que queremos para a mnova imagem"
```
Exemplo;
``` bash
docker commit 0c448678d82d novaimagem
```
Assim geramos uma imagem já com as ferramentas que necessitamos para que da próxima vez já não tenhamos que partir da imagem base. Agora se desejarmos usar essa imagemé só digitar o seguinte.
``` bash
docker run -it novaimagem
```

## Instalar o ambiente gráfico xfce e fazer acesso através do protocolo RDP

Criamos o nosso contentor,instalamos a interface gráfica xfce, algumas ferramentas e defeniremos uma senha para o utilizador root.
``` bash
docker run -it -p 3389:3389 --name kali-Teste kalilinux/kali-rolling
```
O parâmetro -it é para interagir com o contentor, o parâmetro -p é a porta que iremos utilizar, o parâmetro --name é o nome que queremos dar a este contentor e neste caso (kalilinux/kali-rolling) é a imagem base que iremos utilizar neste caso a imagem base.

Instalamos a interface gráfica xfce e o pacote (xrdp) que nos irá permitir fazer o acesso remoto a este contentor.
``` bash
apt install kali-desktop-xfce xrdp 
```
Defenimos uma senha para o utilizador root.
``` bash
passwd root
```
E iniciamos o serviço (xrdp).
``` bash
service xrdp start
```
Agora com um cliente RDP como o (Remmina) é só conectar ao contentor usando a seguinte informação.
``` bash
localhost:3389
Utilizador= root
Password a que defenimos.
```
E voilá temos o nosso Kali-Linux com o xfce a rodar perfeitamente no docker. Agora é so fazer um "commit" do contentor e temos uma imagem sempre pronta com interface gráfica.




