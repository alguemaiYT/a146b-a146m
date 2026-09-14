---
name: samsung-kernel-debug
description: >-
  Metodologia definitiva de diagnóstico e depuração de kernel para Samsung Galaxy (Exynos 1330 / s5e8535 / A146B).
  Utilize quando houver bootloops, kernel panics, travamento em initcalls ou necessidade de capturar logs de crash após reinicialização.
---

# 🩺 Metodologia Definitiva de Debug de Kernel Samsung (Exynos 1330 / A146B)

Diagnosticar bootloops e kernel panics sem adivinhação exige a preservação do buffer de memória no hardware e a correspondência exata de símbolos de depuração DWARF.

---

## ⚠️ Os 4 Mandamentos Críticos do Debug no Exynos 1330

### 1. Ramoops vs Samsung Debug Snapshot (DSS)
* **Regra de Ouro**: `CONFIG_PSTORE_RAM=y` **não cria sozinho** a região de memória. Ele exige um bloco de `reserved-memory` no Device Tree.
* **No Exynos 1330 (`s5e8535.dts`)**: A Samsung já possui uma área proprietária dedicada chamada **Debug Snapshot (`dss_rmem`)**:
  * `header`: `0x00 0xfd000000` (64KB)
  * `log_kernel`: `0x00 0xfd010000` (2MB) — buffer de log de kernel persistente
  * `log_s2d`: `0x00 0xfd210000` (5MB)
  * `log_first`: `0x08 0xad400000` (2MB)
  * `log_kevents`: `0x08 0xad600000` (5MB)
* **Perigo**: **Nunca** declare uma região `ramoops@...` que se sobreponha a esses endereços físicos. No Exynos, os logs de pânico sobrevivem via DSS da Samsung e via PSTORE quando mapeado em memória não conflitante.

---

### 2. `CONFIG_CMDLINE_EXTEND=y` (Jamais use `CMDLINE_FORCE`)
* No ARM64 com Android moderno, se o bootloader passar a linha de comando de boot, a configuração padrão pode ignorar o `CONFIG_CMDLINE` interno.
* **A Solução Correta**:
  * `CONFIG_CMDLINE_EXTEND=y` (Garante que os bootargs de debug sejam concatenados aos do bootloader).
  * **Nunca** use `CONFIG_CMDLINE_FORCE=y`: isso descartaria parâmetros essenciais de hardware passados pelo bootloader da Samsung (revisão de placa, calibração de PMIC, display), gerando um novo bootloop antes do kernel carregar.
* **Verificação pós-boot**:
  ```bash
  cat /proc/cmdline
  cat /proc/bootconfig
  ```

---

### 3. Símbolos Offline com `CONFIG_DEBUG_INFO_DWARF4=y`
* Ter `CONFIG_KALLSYMS_ALL=y` no aparelho é ótimo para a stack trace imediata, mas para encontrar a linha exata no código-fonte, o arquivo `vmlinux` unstripped com **DWARF4** é indispensável.
* **Configs obrigatórias**:
  ```ini
  CONFIG_DEBUG_INFO=y
  CONFIG_DEBUG_INFO_DWARF4=y
  CONFIG_DEBUG_INFO_BTF=y
  ```
* **Decodificando o endereço do Panic**:
  Quando o log do crash apontar: `foo_bar+0x94/0x1c0`:
  ```bash
  # Via faddr2line oficial do kernel:
  ./scripts/faddr2line out-debug/vmlinux 'foo_bar+0x94/0x1c0'
  
  # Ou via llvm-addr2line:
  llvm-addr2line -e out-debug/vmlinux -f -C 0xffffffc0XXXXXXXX
  ```
  Saída direta: `drivers/foo/bar.c:472` (o arquivo e a linha exata do pânico).

* **Preservação de Artefatos por Build**:
  Sempre arquive juntos em `artifacts/`:
  * `Image` (o binário flashado)
  * `vmlinux` (o ELF gigante com os símbolos DWARF4 exatos daquele build)
  * `System.map`
  * `.config`
  * `build.log`

---

### 4. Rastreabilidade com Git Hash (`CONFIG_LOCALVERSION`)
Para garantir correspondência inequívoca de 1:1 entre o crash do aparelho e o arquivo `vmlinux` guardado no PC:
```bash
GIT_HASH=$(git rev-parse --short HEAD)
CONFIG_LOCALVERSION="-A146B-DEBUG-g${GIT_HASH}"
```
Ao executar `uname -a` no Android, o kernel exibirá exatamente:
`Linux localhost 5.15.180-A146B-DEBUG-g5a557e0 #1 SMP PREEMPT ...`

---

## 🛠️ O Perfil Completo de Compilação DEBUG

```ini
# --- PSTORE / CRASH LOGS ---
CONFIG_PSTORE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_PMSG=y

# --- LOGGING ---
CONFIG_PRINTK=y
CONFIG_PRINTK_TIME=y

# --- SÍMBOLOS & TRACE ---
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_STACKTRACE=y
CONFIG_FRAME_POINTER=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF4=y

# --- RUNTIME CONTROL ---
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_MAGIC_SYSRQ=y

# --- COMPORTAMENTO DE PANIC ---
CONFIG_PANIC_ON_OOPS=n

# --- LINHA DE COMANDO ---
CONFIG_CMDLINE_EXTEND=y
CONFIG_CMDLINE="... ignore_loglevel log_buf_len=4M initcall_debug printk.time=1 panic=5"
```

---

## 🚫 O que NÃO Ativar no Debug Inicial

* ❌ `CONFIG_KASAN`
* ❌ `CONFIG_KCSAN`
* ❌ `CONFIG_LOCKDEP`
* ❌ `CONFIG_DEBUG_PAGEALLOC`
* ❌ `CONFIG_SLUB_DEBUG`

Esses instrumentos aumentam a latência e o consumo de memória, criando novos bugs de concorrência ou escondendo o problema original.
