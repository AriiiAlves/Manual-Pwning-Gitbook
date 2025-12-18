# 2. Assembly

## 2.1. O que é Assembly?

Assembly ou linguagem de montagem é uma **linguagem de programação**, assim como C, Python, JavaScript, etc. Porém se destaca por ser a linguagem mais próxima do código que o processador realmente executa. O C abstrai muitas das operações.

Tome como exemplo a operação de declarar uma variável no C: `int a = 10`. Você acha mesmo que o processador apenas recebe isso e já tem um lugar pronto para a variável `a`, uma caixinha onde é guardado 10? Não, muito longe disso. Uma série de operações que o processador realmente executa são escondidas de você, para simplificar o trabalho do programador.

Algumas linguagens de programação abstraem mais do que outras. O Assembly é **muito próximo do código de máquina, muito verboso**. O C já abstrai mais, mas é rígido e permite manipulação de memória. Já o Python seria o "nível mais fácil" das linguagens de programação, pois tem inúmeras funcionalidades prontas que facilitam seu trabalho.

Podemos ver isso com a operação de soma de dois números.

Código de máquina:
```Bin
01001000 11000111 11000111 00000101 00000000 00000000 00000000
01001000 11000111 11000110 00000011 00000000 00000000 00000000
01001000 00000001 11110111
```
Código de máquina em hexadecimal (cada hexadecimal representa 1 byte. Se acostume com isso, pois analisamos informações na memória desse jeito, o famoso `hexdump`):
```hex
48 c7 c7 05 00 00 00 48 c7 c6 03 00 00 00 48 01 f7
```
Código em Assembly:
```Assembly
mov rdi, 0x5
mov rsi, 0x3 
add rdi, rsi
```
Código em C:
```C
int a = 5;
int b = 3;
int resultado = a + b;
```
Código em Python:
```py
resultado = 5 + 3
```

Não há linguagem de programação melhor do que outra, apenas aquela que é uma ferramenta boa para você. Python oferece agilidade, enquanto C oferece otimização e robustez, ao passo que Assembly é próxima do código de máquina, útil para otimização e também análise de Engenharia Reversa, como veremos aqui.

O Assembly possui **acesso direto ao hardware**, de modo que permite manipular diretamente os registradores da CPU (falaremos mais à frente sobre isso). Cada instrução (linha) de um código Assembly geralmente corresponde a uma única instrução de máquina, assim temos uma proporção 1:1, algo que não acontece no C e Python.

## 2.2 Por que Assembly?

Todo código, qualquer que seja a linguagem, é compilado e transformado em código de máquina. Engenharia reversa é a arte de abrir esses binários e entendê-los, e para isso precisamos entender a sintaxe do Assembly, pois ela torna o código de máquina legível. Ou seja, ela traduz 0 e 1 em instruções que podemos entender. O assembly não é tão trivial, mas uma vez que você se acostuma, te ajuda perfeitamente a entender quealquer binário.

Além disso, as vulnerabilidades que são exploradas aqui e em CTF's geralmente necessitam que  exploremos o Assembly e a organização interna do código, para ver como podemos abusar dessa estrutura.

## 2.3 Registradores

Registradores são **locais de memória temporária dentro do processador**. Eles possuem uma **quantidade pequena de memória**, em geral 8 bytes, mas possuem **alta velocidade de acesso e operação** (muito mais rápido que memḿoria RAM).

Basicamente, os registradores são utilizados para armazenar informação útil, como valores de variáveis, endereços de memória para outros lugares (ponteiros), etc. Os registradores sempre estarão presentes em código assembly, pois enviar valores diretamente para registradores no processador é o modo mais otimizado de se realizar uma operação.

<div styles="display: block; text-align:center">
    <img src="./images/registers.png"/>
</div>

Na **arquitetura x64 (64 bits = 8 bytes)**, os registradores da CPU do computador **possuem 8 bytes cada**. Na **arquitetura x86 (32 bits = 4 bytes)**, os registradores possuem **4 bytes cada**. Cada programa **.exe** (ou qualquer outra extensão em código de máquina) segue uma dessas duas arquiteturas.

Abaixo, temos uma lista de registradores e do modo como cada um é usado em um programa.

| 8 Byte Register | Lower 4 Bytes | Lower 2 Bytes | Lower Byte | Uso principal |
|-----------------|---------------|---------------|------------|---------------|
|   rbp           |     ebp       |     bp        |     bpl    |Base Pointer (Ponteiro de Base do Frame da Pilha)|
|   rsp           |     esp       |     sp        |     spl    |Stack Pointer (Ponteiro do Topo da Pilha)
|   rip           |     eip       |     (N/A)          |   (N/A)         |Instruction Pointer (Ponteiro da Próxima Instrução)
|   rax           |     eax       |     ax        |     al     |Acumulador (Valor de Retorno, Operações Aritméticas)
|   rbx           |     ebx       |     bx        |     bl     |Base (Ponteiro de Dados, Valor Preservado em Chamadas)
|   rcx           |     ecx       |     cx        |     cl     |Contador (Contador para Loops/Shifts, 3º Argumento)
|   rdx           |     edx       |     dx        |     dl     |Dados (Extensão do Acumulador, 4º Argumento)
|   rsi           |     esi       |     si        |     sil    |Source Index (Ponteiro Fonte para Operações de String/Memória, 2º Argumento)
|   rdi           |     edi       |     di        |     dil    |Destination Index (Ponteiro Destino para String/Memória, 1º Argumento)
|   r8            |     r8d       |     r8w       |     r8b    |5º Argumento de Função (Uso Geral)
|   r9            |     r9d       |     r9w       |     r9b    |6º Argumento de Função (Uso Geral)
|   r10           |     r10d      |     r10w      |     r10b   |Temporário (Uso Geral, Não Preservado em Chamadas)
|   r11           |     r11d      |     r11w      |     r11b   |Temporário (Uso Geral, Não Preservado em Chamadas)
|   r12           |     r12d      |     r12w      |     r12b   |Local (Uso Geral, Preservado em Chamadas)
|   r13           |     r13d      |     r13w      |     r13b   |Local (Uso Geral, Preservado em Chamadas)
|   r14           |     r14d      |     r14w      |     r14b   |Local (Uso Geral, Preservado em Chamadas)
|   r15           |     r15d      |     r15w      |     r15b   |Local (Uso Geral, Preservado em Chamadas)

## 2.4 Flags

**Existe um registrador que contém flags**, o **RFLAGS** (x64 = 64 bits) ou **EFLAGS** (x86 = 32 bits). Cada bit do registrador é uma flag diferente.

| Bit | Flag|
|-----|-----|
|00   | Carry Flag
|01|     always 1
|02|     Parity Flag
|03|     always 0
|04|     Adjust Flag
|05|     always 0
|06|     Zero Flag
|07|     Sign Flag
|08|     Trap Flag
|09|     Interruption Flag     
|10|     Direction Flag
|11|     Overflow Flag
|12|     I/O Privilege Field lower bit
|13|     I/O Privilege Field higher bit
|14|     Nested Task Flag
|15|     Resume Flag

Esse registrador é usado, por exemplo, para a operação de comparar se dois números são iguais. Para verificar se `a > b`:

-  `cmp a, b` (instrução assembly)
    - Faz `a - b`
    - Se dá zero, **Zero Flag** (bit 06) fica como `1`.
    - Se dá diferente de zero, **Zero Flag** (bit 06) fica como `0`.
        - Se dá negativo, **Sign Flag** (bit 07) fica como `1`
        - Se dá positivo, **Sign Flag** (bit 07) fica como `0`
- Resultado da subtração é descartado, mas as Flags são utilizadas como referência para os próximos passos.

Isso sempre irá ocorrer quando houver um `IF/ELSE`, por exemplo, no código original.

## 2.5 Instruções Assembly comuns

### Words

Words não são comandos, mas indicam o tamanho do tipo de dado em Assembly.

- WORD = **2 bytes de dados**.
- DWORD = **4 bytes de dados**.
- QWORD = **8 bytes de dados**.

### Comentários

Para comentários, usa-se `;`

### mov

**Move dados de um registrador para outro** (segundo para o primeiro). 

Ex: `mov rax, rdx` (move dados de rdx para rax)

### dereference

**colchetes referenciam dados ao qual um ponteiro aponta**. Ou seja, se `RBP-0x8` = `0xfffff72c` (endereço), então `[RBP-0x8]` = 10 (valor da variável).

Ex: `mov rax, [rdx]`

### lea

**Calcula o endereço do segundo operando**, e move para o primeiro.

Ex: `lea rdi, [rbx+0x10]` (move o endereço rbx+0x10 para o registrador rdi)

### add

**Adiciona dois valores** e armazena no primeiro operando.

Ex: `add rax, rdx` (rax = rax + rdx)

### sub

**Subtrai o segundo operando do primeiro** e armazena no primeiro.

Ex: `sub rsp, 0x10` (`rsp` = `rsp` - `0x10`)

### xor

**Faz o xor nos dois argumentos**, e armazena no primeiro operando. and e or são semelhantes.

Ex: `xor rdx, rax`

### push

**Cresce a stack em 8 ou 4 bytes** (para x64, ou 4 para x86), **depois** **adiciona o conteúdo do registrador para o topo da stack**

Ex: `push rax`

### pop

Tira 8 bytes do topo da stack e armazena no argumento. Depois encolhe a stack.

Ex: `pop rax` (Retira dado do topo da Stack e armazena em RAX)

### jmp

**Pula para um endereço de instrução**. Redireciona a execução do código.

Ex: `jmp 0x60201012`

É uma redução de código, e é igual a:

`mov RIP, 0x60201012`

(Alterar o RIP altera onde o código estará sendo executado)

### call

É uma redução de dois comandos. Utilizado sempre que se chama uma função no código.

1. `push RIP` - empilha Return Address na stack. É o endereço de retorno da PRÓXIMA instrução a ser executada na função atual quando a nova função retornar.
2. `jmp function`- Salta para as linhas de instrução da nova função.

### ret

Depois que a função finaliza, **a instrução `ret` é chamada**. Ela consiste em uma redução de comando:

`pop RIP` - Desempilha topo da stack (que deve ser o return address) e coloca no RIP (registrador que guarda o endereço de onde está sendo executado código)

Você pode se perguntar: Mas o topo da stack não tem as variáveis, RBP antigo, etc? Sim, exato. Por isso, antes da função finalizar, temos ajustes na stack:

```assembly
mov RBP, RSP ; RSP (topo da stack) agora é RBP, que contém endereço do RBP da função anterior

pop RBP ; Faz o pop, restaurando o endereço do RBP da função anterior
ret
```

(OBS: `;` é um **comentário** em assembly)

E o que acontece com as variáveis e todas as outras coisas que estavam na stack da função? Ficam lá, como lixo de memória, sem serem modificadas, até que um programa sobrescreva elas.

### cmp

**Compara dois operandos, faz o primeiro menos o segundo e checa se o resultado é maior/menor/igual a zero**. Dependendo do valor, define uma flag de acordo.

Ex: `cmp RAX, 0x10`

### jnz/jz (jumpt if not zero/jump if zero)

Similar ao `jmp`. Mas apenas executa dependendo do status da zero flag.

Existem outros, como `jle` (jump if less or equal).

## 2.6 Stack em Arquitetura de programas (Pilha)

### 2.6.1 Conceito de Stack

A stack é um conceito simples de Estrutura de Dados. Consiste apenas em uma ideia de como organizar informação. Imagine que você possui um tubo onde você pode colocar bolinhas, somente pelo topo. Se você enxer o tubo, para retirar bolinhas, você precisará começar a remover pelas bolinhas mais recentes que você colocou.

A Stack é basicamente isso. Começamos a resolver, ou **desempilhar** as coisas começando das mais recentes para as mais antigas. Na Stack, **o último será o primeiro**.

A stack é uma estrutura de dados com duas operações: push e pop.

- `push` - Coloca novo valor no topo da stack
- `pop` - Remove valor do topo da stack e extrai seu valor

![alt text](./images/stack_data_structure.png)

Imagine agora que temos uma função que chama outra função, e esta função chama outra função. **Função > Função >  Função**. Quando a ultima função retorna, devemos começar a resolver as coisas pela primeira função? Não. Vamos resolvendo pela ordem inversa, até que a segunda retorne na primeira, e a primeira conclua sua execução.

```C
int function1(int n){
    return function2(n) + 1;
}
int function2(int n){
    return function3(n) + 1;
}
int function3(int n){
    return n + 1;
}

int main(){
    int result = function1(10);

    return 0;
}
```

Dessa forma, o conceito de Stack é utilizado na arquitetura de programas para organizar o fluxo do programa e o escopo de funções.

### 2.6.2 Stack para Assembly

A stack é uma **região contínua de memória no computador onde as variáveis locais serão armazenadas**. Ela é uma estrutura de dados com duas operações: **PUSH** (adicionar valor ao topo) e **POP** (retirar valor do topo).

No assembly:
- Instrução `push` - **Empilha** valor na stack
- Instrução `pop` - **Desempilha** valor na stack
- Registrador `RSP` - Possui ponteiro para **topo da stack** (endereço do último bloco de memória correspondente à stack)
- Registrador `RBP` - Possui ponteiro para a **base da stack** (endereço do primeiro bloco de memória correspondente à stack)

Na memória, **o topo da pilha "cresce negativamente"**. Assim, sempre que um valor é empilhado na stack, o valor de esp é decrescido.

Lembre-se que **RSP e RBP NÃO ESTÃO NA STACK, eles são apenas ponteiros**, ou seja, contém os endereços que são a base e o topo da stack.

A stack contém 4 elementos básicos, nesta ordem:

1. **Parâmetros** da **função atual** (isto acontece apenas se houverem mais parâmetros do que a quantidade de registradores disponíveis. **Caso contrário, os parâmetros são passados pelos registradores** `rdi`, `rsi`, `rdx`)
2. **Endereço de retorno** para continuar a partir da **função antiga** (ou seja, a próxima instrução após a chamada da nova função)
```C
int main(){ 
    imprimir_soma(1, 2);
    int a  = 0; // Return Address é um ponteiro para essa instrução, depois da chamada de função
    return 0;
}
```
3. **RBP** da **função antiga** (ou seja, a função que chamou a função atual. Afinal, alguma hora iremos retornar para ela, e voltar a trabalhar com as variáveis locais dela, que ainda estarão armazenados na stack, certo?)
4. **Variáveis locais** da **função atual**

<div styles="display: block; text-align:center">
    <img src="./images/simple_stack.png"/>
</div>

Um ponto importante é que, no código assembly, **todos os dados presentes na stack são acessados com base no RBP** (`RBP-0x4` = Endereço para o qual RBP aponta acrescentado de 4 bytes).

(Lembre-se que, quando mais alto o valor na pilha, ou seja, quanto mais próximo de RSP, mais decrescemos o valor de RBP, pois a stack cresce negativamente)

### 2.6.3 Exemplo de Uso de Stack

Abaixo, temos um código em C e uma simulação de como eles estariam na stack.

Código em **C**:

```C
// Compilado para arquitetura x86 (32 bits) com otimização mínima (ex: -O0)

void simple_function(int p1, int p2, int p3) {
    // Estas serão as Variáveis Locais 1, 2 e 3
    int local1 = p1 * 2;
    int local2 = p2 + 1;
    int local3 = p3 - 1;

    // Usar todas as variáveis garante que o compilador as crie na stack.
    local1 = local2 + local3;
}

int main() {
    // Chamada com 3 parâmetros
    simple_function(10, 20, 30);
    return 0; // Return address da simple_function aponta para aqui (chamada + 1)
}
```

Código em **Assembly**:

```assembly
asm
section .text
    global simple_function
    global main

; int simple_function(int p1, int p2, int p3)
simple_function:
    ; Prólogo
    push rbp
    mov rbp, rsp

    ; local1 = p1 * 2
    mov eax, edi        ; Copia p1 (EDI) para EAX
    shl eax, 1          ; Deslocamento de bits para a esquerda (multiplica por 2)
    ; local1 fica armazenado em EAX

    ; local2 = p2 + 1
    mov ebx, esi        ; Copia p2 (ESI) para EBX
    add ebx, 1          ; Adiciona 1 de EBX

    ; local3 = p3 - 1
    mov ecx, edx        ; Copia p3 (EDX) para ECX
    sub ecx, 1        ; Subtrai 1 de ECX

    ; local1 = local2 + local3
    mov eax, ebx        ; Copia local2 para EAX (EAX é sobrescrito)
    add eax, ecx        ; Adiciona ECX de EAX (local2 + local3)

    ; Epílogo
    pop rbp        ; Desempilha valor do RBP da função anterior salva
    ret        ; Retorna

; int main()
main:
    ; Prologue
    push rbp
    mov rbp, rsp

    ; Call simple_function(10, 20, 30)
    mov edi, 10
    mov esi, 20
    mov edx, 30
    call simple_function

    ; Return 0
    mov eax, 0

    ; Epilogue
    pop rbp
    ret
```

**Stack** ao entrar na função `simple_function`:

<div styles="display: block; text-align:center">
    <img src="./images/stack.png"/>
</div>

Perceba que a Stack está de 4 em 4 bytes (arquitetura x86 = 32 bits). Também perceba que localizamos as variáveis com base no EBP: 

- **[EBP+16]** = parâmetro p3
- **[EBP+12]** = Parâmetro p2
- **[EBP+8]** = Parâmetro p1
- **[EBP+4]** = Endereço de retorno
- **[EBP]** = Valor do EBP da função anterior
- **[EBP-4]** = local1 
- **[EBP-8]** = local2
- **[EBP-12]** = local3

## 2.7 Assembly na prática

Começaremos com alguns problemas básicos de Engenharia Reversa de Assembly.

Os problemas são do repositório: [kablaa-CTF-Workshop](https://github.com/kablaa/CTF-Workshop/blob/master/Reversing/Challenges/IfThen/if_then).

### Hello World

Arquivo: [hello_word](./reversing-assembly/hello_world)

Em Linux, podemos usar o seguinte comando para olhar o código em Assembly: 

```
$    objdump -D hello_world -M intel | less
```

Depois de procurar pela string `main` (que é a função principal), vemos isso:

```assembly
080483fb <main>:
 80483fb:       8d 4c 24 04             lea    ecx,[esp+0x4]
 80483ff:       83 e4 f0                and    esp,0xfffffff0
 8048402:       ff 71 fc                push   DWORD PTR [ecx-0x4]
 8048405:       55                      push   ebp
 8048406:       89 e5                   mov    ebp,esp
 8048408:       51                      push   ecx
 8048409:       83 ec 04                sub    esp,0x4
 804840c:       83 ec 0c                sub    esp,0xc
 804840f:       68 b0 84 04 08          push   0x80484b0
 8048414:       e8 b7 fe ff ff          call   80482d0 <puts@plt>
 8048419:       83 c4 10                add    esp,0x10
 804841c:       b8 00 00 00 00          mov    eax,0x0
 8048421:       8b 4d fc                mov    ecx,DWORD PTR [ebp-0x4]
 8048424:       c9                      leave  
 8048425:       8d 61 fc                lea    esp,[ecx-0x4]
 8048428:       c3                      ret    
 8048429:       66 90                   xchg   ax,ax
 804842b:       66 90                   xchg   ax,ax
 804842d:       66 90                   xchg   ax,ax
 804842f:       90                      nop
```

Olhando para o código, vemos uma function call para `puts`:

```assembly
push   0x80484b0
call   80482d0 <puts@plt>
```

O `puts` é uma função que imprime algo no terminal como texto. O resto não é nada muito interessante. Assim, sabemos que algo será impresso na saída do programa. Se executarmos o código:

```
$    ./hello_world
hello world!
```

### If Then

Começamos vendo o código em assembly com `objdump`:

```
$    objdump -D if_then -M intel | less
```

Após o comando, procuramos pela função main:

```assmebly
080483fb <main>:
 80483fb:       8d 4c 24 04             lea    ecx,[esp+0x4]
 80483ff:       83 e4 f0                and    esp,0xfffffff0
 8048402:       ff 71 fc                push   DWORD PTR [ecx-0x4]
 8048405:       55                      push   ebp
 8048406:       89 e5                   mov    ebp,esp
 8048408:       51                      push   ecx
 8048409:       83 ec 14                sub    esp,0x14
 804840c:       c7 45 f4 0a 00 00 00    mov    DWORD PTR [ebp-0xc],0xa
 8048413:       83 7d f4 0a             cmp    DWORD PTR [ebp-0xc],0xa
 8048417:       75 10                   jne    8048429 <main+0x2e>
 8048419:       83 ec 0c                sub    esp,0xc
 804841c:       68 c0 84 04 08          push   0x80484c0
 8048421:       e8 aa fe ff ff          call   80482d0 <puts@plt>
 8048426:       83 c4 10                add    esp,0x10
 8048429:       b8 00 00 00 00          mov    eax,0x0
 804842e:       8b 4d fc                mov    ecx,DWORD PTR [ebp-0x4]
 8048431:       c9                      leave  
 8048432:       8d 61 fc                lea    esp,[ecx-0x4]
 8048435:       c3                      ret    
 8048436:       66 90                   xchg   ax,ax
 8048438:       66 90                   xchg   ax,ax
 804843a:       66 90                   xchg   ax,ax
 804843c:       66 90                   xchg   ax,ax
 804843e:       66 90                   xchg   ax,ax
```

Podemos ver que o valor `0xa` (10 em hexadecimal) é carregado em `ebp-0xc`:

```
mov    DWORD PTR [ebp-0xc],0xa
```

Imediatamente após isso, vemos que há uma instrução `cmp`, seguida de `jne`. Isso checa se é o dado presente em `ebp-0xc` é igual a 0xa. Se não for igual, `jne` irá pular para `main+0x2e` (obs: `main+0x2e` é a mesma coisa que pegar o endereço base da função main, ou seja, o endereço da primeira instrução, e acrescentar `0x2e` a ele)


Como o valor é igual, o jump não será executado:

```
cmp    DWORD PTR [ebp-0xc],0xa
jne    8048429 <main+0x2e>
```

Prosseguindo, é feita uma chamada para a função puts, o que quer dizer que algo será impresso na tela:

```
sub    esp,0xc
push   0x80484c0
call   80482d0 <puts@plt>
```

Após rodar o código, podemos rodá-lo e ver o que faz:

```
$    ./if_then
x = ten
```

### Loop

Começamos vendo o código em assembly com `objdump`:

```
$    objdump -D loop -M intel | less
```

Procurando pela função main, vemos isso:

```
080483fb <main>:
 80483fb:       8d 4c 24 04             lea    ecx,[esp+0x4]
 80483ff:       83 e4 f0                and    esp,0xfffffff0
 8048402:       ff 71 fc                push   DWORD PTR [ecx-0x4]
 8048405:       55                      push   ebp
 8048406:       89 e5                   mov    ebp,esp
 8048408:       51                      push   ecx
 8048409:       83 ec 14                sub    esp,0x14
 804840c:       c7 45 f4 00 00 00 00    mov    DWORD PTR [ebp-0xc],0x0
 8048413:       eb 17                   jmp    804842c <main+0x31>
 8048415:       83 ec 08                sub    esp,0x8
 8048418:       ff 75 f4                push   DWORD PTR [ebp-0xc]
 804841b:       68 c0 84 04 08          push   0x80484c0
 8048420:       e8 ab fe ff ff          call   80482d0 <printf@plt>
 8048425:       83 c4 10                add    esp,0x10
 8048428:       83 45 f4 01             add    DWORD PTR [ebp-0xc],0x1
 804842c:       83 7d f4 13             cmp    DWORD PTR [ebp-0xc],0x13
 8048430:       7e e3                   jle    8048415 <main+0x1a>
 8048432:       b8 00 00 00 00          mov    eax,0x0
 8048437:       8b 4d fc                mov    ecx,DWORD PTR [ebp-0x4]
 804843a:       c9                      leave  
 804843b:       8d 61 fc                lea    esp,[ecx-0x4]
 804843e:       c3                      ret    
 804843f:       90                      nop
```

Nessa função, podemos ver que é inicializada uma variável de stack em `ebp-0xc` como `0`, e depois pula para `0x804842c` (`main_0x31`):

```
mov    DWORD PTR [ebp-0xc],0x0
jmp    804842c <main+0x31>
```

Olhando para as instruções em `0x804842c` vemos isso:

```
cmp    DWORD PTR [ebp-0xc],0x13
jle    8048415 <main+0x1a>
```

Isso compara o valor da stack em `ebp-0xc` contra `0x13` (19 em decimal), e se for menor ou igual então irá pular para `0x8048415` (`0x80483fb+0x1a`). Isso nos leva a uma call para uma função printf:

```
sub    esp,0x8
push   DWORD PTR [ebp-0xc]
push   0x80484c0
call   80482d0 <printf@plt>
```

Parece que isso está imprimindo o coteúdo de `ebp-0xc` em algum tipo de format string. Depois disso podemos ver que o valor de `ebp-0xc` é incremendado em 1, antes de fazer o `cmp` de novo:

```
add    DWORD PTR [ebp-0xc],0x1
```

Certo, juntando todas as peças, agora nós estamos provavelmente olhando para um loop for que vai rodar 20 vezes, e printar o contador de iteração em cada uma das iterações. Isso parece algo similar a:

```C
int i = 0;
for (i = 0; i < 20; i ++)
{
    printf("%d", i);
}
```

Quando rodamos o binário, vemos que isso é verdade:

```
$    ./loop
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19
```

## 2.8 Tópico Adicional: Diretivas Assembly

Usamos assembly apenas para análise, mas ele pode ser trabalhado como se fosse uma linguagem de programação comum. Para compilar um código em assembly em código de máquina, temos o assembler.

Dado isto, temos as **diretivas** no código assembly, que não existem no executável final. Elas servem para guiar o Assembler na compilação do código:

```
Assembly Source (.s) → Assembler → Object File (.o) → Linker → Executable (.exe)
     ↓                    ↓              ↓               ↓          ↓
  [DIRETIVAS]        [PROCESSADAS]   [REMOVIDAS]   [REMOVIDAS] [CÓDIGO PURO]
```

Abaixo temos uma lista de diretivas, e o que elas fazem:

```nasm
; Diretivas de Dados/Strings
.byte 0x48, 0x31, 0xc0    ; 3 bytes: 48 31 c0
.byte 'A','B','C'          ; Bytes ASCII: 41 42 43
.word 0x1234              ; 2 bytes: 34 12 (little-endian)
.short 255                ; 2 bytes: FF 00
.long 0x12345678          ; 4 bytes: 78 56 34 12
.int 0x41414141           ; 4 bytes: 41 41 41 41
.quad 0x4141414141414141  ; 8 bytes de 'A'
.quad 0x1234567890ABCDEF  ; 8 bytes qualquer
.ascii "Hello"            ; 48 65 6c 6c 6f (sem \0)
.ascii "flag.txt"         ; 66 6c 61 67 2e 74 78 74
.asciz "Hello"            ; 48 65 6c 6c 6f 00
.string "test"            ; 74 65 73 74 00

; Diretivas de Arquivos/Inclusão
.include "other.s"        ; Inclui código de other.s
.incbin "file.bin"        ; Insere bytes brutos de file.bin
.file "mysource.s"        ; Informa nome do arquivo fonte

; Diretivas de Seções/Organização
.section .text            ; Seção de código
.section .data            ; Seção de dados
.section .rodata          ; Seção de dados somente leitura
.text                    ; Mudar para .text
.data                    ; Mudar para .data  
.bss                     ; Mudar para .bss (dados não inicializados)
.global _start           ; _start visível externamente
.globl main              ; main visível externamente

; Diretivas de Alinhamento/Tamanho
.align 4                 ; Alinha para múltiplo de 4
.align 8                 ; Alinha para múltiplo de 8
.skip 100                ; 100 bytes de 0x00
.space 50, 0x41          ; 50 bytes de 'A'
.zero 64                 ; 64 bytes de 0x00

; Diretivas de Repetição
.rept 10
.byte 0x90               ; 10 bytes de NOP
.endr
.fill 100, 1, 0x41       ; 100 bytes de 'A'
.fill 50, 4, 0x90909090  ; 50 grupos de 4 bytes NOP

; Diretivas de labels/Símbolos
.set BUFFER_SIZE, 100
.equ SYS_EXIT, 60
.macro exit code=0
    mov rax, 60
    mov rdi, \code
    syscall
.endm

; Condicionais
.if 1 == 1
.byte 0x41
.else  
.byte 0x42
.endif

; Definir endereço atual
.org 0x400000            ; Define endereço atual

; Declarar variável comum
.comm buffer, 100        ; Variável global não inicializada
```

Um exemplo de uso:

```
.text
.global _start
.file "exploit.s"

_start:
    .align 8 ; Adiciona padding
    mov rax, 1 ; número 1 no registrador rax
    .byte 0x48, 0x31, 0xc0 ; Insere bytes
    .ascii "TEST" ; Insere string ("TEXT" = 0x54 0x45 0x53 0x54)
```

O executável final ficaria:

```
00000000: 48b8 0100 0000 0000 0000 4831 c054 4553  H.........H1.TES
00000010: 54                                        T
```

Perceba que:

- `48b8 0100` - mov rax, 1
- `4831 c0` - Bytes inseridos
- `54 4553 54` - String "TEXT"

`.file`, `.text` e `.global` sumiram.

## 2.8 Assembly CheatSheet

- **Registradores** - Memória no processador que armazena informação
    - `rbp`: Base Pointer, aponta para a **base do stack** frame atual
    - `rsp`: Stack Pointer, aponta para o **topo da stack** frame atual
    - `rip`: Instruction Pointer, aponta para a **instrução a ser executada**
    - Prefixos: `r` - 8 bytes. `e` - 4 bytes. `nada` - 2 bytes. Sufixo `l` - Lower byte
- Words
    - `word` - 2 bytes
    - `dword` - 4 bytes
    - `qword` - 8 bytes
- **Stack** - Região de memória com operações push/pop, onde variáveis locais são armazenadas. Cresce negativamente (topo é endereço menor, base é endereço maior).
- **Instruções**
    - `mov` - **Move dados de um registrador para outro** (segundo para o primeiro).
    - `dereference` - **colchetes [ ] referenciam dados ao qual um ponteiro aponta**.
    - `lea` - **Calcula o endereço do segundo operando**, e move para o primeiro.
    - `add` - **Adiciona dois valores** e armazena no primeiro operando.
    - `sub` - **Subtrai o segundo operando do primeiro** e armazena no primeiro.
    - `xor` - **Faz o xor nos dois argumentos**, e armazena no primeiro operando. and e or são semelhantes.
    - `push` - **Cresce a stack em 8 ou 4 bytes** (para x64, ou 4 para x86), **depois** **adiciona o conteúdo do registrador para o topo da stack.**
    - `pop` - Tira 8 bytes do topo da stack e armazena no argumento. Depois encolhe a stack.
    - `jmp` - **Pula para um endereço de instrução**. Redireciona a execução do código.
    - `call` - Redução dos comandos `push RIP` e `jmp function`. Utilizado sempre que se chama uma função.
    - `ret` - Redução do comando `pop RIP` (desempilha topo da stack para voltar para a execução de código antes da chamada de função, e atribui o valor do pop ao RIP).
    - `cmp` - **Compara dois operandos, faz o primeiro menos o segundo e checa se o resultado é maior/menor/igual a zero**. Dependendo do valor, **define uma flag** de acordo.
    - `jnz/jz` -  (jump if not zero/jump if zero) Similar ao jmp. Mas apenas **executa dependendo do status da zero flag**. Existem outros jumps.
