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

# Referências

[1] Linux Kernel Development - a thorough guide to the design and implementation
of the Linux kernel  (third edition)
