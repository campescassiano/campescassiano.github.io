---
layout: post
title:  "The Page Cacha and Page Writeback"
date:   2020-06-22 11:00:00 +0700
categories: [linux]
---

#Métodos de _Caching_

O _page cache_ consiste em páginas físicas em RAM, onde seu conteúdo corresponde aos
blocos físicos do disco. O tamanho do _page cache_ é dinâmico de acordo com a demanda
de uso de memória do sistema.

Chamamos o dispositivo de armazenamento sendo _cached_ o _backing storage_.

Sempre que o kernel começa uma operação de leitura, ele primeiro verifica se os dados
estão no _page cache_. Se sim, o kernel não precisa acessar o disco para realizar a
leitura, bastando ler da RAM. Isso é chamado de _cache hit_. Se os dados não estão
no _page cache_, é chamado de _cache miss_, e então o kernel deve executar uma operação
de I/O para ler o dado do disco. E então os dados são mantidos no _page cache_ para futuro
acesso, tirando vantagem do _temporal locality_.

Arquivos inteiros não precisam estar na _cache_; o _page cache_ pode manter arquivos
inteiros ou não, dependendo da necessidade.

## _Caching_ de escrita

Basicamente existem três formas de realizar _caching_ para escrita:
(1) _no write_, o _cache_ simplesmente não realiza o _cache_ das operações de escrita, ou seja,
uma escrita ao disco será necessária, invalidando o _page cache_ dos dados, tornando necessário
novamente a leitura do disco para atualizar o _page cache_;
(2) _write through_, uma operação de escrita será aplicada tanto no _page cache_ quanto no disco.
Assim o disco e o _page cache_ se mantém atualizados. Assim, a _cache_ se mantém coerente sem
precisar invalidá-la, causando a necessidade de leitura do disco.
(3) _write back_, utilizada no Linux. As operações de escrita são realizadas diretamente no
_page cache_. O _backing store_ não é atualizado imediatamente. Ao invés disso, as páginas
escritas no _page cache_ são marcadas como _dirty_ e são adicionados em uma _dirty list_.
Periodicamente, páginas no _dirty list_ são escritas no dispositivo por um processo chamado
_writeback_, trazendo o disco em sincronismo com o _page cache_. Assim, as páginas são marcadas
como limpas, para indicar que estão em sincronia com o disco.

## _Cache eviction_

Pode acontecer a necessidade de limpar blocos de memória que estão sendo utilizados pelo _page cache_
e para isso, é necessário a _eviction_ de blocos que não estão mais sendo utilizados, e que estão
limpos. Se não tiver páginas limpas o suficiente, o Kernel irá realizar operações de escrita via
_writeback_ para que as páginas possam ser _evicted_.

A forma adotada pelo Kernel para lidar com isso é o _Least Recently Used_ que é a lista mantendo
as páginas ordenadas de acordo com seu tempo de acesso. As páginas que estão a mais tempo sem
serem acessadas serão _evicted_.

## O Linux _page cache_

### O objeto _address-space_

Uma página no _page cache_ pode consistir de múltiplos blocos não contíguos. Desta forma, não é
possível de indexar os dados no _page cache_ utilizando apenas o nome do dispositivo e o
número do bloco, que de certa forma seria a  maneira mais simples.

O Kernel para manter uma _page cache_ genérica - que não fosse amarrada aos arquivos físicos ou
a estrutura de _inode_ - o _page cache_ do Linux utiliza um novo objeto para gerenciar as entradas
no _cache_ e as operações de I/O em páginas, o _address-space_. Pense no _address-space_ como um
análogo ao _vm-area-struct_ introduzido no capítulo 15 do [1](# Referências).

Como muitas outras coisas no kernel, o _address-space_ tem seu nome ambíguo. O correto poderia
ser _page-cache-entity_ ou _physical-pages-of-a-file_.

O _address-space_ é definido em `<linux/fs.h>`.

O _i-mmap_ é uma árvore de busca prioritária de todos os mapeamentos privados e compartilhados
neste _address-space_. Relembre que enquanto um arquivo _cached_ é associado com uma estrutura
_address-space_, ele pode ter vários _vm-area-struct_ - como um mapeamento um-para-muitos
das páginas físicas para várias páginas virtuais. O _i-mmap_ possibilita o kernel a eficientemente
encontrar mapeamentos associados com este arquivo em _cache_.

Existem ao total _nrpages_ no _address-space_.

O _address-space_ é associado com algum objeto do kernel. Normalmente isso é um _inode_.
Se sim, o campo _host_ aponta para o _inode_ associado. O campo _host_ é _NULL_ se o objeto
associado não é um inode - por exemplo, se é associado com um _swapper_.


# Referências

[1] Linux Kernel Development - a thorough guide to the design and implementation
of the Linux kernel  (third edition)
