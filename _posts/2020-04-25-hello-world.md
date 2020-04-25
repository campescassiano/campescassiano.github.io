---
layout: post
title:  "Hello, World!"
date:   2020-04-25 10:17:00 +0700
categories: [linux]
---

Vamos fazer isso, mas de uma maneira um pouco mais doida, em `assembly`.

```nasm
section	.text
global	_start

_start:
	mov	edx, len
	mov	ecx, msg
	mov	ebx, 1
	mov	eax, 4
	int	0x80

	mov	ebx, 0
	mov	eax, 1
	int	0x80

section	.data

msg	db	'Hello, world!',0xa
len	equ $ - msg
```

O equivalente em `C` seriam poucas linhas, para ser mais exato, apenas cinco:
```c
int main()
{
	printf("Hello, world!\n");
	return 0;
}
```

Mas como que este simples código em `C` se transforma em tantas instruções em `assembly`?
Vamos verificar um pouco melhor que está acontecendo aqui!

Sabemos que as chamadas de sistema estão por trás de tudo que acontece com os nossos programas
desenvolvidos, e o `printf` não é diferente. Ele no final irá realizar uma `systemcall` para
escrever na tela do seu computador, esta `systemcall` é a `sys_write` onde sua assinatura
pode ser encontrada aqui: `linux/include/linux/syscalls.h`.

Vejamos como é a assinatura da `sys_write`:

```c
asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count);
```
Vejamo só, são três parâmetros, o `fd` ao qual queremos escrever, que no
nosso caso é o `stdout` que nada mais é que o `fd == 1`. O próximo parâmetro
é um ponteiro vindo do  `user space` que contém a mensagem a ser impressa,
este ponteiro é o `*buf`. Por fim, o tamanho desta mensagem em `count`.

Bem simples, não? O `printf` abstraiu algumas partes para nós, como o `fd`
e o `count`, mas agora sabemos que eles estão lá por debaixo dos panos.

Mas tem uma coisa faltando, a chamada da `syscall`. Cada `syscall` possui
um número específico que a identifica. Para vermos quais são os números
para cada uma delas, vamos em: `linux/arch/arm64/include/asm/unistd32.h`
por exemplo, onde teremos as definições dos números para a arquitetura
ARM.

Lá, veremos que o `sys_write` possui atribuído o número `4`.

Agora nosso código `assembly` começa a fazer sentido, fica fácil identificar
as instruções que vão realizar a nossa chamada de função:

```nasm
	mov	edx, len
	mov	ecx, msg
	mov	ebx, 1
	mov	eax, 4
	int	0x80
```

Estamos escrevendo nos registradores `[edx]`, `[ecx]`, `[ebx]` e `[eax]`
justamente o que acabamos de falar, o tamanho da `string`, o ponteiro,
o `fd` que iremos escrever, e qual `syscall` iremos utilizar.

Por fim, é chamada a `int 0x80` que gera a interrupção. Lindo não?
Temos então nosso `printf` traduzido para `assembly`.

Por fim, é a vez do nosso `return 0`, que nada mais é do que a chamada
da `syscall` de número `1`, a `sys_exit`, onde seu único parâmetro,
se olharmos lá no arquivo das `syscalls`, é o `error_code`:

```c
asmlinkage long sys_exit(int error_code);
```

Por isso nossa última parte do código, onde estamos colocando o nosso
código de retorno no registrador `[eax]` contendo a `syscall` de número
`1`, e no registrador `[ebx]` estamos colocando o código de retorno.

```nasm
	mov	ebx, 0
	mov	eax, 1
	int	0x80

Este então é o fim de nosso programa. E o início de um `Hello, world!`.

