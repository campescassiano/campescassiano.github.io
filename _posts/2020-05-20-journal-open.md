---
layout: post
title:  Entendendo o debugfs journal-open
date:   2020-05-20 21:00:00 +0700
categories: [linux]
---

Vamos aqui estudar um pouco o `debugfs` com o modo `journal_open`.

Pode acontecer e precisarmos habilitar o `checksum` do `journal`, e para isso
podemos contar com o auxílio do `debugfs`.

Abaixo exemplificamos como fazemos para fazer o debug no journal:

```c
debugfs journal_open [-c] [-v ver] [-f ext_jnl]
```

Com isso, podemos alterar dinâmicamente se o `journal` deve executar
com o `checksum` ativado, sua versão, bem como se vamos carregar ele de
um armazenamento separado, via `-f`.

No código, podemos verificar isso sendo avaliado no arquivo `e2fsprogs/debugfs/do_journal.c`
na função `do_journal_open`. Lá teremos o `while` que verifica cada
uma das opções como `-f -v` etc.

Por sua vez, caso seja habilitado o `checksum`, será então chamado a função
`jfs_set_feature_##`.

