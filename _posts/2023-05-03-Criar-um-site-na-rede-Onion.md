---
title: Criar um Site na Rede Onion
author: pa3r1ck
date: 2023-05-03 21:10:00 +0800
categories: [debian, tor]
tags: [tor]
pin: false
---
![image](https://blog.torproject.org/how-we-plant-and-grow-new-onions/lead.webp)


# Criar um Site na Rede Onion com um servidor Debian.

 Muitos de nós gostamos de privacidade na navegação que efetuamos todos os dias assim como poder expressar as nossas ideias de forma anónima e privada.

Muitos de nós temos sites, blogues onde expressamos o que bem entendemos,mas e se quisermos expressar essas ideias sem ter de partilhar nenhuma informação pessoal? 

Ou até falar de assuntos que não são concensuais na sociedade e podermos expressar a nossa opinião da forma que bem entendermos. 

Já para não falar em países onde a censura impera e é crucial manter o anonimato para evitar problemas.

Qual quer que seja o caso 
a rede TOR proporciona esse tipo de anonimato a custo zero. Embora seja extremamente importante contribuir tanto financeiramente como na própria extrutura da rede.

Disto isto a proposta deste post é criar um site básico na rede TOR  a fim de podermos executar o nosso site de forma totalmente anonima. Para tal teremos de seguir os seguintes passos.

Após a instalação de um servidor Debian temos de isntalar alguns programas para que tudo funcione e o nosso objetivo seja cumprido.

1º Instalação de um servidor apache.

```bash
sudo apt installl apache2
```
2º Iniciar o servidor apache.

```bash
sudo systemctl start apache2
```
3º Em seguida colocar o servidor a iniciar com o sistema.

```bash
sudo systemctl enable apache2
```

4º Criar um local no nosso servidor para alojar o site.

```bash
sudo mkdir /var/www/html
```

5º Dentro desse diretório vamos criar um arquivo chamado index.html onde iremos ecrever o sosso conteúdo neste caso com o editor de texto nano. 

```bash
sudo nano /var/www/html/index.html
```

6º Nesse arquivo escrevemos em html o conteúdo que será exibido.


7º Agora precisamos de intalar o TOR no nosso servidor.

```bash
sudo apt install tor
```
8º Mais uma vez com o editor nano vamos alterar o seguite arquivo.

```bash
sudo nano /etc/tor/torrc
```
9º E no final do mesmo colocamos as seguintes linhas.

```bash
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```

10º Precisamos de reeniciar o serviço para que seja relido o arquivo de configuração.

```bash
sudo systemctl restart tor
```

A partir deste momento o site já está ativo na rede tor. Pra sabermos qual o endereço do nosso site digitamos o seguinte comando.

```bash
cat /var/lib/tor/hidden_service/hostname
```

E desta forma extremamente simples podemos alujar o nosso site na rede TOR usando um servidor Debian.

