---
layout: post
title:  "Writeback no Sistema de Arquivos"
date:   2020-05-21 11:00:00 +0700
categories: [linux]
---

Neste post vamos falar um pouco sobre o `writeback` no kernel Linux.
O `writeback` possui a função de escrever os `inodes` que estão sujos
no sistema de arquivos de maneira periódica a fim de persistir estes
dados na mídia não volátil (HDD, SSD, etc).

O ponto de entrada para o entendimento é na função `wb_workfn()` que é
cadastrada no `bdi` para ser a função chamada periodicamente para realizar
o `writeback`. Isso é feito então dentro da função `wb_init` que por
sua vez irá inicializar o trabalho via `INIT_DELAYED_WORK`, passando
então o `wb_workfn` como parâmetro.

Partindo então da `wb_workfn`, ela chama então a função `wb_do_writeback`
enquanto houver trabalho a ser feito (`while (!list_empty(&wb->work_list))`).
Nisso, será chamado então o `wb_writeback` enquanto houver trabalho para ser feito.

Neste ponto, temos dois possíveis caminhos:

1 - `writeback_sb_inodes()`
2 - `__writeback_inodes_wb()`

O passo `2` vai convergir no `1`, então vamos falar do `1`.

Seguindo pelo `1`, será chamado o `writeback_sb_inodes()` onde
esta função terá o _loop_ `while (!list_empty(&wb->b_io))`.

Será chamado o `writeback_chunk_size`, e posteriormente o `__writeback_single_inode`.
Dentro diso, será chamado o `do_writepages` que está encarregado de
chamar a função correspondente do `a_ops->writepages`.

Após isso, é feito a checagem se o `wb == WB_SYNC_ALL` que vai chamar o
`filemap_fdatawait()` que chama o `filemap_fdatawait_range()`, que vai
chamar o `__filemap_fdatawait_range()`. Em um momento será chamado o
`write_inode` que vai chmar o `s_op->write_inode`

TODO:: continuar a partir do `do_writepages()`

writeback_sb_inodes
