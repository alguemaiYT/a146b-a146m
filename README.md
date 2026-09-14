# Samsung Galaxy A14 5G (SM-A146B) Custom Kernel
### Exynos 1330 (`s5e8535` / `a14x`) • Android 15 (Linux 5.15) • Configuration: **Balanced V1**

Kernel de alta fidelidade e estabilidade para o **Galaxy A14 5G (`SM-A146B`)**, baseado na árvore oficial do **[physwizz/a146b-a146m](https://github.com/physwizz/a146b-a146m)**. 

Este projeto adota uma engenharia orientada à plataforma real do dispositivo: **armazenamento UFS interno (`fstab.s5e8535.ufs.ab`)**, compatibilidade estrita de ABI com módulos vendor, **SELinux Enforcing**, mitigação de overhead de depuração da Samsung e aceleração nativa para emulação e jogos (com **NTSync** e **BBRv3**), eliminando placebos e modificações inseguras.

---

## 📱 Especificações da Plataforma

| Parâmetro | Detalhes Técnicos |
| :--- | :--- |
| **Dispositivo Alvo** | Samsung Galaxy A14 5G (`SM-A146B` / `SM-A146M`) |
| **SoC / Plataforma** | Samsung Exynos 1330 (`s5e8535` / `TARGET_SOC=s5e8535`) |
| **Topologia de CPU** | 8 núcleos (2x Cortex-A78 @ 2.4 GHz + 6x Cortex-A55 @ 2.0 GHz) |
| **GPU** | ARM Mali-G68 MP2 (Arquitetura Valhall) |
| **Armazenamento** | **UFS 2.2** (Não eMMC legado) |
| **Kernel Base** | Linux `5.15.180` (Android 15 / One UI 7) |
| **Defconfig de Base** | `erd8535_t_gki_defconfig` / `s5e8535-a14xxx_defconfig` |
| **Árvore Upstream** | `physwizz/a146b-a146m` (branch `main`) |

---

## 🏗️ Configuração “Balanced V1”

A configuração **Balanced V1** foi desenhada para extrair máxima fluidez, retenção de multitarefa e estabilidade térmica sem comprometer a durabilidade dos dados ou a integridade do hardware.

```
BALANCED V1 ARCHITECTURE
├── BASE & ABI
│   ├── physwizz A146B base funcional
│   ├── DTB/DTBO original s5e8535 mantido
│   ├── Módulos vendor originais preservados (sem quebra de ABI)
│   ├── SELinux Enforcing ativo
│   └── RKSU v3.0.0 (Legacy Hooks v2) + SUSFS compatível
│
├── MEMÓRIA & ZRAM
│   ├── ZRAM ativa com algoritmo LZ4 (prioridade performance)
│   ├── ra_pages = 0 (fim da descompressão inútil de páginas vizinhas)
│   ├── vm.swappiness = 130 (retenção agressiva de pagecache da UI)
│   ├── vm.page-cluster = 0 (otimizado para operações por página na ZRAM)
│   ├── dirty_background_ratio = 10 (despejo suave de escrita)
│   └── dirty_ratio = 40 (prevenção de congelamentos de I/O)
│
├── ESCALONADOR & CPU
│   ├── Samsung EMS (Energy-aware Multi-processing Scheduler) original
│   ├── EAS / EGO governor preservado e sintonizado para o s5e8535
│   ├── Port do cgroup dex2oat (alinhamento de prioridades de câmera e IA)
│   ├── Port de Heavy-Task Boost (boost responsivo em saturação)
│   └── Port de Boost para media.swcodec (decodificação de software VP9/AV1)
│
├── ARMAZENAMENTO & UFS
│   ├── Driver UFS stock preservado
│   ├── Readahead UFS calibrado em 256 KB
│   ├── Elevador I/O mq-deadline (baixa latência em flash)
│   └── F2FS congestion tuning conservador (sem hacks destrutivos de fsync)
│
├── REDE & CONECTIVIDADE
│   ├── Google TCP BBRv3 (backport oficial 5.15)
│   ├── TCP Cubic mantido como fallback
│   ├── Escalonamento fq / pacing ativo
│   └── TCP Fast Open habilitado
│
├── GAMING, EMULAÇÃO & VIRTUALIZAÇÃO (Winlator / Mobox)
│   ├── NTSync nativo (primitivas de sincronização NT no kernel)
│   ├── Futex / Futex2 preservados
│   ├── Namespaces completos, cgroups, OverlayFS
│   ├── Drivers virtuais de rede: TUN/TAP e VETH
│   └── Suporte a sistemas de arquivos NTFS3 e exFAT nativos
│
├── BATERIA & TÉRMICA
│   ├── Driver térmico e SSRM da Samsung intactos (segurança absoluta)
│   ├── DVFS da GPU Mali-G68 preservado
│   ├── Proteções de carga e saúde da bateria mantidas
│   └── Bypass Charging no chip SM5714 (alimentação direta sem aquecimento)
│
└── RELEASE CLEANUP (Sem Bloat de Depuração)
    ├── KASAN = n
    ├── KFENCE = n
    ├── LOCKDEP = n
    ├── DEBUG_PAGEALLOC = n
    ├── Redução de SEC_DEBUG e ACPM logspam
    └── Supressão de IRQ logspam contínuo
```

---

## 🛠️ Toolchain & Especificações de Compilação

* **Compilador**: Google AOSP Clang (`r487747c` / `r522817` – base Clang 17-19)
* **Cross-Tools Host**: `aarch64-linux-gnu-gcc` (para assembly/host fallback)
* **Linker**: LLD (`ld.lld`) com suporte a **ThinLTO / Full LTO**
* **Nível de Otimização**: `-O2` (deixando o compilador otimizar vetorizações de forma determinística)
* **Aceleração**: `ccache` com compilação paralela de 12 núcleos (`make -j12`) em VM Spot Google Cloud
* **Correções Clang**: Patch de compatibilidade `sizeof-pointer-memaccess` para Clang 20+

---

## 🚫 O que foi Intencionalmente Excluído (e por quê)

Para garantir estabilidade profissional, recusamos modificações inseguras ou placebos:

1. **`use_spi_crc = 0` / MMC hacks**:
   * O A146B utiliza barramento **UFS**, não eMMC legado por barramento SPI de 2014. Modificar registros de CRC de MMC não gera nenhum ganho de velocidade no chip UFS interno.
2. **Alignment Hacks 8B/16B Não Verificados**:
   * Forçar alinhamento artificial sem suporte estrutural nas chamadas do driver pode corromper estruturas em 64 bits.
3. **`-O3`, `-Ofast` e Polly**:
   * Aumentam o tamanho binário do kernel (*code bloat*), geram falhas sutis em pontes de assembly ARM64 e degradam acertos de cache L1/L2.
4. **Overclock e Modificação de Trip Points Térmicos**:
   * O Exynos 1330 opera em chassi sem câmara de vapor. Desativar throttling ou elevar limites térmicos acelera a degradação da bateria e causa desligamentos súbitos por PMIC.
5. **Hacks de Desativação de `fsync`**:
   * Quebram a durabilidade ACID do SQLite e F2FS, causando perda massiva de dados e corrupção do sistema em reinicializações inesperadas.
6. **Transplante Cego de Drivers do Exynos 850**:
   * Código de TrustZone e DVFS do Mali Bifrost feitos para o chip antigo de 8 núcleos A55 não se aplicam à arquitetura do Exynos 1330 (Cortex-A78 + Mali Valhall).

---

## 🗺️ Roadmap de Versões

* [x] **Balanced V1**: Base ultraestável, UFS tuning, NTSync, BBRv3, ZRAM LZ4 e correção de cgroups EMS.
* [ ] **Gaming V2**: Sintonização fina de DVFS para a GPU Mali-G68 e pisos mínimos de clock do EMS em sessões de jogo.
* [ ] **Battery V2**: Integração conservadora do Re:Kernel freezer e políticas de downclock em repouso profundo.

---

## 📥 Procedimento de Instalação

1. Obtenha o arquivo compilado (`.tar` para Odin ou `boot.img` para TWRP).
2. Conecte o aparelho em **Download Mode** (Volume Up + Volume Down conectados ao cabo USB).
3. No **Odin3** ou **Brokkr**, insira o `.tar` no slot **AP** e execute o flash.
4. *(Ou via TWRP)*: Instale a imagem diretamente na partição **Boot**.
