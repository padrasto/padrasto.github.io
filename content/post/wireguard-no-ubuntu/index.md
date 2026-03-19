---
title: Instalação do Wireguard num servidor Ubuntu
author: pa3r1ck
date: 2023-02-27 20:10:00 +0800
categories: [Ubuntu]
tags: [wireguard]
pin: false
---

![image](https://upload.wikimedia.org/wikipedia/commons/9/98/Logo_of_WireGuard.svg)


## Este artigo trata da criação de uma VPN com o protocolo Wireguard.

Para tal é necessário tem um servidor neste caso iremos utilizar um servidor Ubuntu.

Para facilitar a instalação vamos utilizar um script que automatiza todo o processo. Esse scipt está disponivél no Github e todos os créditos vão para o seu desenvolvedor. 

Todo este processo em vai ter lugar no nosso servidor em primeiro lugar e logo em seguida configuramos o cliente.

Em primeiro lugar clonamos o script, alteramos as permições e executamos o mesmo.
``` bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh

chmod +x wireguard-install.sh

./wireguard-install.sh
```

No final de executados os comandos a cima é só seguir os passos que são bastante intuitivos. No final é gerado um QR code com toda a informação necessária para configurar o cliente.

## Cofiguração do cliente 

Neste caso vamos utilizar o Linux Ubuntu como cliente mas estes passos podem ser facilmente replicados em qualquer distribuição Linux.

``` bash
sudo apt install wireguard 
```

De seguida criamos um arquivo de texto com o editor "nano" e inserimos a informação obtida através do QR code.

``` bash
sudo nano /etc/wireguard/wg0.conf 
```

Fica aqui um exemplo de como fica o arquivo no final de inserir as informações geradas.

``` bash
[Interface]
PrivateKey = ICkL1OUeq2UOXd9FLlgHxOesvsN+WXxb8/BrNBgSllg=
Address = 10.66.66.2/32,fd42:42:42::2/128
DNS = 94.140.14.14,94.140.15.15

[Peer]
PublicKey = 8o7MZzWVM4jBkcJYSFpbBEdd7bLhWDCOwXMr0K7vVCQ=
PresharedKey = fTO+tOawy912esxEW1zNYrcgLpYvuIqj17UPI+GOyb0=
Endpoint = 139.162.208.222:65513
AllowedIPs = 0.0.0.0/0,::/0
```

Salvamos o aquivo de texto e só nos resta establecer a ligação ao nosso servidor VPN. Para iniciar o serviço basta executar o seguinte comando na nossa máquina cliente.
``` bash
sudo wg-quick up wg0
```

Para a ligação VPN começar na inicialização do sistema o comando é o seguinte.
``` bash
sudo systemctl enable wg-quick@wg0
```
E voilá temos o nosso próprio servidor VPN de maneira bastante simples.


