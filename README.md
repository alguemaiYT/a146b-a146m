# Samsung Galaxy A14 5G (SM-A146B / SM-A146M) Custom Kernel
### Android 15 (Vanilla Ice Cream / One UI 7) • Exynos 1330 (s5e8535 / a14x)

Kernel de alto desempenho baseado no código oficial do **[physwizz/a146b-a146m](https://github.com/physwizz/a146b-a146m)** (Linux 5.15.180), integrando as melhores correções de subssistemas Samsung do **langsdorffkernel**, a suíte de otimizações **Exynos 5.15 do Project-24 (MrPankaj24)** e suporte avançado a **RKSU v3.0.0** com Kprobes desativado.

---

## 📱 Especificações Técnicas

| Parâmetro | Valor / Detalhe |
| :--- | :--- |
| **Aparelho** | Samsung Galaxy A14 5G (`SM-A146B` / `SM-A146M`) |
| **Plataforma / SoC** | Samsung Exynos 1330 (`s5e8535` / `a14x`) |
| **Arquitetura** | ARM64 (2x Cortex-A78 @ 2.4 GHz + 6x Cortex-A55 @ 2.0 GHz) |
| **GPU** | ARM Mali-G68 MP2 (Valhall) |
| **Versão Base do Linux** | Linux `5.15.180` |
| **Base Android** | Android 15 (V / Vanilla Ice Cream / One UI 7) |
| **Defconfig Principal** | `arch/arm64/configs/s5e8535-a14xxx_defconfig` |

---

## 🛠️ Toolchain & Ambiente de Build (TC)

* **Compilador Principal**: AOSP Clang `r487747c` / `r522817` (Clang 17 / 18 / 19 base)
* **Cross-Compiler Host**: `aarch64-linux-gnu-gcc` & `arm-linux-gnueabi-gcc`
* **Ferramentas de Linkagem**: LLD (`ld.lld`), `llvm-ar`, `llvm-nm`, `llvm-objcopy`
* **Otimização de Linkagem (LTO)**: Clang ThinLTO / Full LTO (`ARCH_SUPPORTS_LTO_CLANG_FULL=y`)
* **Aceleração**: `ccache` com threads paralelas (`make -j12` em VM Spot Google Cloud)
* **Compatibilidade**: Patches aplicados para prevenir erros de `sizeof-pointer-memaccess` em Clang moderno

---

## ⚡ Lista Completa de Tweaks, Correções e Patches

### 1. 🛡️ Root, Segurança & Anti-Detecção
* **RKSU v3.0.0 (Legacy Hooks v2)**: Root nativo de alta estabilidade com Syscall Hook.
* **Kprobes Desativado (`CONFIG_KPROBES=n`)**: Desativa instrumentação de kprobes para maior fluidez e menor detecção.
* **Suporte a SUSFS & Zeromount**: Primitivas no kernel para ocultação profunda de root e módulos, garantindo aprovação em Play Integrity (Device/Strong) e aplicativos bancários.
* **Samsung FIVE Desativado (`CONFIG_FIVE=n`)**: Remove a verificação de assinatura da Samsung a cada `exec()` e `mmap()`, acelerando o tempo de abertura de aplicativos.
* **AVB & DM-Verity Desativados**: Permite boot livre com partições modificadas ou GSIs.

---

### 2. 🧠 Escalonador & CPU (Samsung EMS / CFS)
* **Fix de Desalinhamento de Cgroups no EMS**: Adiciona o cgroup `dex2oat` entre `system-background` e `nnapi-hal` em `kernel/sched/ems/`, corrigindo o bug da Samsung que deslocava todas as prioridades de câmera e IA por 1 índice.
* **Heavy Task Boost no Root Cgroup**: Permite que decodificadores de vídeo por software (VP9/AV1 no `media.swcodec`) recebam boost de CPU automático quando demandados.
* **Boot no Governor `energy_aware` (EGO)**: Redireciona a escrita inicial de `schedutil` para `energy_aware` com limite de resposta rápido de 4ms, reduzindo o tempo de quadro em ~6.8%.
* **Calibração de Latência CFS (`fair.c`)**: Restaura `sysctl_sched_latency` para 6ms e `min_granularity` para 0.75ms, eliminando trocas de contexto (*context switches*) excessivas.
* **Restauração de Controle UFCC**: Reabilita escrita nos nós `min_limit` e `min_limit_wo_boost` em `drivers/soc/samsung/exynos-ufcc.c`.

---

### 3. 💾 Gerenciamento de Memória & ZRAM
* **ZRAM Sem Readahead (`ra_pages = 0`)**: Elimina leitura antecipada na ZRAM, impedindo que o processador desperdice ciclos descompactando páginas vizinhas na RAM.
* **Limites de Dirty Writeback Otimizados (40/10)**: Reduz `dirty_background_ratio` para 10% e `vm_dirty_ratio` para 40% em `mm/page-writeback.c`, eliminando engasgos causados por acúmulo de dados na memória flash.
* **Readahead MMC Reduzido para 256KB**: Reduz a sobrecarga do cache de páginas no eMMC/UFS, poupando memória preciosa em aparelhos de 4GB/6GB RAM.
* **Swappiness Ajustado para 130**: Favorece a compressão de páginas anônimas ociosas para a ZRAM antes de descartar caches de arquivos da interface, melhorando a retenção de apps.
* **Remoção de KASAN, MTE e KFENCE**: Libera ~7.4MB de RAM gastos com tabelas de páginas e restaura o mapeamento em blocos contíguos de 2MB no MMU ARM64 (`rodata=on`), acelerando os acertos de TLB.
* **Nós de Sysfs `am_app_launch` Expostos**: Permite que serviços de usuário ativem os perfis de memória `sec_mm` na abertura de apps.

---

### 4. 🔋 Hardware & Bateria (Exynos 1330)
* **Bypass Charging Real para o Chip SM5714 (`Bypass_charging_fix.patch`)**: Habilita a alimentação direta pela fonte no controlador Silicon Mitus SM5714. Permite jogar ou executar cargas pesadas conectado ao carregador sem aquecer a bateria.
* **Desativação de CRC no MMC (`use_spi_crc = 0`)**: Remove o cálculo de redundância de CRC em transferências do armazenamento interno e cartão MicroSD, melhorando a taxa de transferência de I/O.
* **Assembly Memcmp Otimizado para Exynos (`optimise_memcmp_exynos.patch`)**: Rotina de comparação de memória ultraveloz em código de máquina ARM64 para chips Samsung.
* **Alinhamento de Memória (16-byte / 8-byte)**: `clear_page_16bytes_align` e `file_struct_8bytes_align` para maior velocidade de barramento.
* **Otimização de Partição F2FS (`/data`)**: Redução de contenção e ajuste de blocos mínimos de fsync (`f2fs_reduce_congestion`, `f2fs_enlarge_min_fsync_blocks`).
* **Supressão de Logspam**: Desativa o envio contínuo de logs inúteis de interrupção de IRQ e sistema no `dmesg`.

---

### 5. 🌐 Rede & Jogos
* **Google TCP BBRv3 (`tcp_bbr3.c`)**: Algoritmo de controle de congestionamento de rede de 3ª geração do Google backportado para o Linux 5.15, garantindo menor ping e downloads mais rápidos e estáveis no 5G e Wi-Fi.
* **Primitivas NTSync**: Suporte direto no kernel a primitivas de sincronização NT para jogos emulados de Windows via Winlator, Mobox e Box64.
* **Módulo Re:Kernel**: Suporte a congelamento otimizado de processos de segundo plano para preservação de bateria.
* **Elevador de I/O `mq-deadline`**: Menor latência de acesso aos blocos de disco flash em comparação com o escalonador `bfq`.

---

## 📦 Como Instalar

1. Baixe o arquivo `.tar` gerado no build.
2. Coloque o aparelho em **Download Mode** (Volume Up + Volume Down conectados ao cabo USB).
3. Abra o **Odin3** ou **Brokkr** no PC.
4. Insira o `.tar` no slot **AP** e clique em **Start**.
5. *(Alternativa)* Se estiver usando TWRP, instale a imagem diretamente na partição **Boot** (`boot.img`).
