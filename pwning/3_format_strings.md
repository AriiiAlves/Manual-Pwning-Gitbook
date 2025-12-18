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

