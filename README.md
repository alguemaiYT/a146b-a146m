# 🚀 Samsung Galaxy A14 5G (SM-A146B) Custom Kernel
### Exynos 1330 (`s5e8535` / `a14x`) • Android 15 (Linux 5.15) • Configuration: **Balanced V1 Master Blueprint**

Kernel customizado de alto rendimento e estabilidade cirúrgica para o **Galaxy A14 5G (`SM-A146B` / `SM-A146M`)**, baseado na árvore de fontes oficial do **[physwizz/a146b-a146m](https://github.com/physwizz/a146b-a146m)** (Linux 5.15.180).

Este projeto rejeita "placebos" e receitas genéricas herdadas da era de eMMC/2014, adotando uma arquitetura moderna orientada ao hardware real: **Armazenamento UFS 2.2 (`fstab.s5e8535.ufs.ab`)**, subsistema **Samsung EMS (EAS/EGO)**, **Multi-Gen LRU (MGLRU)**, **UCLAMP** calibrado, primitivas **NTSync/futex_waitv** para emulação de PC e **SELinux Enforcing** com root invisível via **RKSU + SUSFS**.

---

## 📱 Especificações Técnicas da Plataforma

| Parâmetro | Especificação / Hardware Real |
| :--- | :--- |
| **Aparelho** | Samsung Galaxy A14 5G (`SM-A146B`, `SM-A146M`) |
| **SoC / Plataforma** | Samsung Exynos 1330 (`s5e8535` / `TARGET_SOC=s5e8535`) |
| **Topologia de CPU** | Octa-Core: 2x Cortex-A78 @ 2.4 GHz (Big) + 6x Cortex-A55 @ 2.0 GHz (LITTLE) |
| **GPU** | ARM Mali-G68 MP2 (Arquitetura Valhall) |
| **Armazenamento** | **UFS 2.2** (Partição `/data` formatada em F2FS nativo) |
| **Carregador / PMIC** | Silicon Mitus SM5714 (`drivers/battery/charger/sm5714_charger/`) |
| **Kernel Base** | Linux `5.15.180` (Android 15 / One UI 7) |
| **Defconfigs Base** | `erd8535_t_gki_defconfig` / `s5e8535-a14xxx_defconfig` |
| **Árvore Upstream** | `https://github.com/physwizz/a146b-a146m` (branch `main`) |

---

## 📊 Matriz de Prioridade & Impacto Real

Diferente de listas comuns que misturam pequenas alterações com mudanças fundamentais, o projeto organiza cada tweak pelo seu impacto comprovado no A14 5G:

```
★★★★★ (MÁXIMA PRIORIDADE - GAME CHANGERS)
├── MGLRU (Multi-Gen LRU) ────────── Elimina engasgos de refault e spikes de reclaim em RAM de 4/6GB
├── UCLAMP por Cgroup / Top-App ─── Desempenho sob demanda sem travamento permanente de clock
├── NTSync & futex_waitv ────────── Primitivas no kernel essenciais para Wine/Winlator sem gargalo
├── Perfil EMS para Top-App ─────── Uso inteligente dos 2 núcleos A78 sem queimar eficiência térmica
└── Cpuset / Cgroups Corretos ───── Segregação rígida de threads em background vs foreground

★★★★ (ALTA PRIORIDADE - ESTABILIDADE & EFICIÊNCIA)
├── PSI + LMKD Calibrados ───────── Pressão de memória detectada antes do congelamento da interface
├── ZRAM LZ4/ZSTD + ra_pages=0 ──── Fim da descompressão fantasma; compressão por página (page-cluster=0)
├── Reclaim & Dirty Writeback ───── Limites 40/10 evitando congelamentos de escrita no UFS
├── ARM64 LSE Atomics ───────────── Operações atômicas nativas ARMv8.1+ acelerando multithread e Wine
└── F2FS Congestion Conservador ─── Ajustes de I/O em /data sem quebrar durabilidade de dados

★★★ (MÉDIA PRIORIDADE - PERFORMANCE CONTÍNUA)
├── UFS Tuning & Readahead 256KB ── Sintonização fina de I/O sem sobrecarregar pagecache
├── Google TCP BBRv3 + fq/pacing ── Menor latência e maior vazão no 5G e Wi-Fi sob tráfego concorrente
├── THP em madvise ──────────────── Páginas grandes transparentes sob demanda (sem fragmentação)
├── ARM64 Crypto Acelerado ──────── Instruções ARMv8 CE para AES, SHA2 e ChaCha20
└── Elevador mq-deadline ────────── Menor sobrecarga de CPU comparado ao BFQ tradicional

★★ (BAIXA PRIORIDADE - AJUSTES MARGINAIS)
├── Redução de Logspam Samsung ──── Menos ruído de SEC_DEBUG e IRQ no dmesg
└── Latência CFS Balanceada ─────── Redução de context switching excessivo em fair.c

❌ (ANTI-PATTERNS & PLACEBOS EXCLUÍDOS)
├── use_spi_crc = 0 ─────────────── Irrelevante para UFS interno (hack de MMC SPI de 2014)
├── Hacks de alinhamento 8/16B ──── Sem evidência prática; risco de corrupção de estrutura
├── -O3 / -Ofast / Polly ────────── Code bloat, instabilidade de vetorização e perda de cache L1/L2
├── Overclock / Desativação Térmica Destrói a bateria e gera desligamentos por PMIC
└── Desativação insegura de fsync ─ Perda garantida de dados e corrupção de partição
```

---

## 🏛️ Arquitetura Detalhada por Módulo

### 1. 🧠 Memória, MGLRU, ZRAM & LMKD

Em um aparelho de 4GB a 6GB de RAM, o maior inimigo da fluidez não é a velocidade bruta do processador, mas sim o **ciclo vicioso de pressão de memória**:
$$\text{App Aberto} \longrightarrow \text{Pressão de RAM} \longrightarrow \text{Reclaim Agressivo} \longrightarrow \text{ZRAM Thrashing} \longrightarrow \text{Refault} \longrightarrow \textbf{Congelamento de UI}$$

* **MGLRU (Multi-Gen LRU)**:
  * Ativação: `CONFIG_LRU_GEN=y` e `CONFIG_LRU_GEN_ENABLED=y`.
  * Divide páginas ativas e inativas em múltiplas gerações temporais, identificando com precisão cirúrgica quais páginas realmente não são usadas, reduzindo o custo de scanning do kswapd pela metade.
* **ZRAM Avançada**:
  * `ra_pages = 0`: Desativa leitura antecipada. A ZRAM é uma memória comprimida na RAM; readahead gerava desperdício de ciclos descompactando blocos desnecessários.
  * `vm.page-cluster = 0`: Permite troca de páginas individuais (1 por vez) em vez de lotes de $2^3$ páginas, maximizando a eficiência de descompressão.
  * Algoritmo: **LZ4** como padrão para menor latência em jogos; **ZSTD** configurável para aparelhos de 4GB que necessitem de maior razão de compressão.
  * `vm.swappiness = 130`: Envia páginas anônimas frias para a ZRAM antes de descartar cache de arquivos essenciais da interface (ícones, layouts e APKs mapeados).
* **Limites de Dirty Writeback (`mm/page-writeback.c`)**:
  * `dirty_background_ratio = 10` e `vm_dirty_ratio = 40`. Despeja gravações pendentes em lotes menores e frequentes no UFS, evitando travamentos de I/O em downloads e updates.
* **PSI (Pressure Stall Information) + LMKD**:
  * Monitoramento real via `/proc/pressure/memory` e `ro.lmk.use_psi=true`. O `lmkd` toma decisões preditivas de encerramento de processos em background antes que ocorra um congelamento visível na interface.
* **THP (Transparent Huge Pages) em `madvise`**:
  * `CONFIG_TRANSPARENT_HUGEPAGE_MADVISE=y`. Jamais `always` (que fragmenta a RAM em aparelhos de 4GB). Permite que emuladores e o runtime do Android solicitem páginas de 2MB quando estritamente benéfico.

---

### 2. ⚡ CPU, Samsung EMS, UCLAMP & Cgroups

O Exynos 1330 possui **2 núcleos Big (Cortex-A78)** e **6 núcleos LITTLE (Cortex-A55)**. Um escalonamento eficiente precisa garantir que o trabalho pesado vá para os núcleos A78 instantaneamente, sem manter esses núcleos acordados desnecessariamente.

* **Samsung EMS (Energy-aware Multi-processing Scheduler)**:
  * Preservação do subsistema proprietário da Samsung integrado ao EAS/EGO.
  * **Fix de cgroup `dex2oat`**: O Android cria a cgroup `dex2oat` entre `system-background` e `nnapi-hal`. O código stock da Samsung omitia essa entrada em `tune.c` e `ems.h`, deslocando o índice de prioridade de câmera e IA. Correção aplicada na íntegra.
  * **Heavy Task Boost no Root Cgroup**: Tarefas rodando no grupo raiz (como decodificadores de vídeo por software VP9/AV1 no `media.swcodec`) agora qualificam para o boost de CPU do EMS.
  * **Governor `energy_aware` no Boot**: Redirecionamento da escrita inicial do `init` de `schedutil` para `energy_aware` com tempo de resposta ágil de 4ms.
* **UCLAMP (Utilization Clamping) por Cgroup**:
  * `uclamp.min` elevado durante carga em `top-app` (jogos e renderização de tela), evitando a inércia de subida de frequência sem necessidade de forçar a CPU em 2.4 GHz permanentemente.
  * `uclamp.min` baixo em `background` para retenção estrita nos núcleos A55 de baixo consumo.
* **Topologia e Cpusets Rígidos**:
  * `top-app` (UI e Jogos): Acesso irrestrito a todos os núcleos (CPUs 0 a 7).
  * `foreground`: CPUs 0 a 7 sob demanda do escalonador.
  * `background`: Restrito prioritariamente aos núcleos LITTLE (CPUs 0 a 5), impedindo que tarefas secundárias acordem os núcleos A78.
* **ARM64 LSE Atomics (`CONFIG_ARM64_LSE_ATOMICS=y`)**:
  * Habilita instruções atômicas de hardware ARMv8.1+ (`LDADD`, `CAS`, etc.), acelerando sincronizações de mutex no Android Runtime e no Wine.
* **Calibração de Latência CFS (`fair.c`)**:
  * Restauração de `sysctl_sched_latency` para 6ms e granularidade para 0.75ms, eliminando troca excessiva de contexto (*context switching*).

---

### 3. 🎮 Gaming, Emulação de PC (Winlator / Mobox) & Virtualização

* **NTSync Nativo (`drivers/misc/ntsync.c`)**:
  * Primitivas de sincronização direta do Windows NT expostas pelo kernel. Reduz brutalmente o gargalo de sincronização de threads em jogos de Windows rodando via Wine / Proton / Winlator.
* **`futex_waitv` / Futex2**:
  * Permite que um processo aguarde eficientemente por múltiplos futexes simultaneamente com uma única syscall.
* **Suporte a Namespaces, Cgroups e OverlayFS**:
  * Kernel compatível com DroidSpaces, chroots de Linux e camadas de virtualização leve.
* **Drivers de Rede Virtual (TUN/TAP e VETH)**:
  * Essenciais para emuladores, VPNs de baixa latência e ambientes de container.
* **Sistemas de Arquivos Estendidos**:
  * Suporte nativo compilado no kernel a **NTFS3** e **exFAT** para leitura rápida de pendrives, SSDs externos e cartões SD formatados em PC.

---

### 4. 💽 Armazenamento UFS & I/O Real

* **Plataforma UFS Preservada**:
  * O A146B opera com UFS 2.2 (`fstab.s5e8535.ufs.ab`). O driver de UFS stock da Samsung é mantido intacto.
* **Readahead UFS Calibrado em 256 KB**:
  * Reduz a leitura antecipada exagerada de 2MB, poupando espaço no cache de páginas.
* **Elevador de I/O `mq-deadline`**:
  * Configurado no bootline (`elevator=mq-deadline`). Apresenta menor sobrecarga de CPU e menor latência de serviço em unidades flash do que escalonadores complexos como o BFQ.
* **F2FS Congestion Tuning Conservador**:
  * Sintonização de congestionamento na partição `/data` sem desativar barreiras nem fsync, mantendo integridade total de dados em caso de queda de energia.

---

### 5. 🔋 Bateria, Térmica & Carregamento

* **Bypass Charging Real no Chip Silicon Mitus SM5714 (`Bypass_charging_fix.patch`)**:
  * Permite alimentar o hardware diretamente pela fonte USB quando conectado ao carregador, sem forçar corrente de recarga na bateria durante jogos pesados.
* **Proteções Térmicas e SSRM Intactos**:
  * Monitoramento térmico oficial da Samsung preservado. Nenhuma modificação em trip points ou desativação de throttling — estabilidade do chassi e integridade da bateria em primeiro lugar.
* **DVFS Mali-G68 Preservado**:
  * Curvas de voltagem e clock da GPU controladas com segurança dentro das especificações de fábrica.

---

### 6. 🌐 Rede & Conectividade

* **Google TCP BBRv3 (`tcp_bbr3.c`)**:
  * Backport do algoritmo BBR de 3ª geração do Google para o Linux 5.15. Reduz o bufferbloat, estabiliza o ping em jogos online e garante maior vazão sob redes congestionadas 5G e Wi-Fi.
* **TCP Cubic Fallback**:
  * Mantido disponível como alternativa padrão de compatibilidade.
* **Pacing `fq` & TCP Fast Open**:
  * Otimização de janelas de transmissão de pacotes para menor latência em conexões HTTPS e streaming.

---

### 7. 🛡️ Root, Segurança & Anti-Detecção

* **RKSU v3.0.0 (Legacy Hooks v2)**:
  * Root nativo integrado via Syscall Hook, garantindo máxima estabilidade e compatibilidade.
* **Kprobes Desativado (`CONFIG_KPROBES=n`)**:
  * Evita detecção por verificação de símbolos de probe dinâmico.
* **SUSFS v1.5.5 / v2.3.0 & Zeromount**:
  * Ocultação profunda de montagens de root e módulos de sistema, garantindo aprovação em testes de Play Integrity (Device / Strong) e compatibilidade com aplicativos bancários.
* **SELinux Enforcing Preservado**:
  * Mantém o isolamento de sandboxing do Android 15 ativo, preservando a segurança contra vulnerabilidades em aplicativos de terceiros.
* **Samsung FIVE Desativado (`CONFIG_FIVE=n`)**:
  * Elimina a verificação de assinatura da Samsung a cada `exec()` e `mmap()`, acelerando o lançamento inicial de apps.

---

### 8. 🧹 Limpeza de Release (Sem Bloat de Depuração)

* **Instrumentação de Debug Desligada**:
  * `CONFIG_KASAN=n`, `CONFIG_KFENCE=n`, `CONFIG_LOCKDEP=n`, `CONFIG_DEBUG_PAGEALLOC=n`.
* **Restauração de Blocos de 2MB no MMU (`rodata=on`)**:
  * Sem o KASAN/KFENCE dividindo a memória linear em páginas de 4KB, o kernel volta a usar blocos contíguos de 2MB, liberando **~7.4MB de RAM gastos com page tables** e acelerando os acertos de cache TLB.
* **Supressão de Logspam**:
  * Redução drástica das mensagens contínuas de SEC_DEBUG, ACPM e IRQ no `dmesg`, sem desativar o `printk` (diagnóstico de crashes mantido).

---

## 🎭 Os Três Perfis de Sistema (EMS System Profiles)

A arquitetura do kernel prevê suporte à parametrização em tempo de execução para três casos de uso:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        BALANCED (Perfil Padrão)                        │
│  • Núcleos A55 prioritários; A78 escalonados sob demanda              │
│  • Boost de 4ms com decaimento suave                                   │
│  • Background rigorosamente limitado aos núcleos LITTLE                │
│  • Swappiness 130 + ZRAM LZ4 + MGLRU ativo                             │
└────────────────────────────────────────────────────────────────────────┘
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         ▼                                                   ▼
┌──────────────────────────────────┐┌────────────────────────────────────┐
│         GAMING (Desempenho)      ││         BATTERY (Economia)         │
│ • UCLAMP_MIN elevado no top-app  ││ • UCLAMP_MAX rebaixado em bg       │
│ • Escalonamento agressivo no A78 ││ • Núcleos A78 desativados em idle  │
│ • NTSync ativo para emuladores   ││ • Boost de frequência minimizado   │
│ • Bypass Charging habilitado     ││ • Suspensão profunda agressiva     │
└──────────────────────────────────┘└────────────────────────────────────┘
```

---

## 📚 Fontes e Repositórios de Referência

O desenvolvimento e backport dos patches deste projeto baseiam-se em referências técnicas consolidadas:

* **[Android Common Kernel (AOSP)](https://android.googlesource.com/kernel/common/)** (`android13-5.15` / `android14-5.15`): Referência máxima para implementação oficial do MGLRU e correções da Google.
* **[WildKernels](https://github.com/WildKernels)**: Referência em backports modernos de NTSync, KernelSU, SUSFS e BBRv3 para kernels 5.15 GKI.
* **[Project-24 (MrPankaj24)](https://github.com/MrPankaj24/Project-24)**: Base de automação de compilação, bypass charging e suíte Exynos 1330.
* **[langsdorffkernel (Clangsdorff)](https://github.com/clangsdorff/langsdorffkernel)**: Engenharia reversa dos subsistemas proprietários Samsung EMS, TrustZone e cgroups.
* **[KTweak (tytydraco)](https://github.com/tytydraco/KTweak)**: Biblioteca conceitual de sintonia de latência de scheduler e I/O.
* **[Crocus-Vernus (DEMONNICA)](https://github.com/DEMONNICA/Crocus-Vernus)**: Referência para parâmetros sysctl de VM e ZRAM em kernels recentes.
* **[Trinity (kanaodnd)](https://github.com/kanaodnd/Trinity)**: Lógica de controle de rajada de frequência (*boost & decay*).

---

## 🛠️ Toolchain & Instruções de Build

### Configuração da Toolchain
* **Compilador Principal**: AOSP Clang `r487747c` ou `r522817`
* **Cross-Compiler Host**: `aarch64-linux-gnu-gcc`
* **Nível de Otimização**: `-O2` (determinístico e estável)
* **Linker**: LLD (`ld.lld`) com **ThinLTO / Full LTO** (`ARCH_SUPPORTS_LTO_CLANG_FULL=y`)
* **Aceleração**: `ccache` com 12 núcleos paralelos na VM Spot Google Cloud

### Comandos de Compilação
```bash
# 1. Definir variáveis de ambiente da toolchain
export ARCH=arm64
export SUBARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export CC=clang
export CLANG_TRIPLE=aarch64-linux-gnu-
export LTO=thin

# 2. Carregar o defconfig da plataforma Exynos 1330
make s5e8535-a14xxx_defconfig

# 3. Compilar o kernel gerando a imagem de boot
make -j12 Image.gz-dtb
```

---

## 📥 Como Instalar no Dispositivo

1. Baixe o pacote compilado (`.tar` para instalação via Odin ou `boot.img` para TWRP).
2. Coloque o Galaxy A14 5G em **Download Mode** (desligue o aparelho, segure `Volume Up` + `Volume Down` e conecte o cabo USB ao PC).
3. No **Odin3** ou **Brokkr**, insira o arquivo `.tar` no slot **AP**.
4. Clique em **Start** e aguarde o dispositivo reiniciar.
5. *(Via TWRP)*: Se possuir recovery customizado, instale a imagem diretamente selecionando a partição **Boot**.
