Excelente coleção! Aqui estão resumos concisos de cada tópico em pwning:

## **Vulnerabilidades Clássicas**
- **Stack Buffer Overflows** - Estouro de buffer na stack permitindo sobrescrever endereço de retorno ✅
- **Format Strings** - Input do usuário usado como format string, permitindo leitura/escrita arbitrária ✅
- **Array indexing** - Acesso a índices fora dos limites do array ✅
- **Bad seed** - Vulnerabilidades em geradores de números aleatórios mal implementados ✅
- **Integer overflows** - Estouro em operações aritméticas causando comportamentos inesperados ✅

https://ctftime.org/writeup/31381

## **Técnicas de Exploração Avançada**
- **Z3 & Symbolic execution (angr)** - Análise simbólica para resolver constraints e automatizar exploits
- **Uninitialized variables** - Uso de variáveis não inicializadas que contêm dados residuais da memória

## **Return Oriented Programming (ROP)**
- **ROP básico** - Encadeamento de gadgets (trechos de código existentes) para execução arbitrária
- **Partial Overwrite** - Sobrescrever apenas parte de um endereço para bypass de ASLR
- **Stack pivoting** - Mover stack pointer para área controlada pelo atacante
- **SIGROP (SROP)** - Abuso do signal handling para controlar todos os registradores
- **ret2csu** - Uso do gadget `__libc_csu_init` para configurar múltiplos registradores
- **ret2system** - Retorno direto para `system()` com argumento controlado

## **Heap Exploitation**
- **double frees** - Liberar mesmo chunk de memória duas vezes
- **Heap consolidation** - Fusão de chunks livres adjacentes
- **Use-after-frees** - Usar memória já liberada
- **Protostar** - Série de desafios introdutórios de heap exploitation
- **unlink() exploitation** - Abuso do mecanismo de unlink em chunks livres
- **heap grooming** - Manipular layout do heap para facilitar exploração
- **fastbin attack** - Corrupção de fastbins para alocação arbitrária
- **unsortedbin attack** - Corromper unsorted bin para escrever em endereço arbitrário
- **largebin attack** - Ataque similar usando largebins
- **glibc tcache** - Thread local caching introduzido no glibc 2.26+
- **house of spirit** - Enganar malloc a aceitar chunk falso
- **house of lore** - Abuso de smallbins para alocação arbitrária
- **house of force** - Estourar wilderness para controlar futuras alocações
- **house of einherjar** - Consolidar chunk falso via overflow
- **house of Orange** - Usar _IO_FILE para obter execução arbitrária

## **FILE Structure Exploitation**
- **FILE exploitation** - Abuso da estrutura FILE (`_IO_FILE`) para RCE via file streams

## **Técnicas Diversas (Grab Bag)**
- **shellcoding** - Escrever código de shell em assembly para execução direta
- **patching** - Modificar binários para facilitar exploração ou remover proteções
- **.NET** - Exploração em ambiente .NET (métodos gerenciados)
- **obfuscation** - Técnicas para ofuscar código e dificultar análise
- **custom architecture** - Exploração em arquiteturas não convencionais
- **emulation** - Usar emuladores para análise dinâmica
- **unitialized variables** - Uso de variáveis não inicializadas que contêm dados residuais

## **Hierarquia de Importância (para iniciantes):**
1. **Fundamental**: Stack BO, ROP básico, Format Strings
2. **Intermediário**: Heap básico, Integer Overflows, Shellcoding
3. **Avançado**: Houses of Heap, FILE exploitation, SROP
4. **Especializado**: .NET, custom arch, symbolic execution

## **Tendências Atuais:**
- **Heap > Stack** (com proteções modernas)
- **FILE struct** ainda relevante mesmo com mitigações
- **tcache** foco principal em heap pós-2017
- **Symbolic exec** para automação em CTFs complexos

Cada tópico representa uma "ferramenta" no arsenal do pentester/CTF player - o domínio vem da combinação adequada conforme a vulnerabilidade encontrada! 🏴‍☠️

--------------------------------------------------------------------------------

Stack Buffer Overflows
Format Strings
Array indexing
Bad seed
Z3 & Symbolic execution (angr)
Return Oriented Programming (ROP)
	Partial Overwrite
	Stack pivoting
	SIGROP (SROP)
	ret2csu
	ret2system
Heap exploitation
	double frees
	Heap consolidation
	Use-after-frees
	Protostar
	unlink() exploitation
	heap grooming
	fastbin attack
	unsortedbin attack
	largebin attack
	glibc tcache
	house of spirit
	house of lore
	house of force
	house of einherjar
	house of Orange
inteer overflows
FILE exploitation
grab bag
	shellcoding
	patching
	.NET
	obfuscation
	custom architecture
	emulation
	unitialized variables