---
name: samsung-kernel-debug
description: >-
  Metodologia definitiva de diagnóstico e depuração de kernel para Samsung Galaxy (Exynos 1330 / s5e8535 / A146B).
  Utilize quando houver bootloops, kernel panics, travamento em initcalls ou necessidade de capturar logs de crash após reinicialização.
---

# 🩺 Metodologia Definitiva de Debug de Kernel Samsung (Exynos / A146B)

Diagnosticar bootloops e kernel panics sem adivinhação exige a preservação do buffer de memória no hardware. Em dispositivos Android modernos, a ferramenta mais importante não é depuração pesada, mas sim **PSTORE / RAMOOPS**.

---

## 👑 A Estrela do Debug: PSTORE + RAMOOPS

Se o aparelho entrar em:
$$\text{Boot} \longrightarrow \text{Kernel Panic / Oops} \longrightarrow \text{Reboot Automático}$$
Os logs voláteis do `dmesg` desaparecem, a menos que o **RAMOOPS** preserve o buffer de memória na RAM reservada durante o ciclo de reinicialização.

### 1. Configurações Obrigatórias do Kernel

```ini
# --- PSTORE / RAMOOPS (Preservação de Crash) ---
CONFIG_PSTORE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_PMSG=y

# --- LOGGING & TEMPO ---
CONFIG_PRINTK=y
CONFIG_PRINTK_TIME=y

# --- SÍMBOLOS HUMANAMENTE LEGÍVEIS (Evita endereços hexadecimais puros) ---
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_STACKTRACE=y
CONFIG_FRAME_POINTER=y

# --- RUNTIME CONTROL & DEBUG FS ---
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_MAGIC_SYSRQ=y

# --- COMPORTAMENTO SOBRE PANIC ---
CONFIG_PANIC_ON_OOPS=n
```

### 2. Bootargs de Diagnóstico (Linha de Comando de Boot)

Injetar no `CONFIG_CMDLINE` ou via bootloader:
```bash
ignore_loglevel log_buf_len=4M initcall_debug printk.time=1 panic=5
```
* `ignore_loglevel`: Garante que até mensagens de nível baixo cheguem ao console/ramoops.
* `log_buf_len=4M`: Aumenta o buffer circular para não truncar mensagens iniciais de inicialização.
* `initcall_debug`: Mostra exatamente qual driver iniciou e em qual linha o boot travou:
  ```text
  calling  exynos_foo_init+0x0/0x...
  initcall exynos_foo_init returned 0 after 1234 usecs
  calling  samsung_bar_init...  <-- SE TRAVAR AQUI, O CULPADO FOI ENCONTRADO
  ```
* `panic=5`: Reinicia após 5 segundos, garantindo tempo para o ramoops escrever os dados.
*(Nota: Para depuração via UART direta com cabo serial, use `panic=-1` para congelar a tela sem reiniciar).*

---

## 📥 Como Resgatar os Logs Após o Bootloop

Assim que o dispositivo reiniciar após o pânico, conecte via ADB:

```bash
adb shell
su
ls -lah /sys/fs/pstore
```

Arquivos esperados:
* `console-ramoops-0`: Contém o console serial completo até o exato milissegundo do panic.
* `dmesg-ramoops-0`: O ringbuffer do dmesg preservado.
* `pmsg-ramoops-0`: Mensagens do subsistema de logging do Android.

Para ler o crash:
```bash
cat /sys/fs/pstore/console-ramoops-0 | tail -n 100
```

---

## 🔍 Depuração Dinâmica em Runtime (Dynamic Debug)

Com `CONFIG_DYNAMIC_DEBUG=y`, não é necessário encher o código de `pr_info()`. Ative logs específicos em tempo de execução via sysfs:

```bash
# Ativar debug para um arquivo específico:
echo 'file drivers/gpu/arm/.../gpex_dvfs.c +p' > /sys/kernel/debug/dynamic_debug/control

# Ativar debug para um módulo inteiro:
echo 'module sm5714_charger +p' > /sys/kernel/debug/dynamic_debug/control
```

---

## 🚫 O que NÃO Ligar Inicialmente

Não ative depuradores pesados no primeiro build de diagnóstico:
* ❌ `CONFIG_KASAN`
* ❌ `CONFIG_KCSAN`
* ❌ `CONFIG_LOCKDEP`
* ❌ `CONFIG_DEBUG_PAGEALLOC`
* ❌ `CONFIG_SLUB_DEBUG`

**Motivo**: Esses instrumentos alteram o timing de execução, o consumo de memória e a latência de interrupções, criando novos bugs ou mascarando o problema original (*efeito Heisenbug*).

---

## 🏗️ Padrão dos Dois Defconfigs

Mantenha sempre dois perfis isolados:
1. `s5e8535-a14xxx_defconfig` (Release / Produção limpo)
2. `a146b_debug_defconfig` (Com PSTORE, RAMOOPS, DYNAMIC_DEBUG e initcall_debug)
