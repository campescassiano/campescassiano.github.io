---
layout: post
title:  "Configurando o journal mode"
date:   2020-06-22 11:00:00 +0700
categories: [linux]
---

Bom, seguidamente eu preciso alterar o _journal mode_ no meu sistema de arquivos para
fins de teste, e eu nunca me lembro de primeira como é feito isso.

Sendo assim, existem algumas ferramentas que são interessantes para ver como o
sistema de arquivos está montado, e vou listar aqui.

O primeiro e principal deles que eu gosto é o `dumpe2fs`.

Ele vai mostrar informações do sistema de arquivos, os _group descriptors_, e outra
porrada de coisas que tu não precisa saber. Eu diria que é bem completo.

Aqui vai um trechinho do meu `dumpe2fs`:

```sh
sudo dumpe2fs -h /dev/nvme0n1
[...]

Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Tue Feb 26 23:46:51 2019
Last mount time:          Tue Jun 23 16:14:35 2020
Last write time:          Tue Jun 23 22:20:36 2020
Mount count:              339

[...]
```

Como vocês podem ver, este sistema de arquivos foi criado em 26 de Fevereiro de 2019,
ou seja, faz um ano e meio quase que estou sem formatar meu PC (hahaha).

Outra ferramenta que faz algo muito similar é o `tune2fs`, porém, com ele é
possível alterar algumas coisas em tempo real no sistema de arquivos, como
o que este artigo diz: **mudar o modo de journal** do _JBD2_.

Aqui vai um trecho da saída do `tune2fs`:

```sh
sudo tune2fs -l /dev/nvme0n1

tune2fs 1.44.1 (24-Mar-2018)
Filesystem volume name:   <none>
Last mounted on:          /
Filesystem UUID:          e092829b-8279-4910-8a08-16036bdcd2ca
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Filesystem flags:         signed_directory_hash
Default mount options:    journal_data_ordered user_xattr acl
```

Ele também mostra algumas _features_ do sistema de arquivos, e o que estamos
interessados aqui é na linha: **Default mount options** acima.
Como vocês podem ver, ela está indicando **journal_data_ordered**, o que
significa que estamos utilizando o journal em modo ordenado, isto é,
os dados são escritos primeiro na sua posição final, enquanto os metadados
são escritos no journal, e posteriormente são gravados na posição final
via o chamado _checkpoint_.

Para alterar o modo de journal, é bastante simples. A gente irá utilizar
o `tune2fs` para fazer isso, abaixo um exemplo alterando para o modo
`journal_data` o meu SSD NVMe:

```sh
sudo tune2fs -o journal_data /dev/nvme0n1
```

O modo `journal_data` irá escrever os dados e os metadados no journal,
e posteriormente eles serão gravados no seu destino final. Este modo
é considerado o mais tolerante a falhas, porém ele acaba ocasionando
um maior número de escritas no dispositivo.

Por padrão, um sistema de arquivos `ext4` vai montar o journal no modo
`journal_data_ordered`, pois é o modo que garante um certo grau de
tolerância a falhas, sem impactar demais na performance.

Se quiser desativar o journal, então pode-se fazer o seguinte:

```sh
tune2fs -O ^has_journal /dev/sda1
```

E por fim, para verificar se tudo deu certinho, execute:

```sh
sudo e2fsck -f /dev/nvme0n1
```
