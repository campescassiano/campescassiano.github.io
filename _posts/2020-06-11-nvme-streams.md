---
layout: post
title:  "NVMe streams usage"
date:   2020-06-11 11:00:00 +0700
categories: [linux]
---

Aqui vou falar um pouco do uso dos `multi-streams` de um dado SSD NVMe.

Basicamente temos dois módulos que controlam toda a parte dos drivers nvme, que são:

1. `nvme`
2. `nvme_core`

Por padrão, eles não se inicializam com os streams habilitados.
Vamos ver como está a diretiva para streams em nosso SSD:

```sh
sudo nvme dir-receive /dev/nvme0 --dir-type 0 --dir-oper 1 --human-readable
dir-receive: type 0, operation 0x1, spec 0, nsid 0x1, result 0
	Directive support
		Identify Directive  : not supported
		Stream Directive    : not supported
	Directive status
		Identify Directive  : disabled
		Stream Directive    : disabled
```
Neste [link](https://manpages.ubuntu.com/manpages/cosmic/man1/nvme-dir-receive.1.html) temos
um _manpage_ com detalhes dos comandos suportados no `nvme-cli` para diretivas de streams.

Vemos que o `Stream Directive` está desabilitado, portanto vamos habilitá-lo:

```sh
sudo nvme dir-send /dev/nvme0n1 --dir-type 0 --dir-oper 1 --target-dir 1 --endir 1
```

Outro detalhe que eu precisei fazer foi remover os módulos nvme para subir eles com
os streams habilitados:

```sh
sudo rmmod nvme && sudo rmmod nvme_core
```

E então reinstalar os módulos, passando o parâmetro que habilita os streams no driver:
```sh
cd linux-x.x.x/drivers/nvme/host
sudo insmod nvme-core.ko streams=1
sudo insmod nvme.ko
```

Agora sim temos o driver carregado com os streams ativado.
