---
layout: post
title:  "The Page Cache and Page Writeback"
date:   2020-06-22 11:00:00 +0700
categories: [linux]
---

# Métodos de _Caching_

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

### Operações _address-space_

Pode ser visto em `<linux/fs.h>` o `struct address_space_operations` que define a tabela de
operações. Estas funções apontam para as funções que implementam os I/O das páginas para
este objeto em _cache_.
Cada _backing store_ descreve como ele interage com o _page cache_ via seu próprio
`address_space_operations`.
Por exemplo, o ext3 define suas operações em `fs/ext3/inode.c`. Assim, estas são as funções
que gerenciam o _page cache_, incluindo o mais comum: lendo páginas para dentro do _page cache_
e atualizando os dados no _page cache_. Assim, os métodos `readpage()` e `writepage()` são
os mais importantes.

Para os passos de leitura, o kernel primeiro tenta encontrar os dados no _page cache_ chamando
o `find_get_page(mapping, index)`; onde o _mapping_ é o _address-space_ e o _index_ é o _offset_
dentro do arquivo, em páginas.

Para operações de escrita, o método é um pouco diferente. Para mapeamento de arquivos, sempre
que uma página é modificada, o VM simplesmente chama `setPageDirty(page)`.

O kernel posteriormente escreve as páginas via `writepage()`. Operações de escrita em arquivos
específicos são mais complicados. O caminho generico de escrita em `mm/filemap.c` realiza
os seguintes passos:

```c
page = __grab_cache_page(mapping, index, &cached_page, &lru_vec);
status = a_ops->prepare_write(file, page, offset, offset+bytes);
page_fault = filemap_copy_from_user(page, offset, buf, bytes);
status = a_ops->commit_write(file, page, offset, offset+bytes);
```

Primeiro, o _page cache_ é buscado pela página desejada. Se não está na _cache_, uma entrada
é alocada e adicionada. Depois, o kernel configura a solicitação de escrita e os dados são
copiados do _user-space_ para dentro do _buffer_ do kernel. Finalmente, os dados são escritos
no disco.

## O _Buffer Cache_

Blocos de disco individuais também são amarrados ao _page cache_ por meio dos _block I/O buffers_.
Lembre-se que um _buffer_ é a representação _in-memory_ de um único bloco físico do disco.
_Buffers_ agem como descritores que mapeiam páginas em memoria para blocos do disco.

## As _Flusher Threads_

_Dirty page writeback_ acontece em três situações:

- Quando a memória livre cai abaixo de um específico valor;
- Quando _dirty data_ cresce acima de um tempo específico;
- Quando um processo de usuário invoca o `fsync()`.

Quando a memória livre cai em um certo limite, o kernel invoca o `wakeup_flusher_threads()`
para acordar as threads de flush que irão rodar o `bdi_writeback_all()`.

Durante o boot do sistema, um temporizador é inicializado para acordar as threads de flush
que são rodadas via `wb_writeback()`. Esta função então escreve todos os dados que foram
modificados mais antigos que `dirty_expire_interval` milisegundos atrás.
s
O código do _flusher_ reside no `mm/page-writeback.c` e `mm/backing-dev.c` e o mecanismo
de _writeback_ reside em `fs/fs-writeback.c`.

Antes do kernel 2.6, o trabalho das threads de _flush_ eram feitos por duas outras threads,
_bdflush_ e _kupdated_.
# Referências

[1] Linux Kernel Development - a thorough guide to the design and implementation
of the Linux kernel  (third edition)
