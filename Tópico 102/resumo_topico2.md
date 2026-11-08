# LPIC-1 — Tópico 102: Instalação do Linux e Gerenciamento de Pacotes

| Módulo | Assunto | Peso |
|---|---|---|
| 102.1 | Dimensionar partições de disco | 2 |
| 102.2 | Instalar o gerenciador de inicialização | 2 |
| 102.3 | Controle de bibliotecas compartilhadas | 1 |
| 102.4 | Utilização do sistema de pacotes Debian | 3 |
| 102.5 | Utilização do sistema de pacotes RPM e YUM | 3 |
| 102.6 | Linux virtualizado | 1 |

---

## 102.1: Dimensionar Partições de Disco

**Objetivo:** projetar um esquema de particionamento para um sistema Linux.

**Áreas de conhecimento:**
- alocar sistemas de arquivos e swap em partições ou discos separados;
- adaptar o projeto ao uso pretendido do sistema;
- garantir que `/boot` atenda aos requisitos de hardware/arquitetura para boot;
- conhecer os recursos básicos do LVM.

### Pontos de montagem principais

| Ponto de montagem | Conteúdo / motivo para separar |
|---|---|
| `/` (raiz) | Obrigatória. Todo o resto é montado a partir dela. |
| `/boot` | Kernel, initramfs e arquivos do GRUB. Pode ser separada por exigência do firmware/bootloader. |
| `/boot/efi` | Onde a **ESP** (EFI System Partition) é montada em sistemas UEFI. |
| `/home` | Dados dos usuários. Permite reinstalar o sistema sem perder os dados. |
| `/var` | Dados variáveis: logs, filas, caches, bancos de dados. Evita que o crescimento de logs encha a `/`. |
| `/tmp` | Arquivos temporários. Pode usar opções como `noexec` e `nosuid`. |
| `/usr`, `/opt`, `/srv` | Opcionais. Podem ser separadas conforme o uso. |
| swap | Não tem ponto de montagem (aparece como `none` ou `swap` no `fstab`). |

**Regra do FHS (muito cobrada):** `/etc`, `/bin`, `/sbin`, `/lib` e `/dev` precisam estar no mesmo sistema de arquivos da `/`, porque são necessários para o boot e para montar as demais partições. Portanto **`/etc` nunca é uma partição separada**. Já `/boot`, `/home`, `/var`, `/tmp`, `/opt` e `/usr` podem ser separadas.

### Partição `/boot`
- Guarda o kernel e o initramfs. O bootloader precisa lê-los **antes** de o sistema subir.
- Tradicionalmente fica em uma partição simples (ext2/3/4), fora de LVM e fora de criptografia, porque o bootloader precisa conseguir lê-la. O GRUB 2 já lê LVM, mas `/boot` separada e simples continua sendo a configuração conservadora.
- Motivo histórico para separar: limite do BIOS antigo para ler o kernel (limite de cilindros/posição no disco).

### ESP (EFI System Partition)
- Necessária para boot **UEFI**.
- Sistema de arquivos **FAT** (normalmente FAT32).
- Montada em `/boot/efi`.
- Guarda os executáveis `.efi` dos bootloaders (por exemplo `/boot/efi/EFI/<distro>/grubx64.efi`).
- Tipo no `gdisk`: **EF00**. Flag `esp` no `parted`.

### BIOS boot partition
- Usada quando o firmware é **BIOS (legacy)** mas o disco usa **GPT**. É uma pequena partição **sem sistema de arquivos** (cerca de 1 MiB) onde o GRUB 2 grava o `core.img`.
- Tipo no `gdisk`: **EF02**. Flag `bios_grub` no `parted`.
- Não confundir com a ESP.


> |Firmware |Tabela de partições |O que o GRUB precisa  
> |--------|--------------------|------------------|  
> |BIOS | MBR | Nada extra (usa o espaço após o MBR)|  
> |BIOS | GPT | BIOS boot partition (EF02, sem filesystem)|  
> |UEFI | GPT | ESP (FAT, montada em /boot/efi)|  

### Garantia de Inicialização do Sistema (Boot)

A arquitetura de hardware e firmware impõe limites ao particionamento de boot:

*   **Limitações de BIOS/Legacy:**
    *   Sistemas antigos exigiam que os arquivos de boot ficassem dentro dos primeiros **1024 cilindros** do disco (limite de ~8.4 GB).
    *   Ainda hoje, isolar o `/boot` evita problemas de compatibilidade caso a raiz (`/`) use sistemas de arquivos complexos não suportados nativamente pelo GRUB em estágio inicial.
*   **Sistemas UEFI:**
    *   Exigem obrigatoriamente a criação de uma partição do tipo **ESP (EFI System Partition)**.
    *   Deve ser formatada em **FAT32** (geralmente montada em `/boot/efi`).

### MBR × GPT (resumo para o projeto do disco)

| | MBR (msdos) | GPT |
|---|---|---|
| Tamanho máximo de disco | ~2 TiB (setores de 512 B) | Muito maior (praticamente ilimitado) |
| Partições | 4 primárias (ou 3 primárias + 1 estendida com lógicas) | 128 entradas por padrão |
| Firmware | BIOS (e UEFI em modo legado/CSM) | UEFI (e BIOS com BIOS boot partition) |
| Numeração das lógicas | A partir de 5 (`sda5`, `sda6`...) | Não existe conceito de lógica |

### Swap
- Pode ser **partição** ou **arquivo**.
- Regra tradicional: 1× a 2× a RAM. Hoje o tamanho depende do uso. Para **hibernação**, a swap deve ser **no mínimo do tamanho da RAM**.
- Criação e uso: `mkswap`, `swapon`, `swapoff`; consulta: `swapon --show`, `cat /proc/swaps`, `free`.
- Entrada típica no `/etc/fstab`: `UUID=xxxx none swap sw 0 0`.
- Tipo de partição: MBR `82`, GPT `8200` (Linux swap).

### Tipos de partição mais comuns

| Uso | MBR | GPT (gdisk) |
|---|---|---|
| Linux filesystem | `83` | `8300` |
| Linux swap | `82` | `8200` |
| Linux LVM | `8e` | `8E00` |
| ESP | `ef` | `EF00` |
| BIOS boot | — | `EF02` |

### LVM — recursos básicos

```
Discos/partições  →  PV (Physical Volume)
PVs               →  VG (Volume Group)
VG                →  LV (Logical Volume)  →  recebe o sistema de arquivos
```

- Os VGs são divididos em **extents**: PE (Physical Extent) e LE (Logical Extent). O padrão é 4 MiB.
- Comandos principais:

| Etapa | Criar | Exibir | Outros |
|---|---|---|---|
| PV | `pvcreate /dev/sdb1` | `pvdisplay`, `pvs` | `pvremove`, `pvmove` |
| VG | `vgcreate vg0 /dev/sdb1` | `vgdisplay`, `vgs` | `vgextend`, `vgreduce`, `vgremove` |
| LV | `lvcreate -L 10G -n lv_dados vg0` | `lvdisplay`, `lvs` | `lvextend`, `lvreduce`, `lvremove` |

- Caminhos do LV: `/dev/vg0/lv_dados` ou `/dev/mapper/vg0-lv_dados`.
- Aumentar um LV: `lvextend -L +5G /dev/vg0/lv_dados` e depois aumentar o sistema de arquivos (`resize2fs` para ext2/3/4, `xfs_growfs` para XFS). A opção `-r` (`--resizefs`) do `lvextend` faz as duas coisas.
- **XFS não pode ser reduzido**, apenas aumentado. ext4 pode ser reduzido (com o sistema desmontado).
- **Snapshots** permitem uma cópia instantânea do estado de um LV (`lvcreate -s`).
- Vantagem do LVM: redimensionar volumes sem depender de partições contíguas e adicionar discos ao VG.

### Projeto conforme o uso

| Cenário | Sugestão |
|---|---|
| Desktop | `/` + `/home` + swap (e ESP se UEFI) |
| Servidor de arquivos | Volume de dados separado (`/srv` ou `/home`), possivelmente com quotas |
| Servidor web/aplicação | `/var` separado (logs e conteúdo crescem) |
| Servidor de banco de dados | Volume separado para os dados (por exemplo `/var/lib/mysql`) |
| Servidor multiusuário | `/home`, `/tmp` e `/var` separados |

### Pegadinhas
- `/etc` **não** pode ser separado; `/home`, `/var`, `/boot` **podem**.
- ESP é **FAT**, não ext4.
- A ESP é montada em `/boot/efi`, não em `/boot`.
- BIOS + GPT exige **BIOS boot partition** (EF02) para o GRUB 2.
- Swap não é "montada": é **ativada** (`swapon`).
- XFS **não reduz**.

---

## 102.2: Instalar o Gerenciador de Inicialização

**Objetivo:** selecionar, instalar e configurar um gerenciador de inicialização.

**Áreas de conhecimento:**
- fornecer locais alternativos de boot e opções de boot de backup;
- instalar e configurar um bootloader como o **GRUB Legacy**;
- fazer alterações básicas de configuração do **GRUB 2**;
- interagir com o bootloader.

**Arquivos, termos e utilitários:** `menu.lst`, `grub.cfg` e `grub.conf`, `grub-install`, `grub-mkconfig`, MBR.

### Sequência de boot (visão do bootloader)

**BIOS:** firmware → lê o **MBR** do disco de boot → bootloader → kernel + initramfs → init/systemd.

**UEFI:** firmware lê as entradas de boot (guardadas na NVRAM) → carrega o executável `.efi` da **ESP** → kernel + initramfs → init/systemd. Entradas gerenciadas com `efibootmgr`. O caminho de fallback é `\EFI\BOOT\BOOTX64.EFI`.

### Estrutura do MBR (512 bytes)

| Parte | Tamanho |
|---|---|
| Código de boot (bootstrap) | 446 bytes |
| Tabela de partições (4 × 16) | 64 bytes |
| Assinatura de boot (`0x55AA`) | 2 bytes |

Backup e restauração apenas do código de boot, sem mexer na tabela de partições:

```bash
dd if=/dev/sda of=mbr.bak bs=512 count=1     # backup do MBR inteiro
dd if=mbr.bak of=/dev/sda bs=446 count=1     # restaura só o código (446 bytes)
```

### GRUB Legacy (GRUB 0.97)

- Configuração: **`/boot/grub/menu.lst`** (Debian) ou **`/boot/grub/grub.conf`** (Red Hat). Edição direta do arquivo, **sem comando para regenerar**.
- Não suporta GPT/UEFI de forma nativa.
- Estágios: **stage 1** (no MBR), **stage 1.5** (no espaço entre o MBR e a primeira partição) e **stage 2** (em `/boot/grub`).
- **Notação de dispositivos: disco e partição começam em 0.** `(hd0,0)` = primeira partição do primeiro disco (por exemplo `/dev/sda1`).

```
default 0
timeout 5
title Linux
    root (hd0,0)
    kernel /vmlinuz-5.4 ro root=/dev/sda2
    initrd /initrd.img-5.4
```

| Diretiva | Função |
|---|---|
| `default` | Entrada padrão (número, começando em 0) |
| `timeout` | Segundos antes de iniciar a entrada padrão |
| `title` | Nome da entrada no menu |
| `root` | Partição onde está o kernel (`/boot`) |
| `kernel` | Caminho do kernel e parâmetros |
| `initrd` | Caminho do initramfs |
| `fallback` | Entrada alternativa se a padrão falhar |
| `chainloader +1` | Passa o controle para outro bootloader (por exemplo Windows) |
| `password` | Protege o menu/edição com senha |

Instalação do GRUB Legacy:

```bash
grub-install /dev/sda          # instala no MBR do disco
# ou, no shell do GRUB:
grub> root (hd0,0)
grub> setup (hd0)
```

### GRUB 2

- Configuração gerada: **`/boot/grub/grub.cfg`** (Debian/Ubuntu) ou **`/boot/grub2/grub.cfg`** (RHEL/SUSE e derivados).
- **Não se edita o `grub.cfg` à mão** (ele é sobrescrito). A configuração é feita em:
  - **`/etc/default/grub`** (variáveis);
  - **`/etc/grub.d/`** (scripts executáveis; só entram na geração os scripts com permissão de execução).
- Depois de alterar, é obrigatório **regenerar** o `grub.cfg`:

| Distro | Comando |
|---|---|
| Debian/Ubuntu | `update-grub` (atalho para `grub-mkconfig -o /boot/grub/grub.cfg`) |
| Genérico | `grub-mkconfig -o /boot/grub/grub.cfg` |
| RHEL/Fedora/SUSE | `grub2-mkconfig -o /boot/grub2/grub.cfg` |

- **Notação de dispositivos: disco começa em 0 e partição começa em 1.** `(hd0,1)` = primeira partição do primeiro disco. Com o tipo de tabela: `(hd0,msdos1)` ou `(hd0,gpt1)`.

Variáveis mais comuns em `/etc/default/grub`:

| Variável | Função |
|---|---|
| `GRUB_DEFAULT` | Entrada padrão (número, título ou `saved`) |
| `GRUB_TIMEOUT` | Tempo do menu em segundos |
| `GRUB_TIMEOUT_STYLE` | `menu`, `countdown` ou `hidden` |
| `GRUB_CMDLINE_LINUX` | Parâmetros do kernel em **todas** as entradas |
| `GRUB_CMDLINE_LINUX_DEFAULT` | Parâmetros só nas entradas **normais** (não nas de recuperação) |
| `GRUB_DISABLE_OS_PROBER` | Controla a detecção de outros sistemas (em versões recentes pode vir desativado por padrão) |
| `GRUB_SAVEDEFAULT` | Grava a última entrada escolhida (exige `GRUB_DEFAULT=saved`) |

Scripts em `/etc/grub.d/`: `00_header`, `10_linux`, `30_os-prober`, `40_custom` (entradas personalizadas), entre outros. Os números definem a **ordem** de execução.

Instalação:

```bash
grub-install /dev/sda                                        # BIOS: no MBR do disco
grub-install --target=x86_64-efi --efi-directory=/boot/efi   # UEFI: na ESP
```

- Instala-se no **disco** (`/dev/sda`), não na partição (`/dev/sda1`), para boot pelo MBR.
- Em RHEL/SUSE o comando é `grub2-install`.

Outros comandos úteis:
- `grub-mkpasswd-pbkdf2`: gera o hash de senha para proteger o GRUB 2.
- `grub-set-default`: define a entrada padrão (exige `GRUB_DEFAULT=saved`).
- `grub-reboot`: escolhe a entrada **apenas para o próximo boot** (exige `GRUB_DEFAULT=saved`).

### Componentes do GRUB 2 em BIOS
`boot.img` (no MBR) → `core.img` (no espaço após o MBR ou na BIOS boot partition) → módulos em `/boot/grub/` → lê o `grub.cfg`.

### Interagindo com o bootloader (menu e linha de comando)

| Tecla no menu | Ação |
|---|---|
| `e` | Edita a entrada selecionada (temporário; não grava) |
| `c` | Abre a linha de comando do GRUB |
| `Ctrl+x` ou `F10` | Inicia com as alterações feitas |
| `Esc` | Cancela e volta ao menu |

Boot manual pela linha de comando (GRUB 2):

```
grub> ls
grub> set root=(hd0,1)
grub> linux /vmlinuz root=/dev/sda1 ro
grub> initrd /initrd.img
grub> boot
```

Se o GRUB não achar os arquivos, cai no **`grub rescue>`**. Recuperação típica: `ls`, `set root=...`, `set prefix=...`, `insmod normal`, `normal`.

Parâmetros de kernel comuns: `root=`, `ro`, `quiet`, `splash`, `nomodeset`, `init=`, `single` ou `1` (modo monousuário no SysV), `systemd.unit=rescue.target`.

### Locais alternativos e backup de boot
- Instalar o bootloader no **setor de boot de uma partição** em vez do MBR e usar chainloading.
- Boot por **CD/DVD/USB live** ou de recuperação.
- Reinstalar o GRUB a partir de um live/rescue: montar a raiz (e `/boot`, e a ESP se houver), fazer `bind` de `/dev`, `/proc` e `/sys`, `chroot` e rodar `grub-install` + `update-grub`.
- Manter uma entrada alternativa (`fallback` no Legacy) e backup do MBR com `dd`.

### Outros gerenciadores (apenas conhecimento)
SYSLINUX/ISOLINUX/EXTLINUX/PXELINUX, systemd-boot. O **LILO** não é mais cobrado.

### Pegadinhas
- **Legacy:** partição começa em **0**. **GRUB 2:** partição começa em **1**.
- Em Legacy edita-se `menu.lst`; em GRUB 2 **nunca** o `grub.cfg` diretamente.
- Depois de mexer em `/etc/default/grub` é preciso rodar `update-grub`/`grub-mkconfig`; sem isso nada muda.
- `grub-install` recebe o **disco**, não a partição.
- O arquivo é `grub.conf` no Red Hat e `menu.lst` no Debian (Legacy).
- `GRUB_CMDLINE_LINUX` vale para todas as entradas; `GRUB_CMDLINE_LINUX_DEFAULT`, só para as normais.

---

## 102.3: Controle de Bibliotecas Compartilhadas

**Objetivo:** determinar de quais bibliotecas compartilhadas os executáveis dependem e instalá-las quando necessário.

**Áreas de conhecimento:**
- identificar bibliotecas compartilhadas;
- identificar os locais típicos das bibliotecas do sistema;
- carregar bibliotecas compartilhadas.

**Arquivos, termos e utilitários:** `ldd`, `ldconfig`, `/etc/ld.so.conf`, `LD_LIBRARY_PATH`.

### Conceitos
- **Biblioteca compartilhada (dinâmica):** `.so` (*shared object*), carregada em tempo de execução e usada por vários programas. Economiza memória e disco e facilita atualizações.
- **Biblioteca estática:** `.a`, incorporada ao executável na compilação.
- **Carregador dinâmico:** `ld.so` / `ld-linux.so`, que localiza e carrega as bibliotecas ao iniciar o programa.

### Nomes
```
libexemplo.so.1.2.3   → arquivo real (versão completa)
libexemplo.so.1       → soname (link simbólico; o que o programa pede)
libexemplo.so         → linker name (link simbólico; usado na compilação)
```

### Locais típicos
`/lib`, `/lib64`, `/usr/lib`, `/usr/lib64`, `/usr/local/lib`. Em Debian com multiarch: `/lib/x86_64-linux-gnu`, `/usr/lib/x86_64-linux-gnu`.

`/lib` e `/usr/lib` são sempre pesquisados por padrão, mesmo que não estejam listados em `ld.so.conf`.

### `ldd` — listar dependências
```bash
ldd /bin/ls
# linux-vdso.so.1 (0x00007ffc...)
# libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f...)
# libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
```
- `=> not found` indica biblioteca **ausente**.
- **Cuidado:** `ldd` pode executar o programa em alguns casos; **não use em binários não confiáveis**.
- Alternativas: `objdump -p programa | grep NEEDED`, `readelf -d programa`.

### `ldconfig` — atualizar o cache
- Lê `/etc/ld.so.conf` (e `/etc/ld.so.conf.d/*.conf`) e os diretórios padrão, **recria o cache `/etc/ld.so.cache`** e atualiza os links simbólicos (sonames).
- Precisa ser executado como **root** depois de instalar uma biblioteca em um diretório novo ou de alterar a configuração.

| Opção | Função |
|---|---|
| `ldconfig` | Atualiza o cache |
| `ldconfig -p` | **Lista** o conteúdo do cache (`ldconfig -p \| grep libexemplo`) |
| `ldconfig -v` | Modo verboso |
| `ldconfig -n DIR` | Processa apenas o diretório informado (sem atualizar o cache) |

### `/etc/ld.so.conf`
- Lista de diretórios extras para procurar bibliotecas, um por linha.
- Normalmente contém `include /etc/ld.so.conf.d/*.conf`; cada arquivo `.conf` nesse diretório lista diretórios.

```bash
echo "/opt/app/lib" > /etc/ld.so.conf.d/app.conf
ldconfig
```

### `LD_LIBRARY_PATH`
- Variável de ambiente com diretórios separados por `:`, pesquisados **antes** do cache.
- Efeito **temporário**, por processo/sessão.
- **Ignorada em programas setuid/setgid** (segurança).

```bash
LD_LIBRARY_PATH=/opt/app/lib ./programa
export LD_LIBRARY_PATH=/opt/app/lib:$LD_LIBRARY_PATH
```

**Ordem de busca do carregador (resumo):**
1. `DT_RPATH` do executável (se não houver `DT_RUNPATH`)
2. `LD_LIBRARY_PATH`
3. `DT_RUNPATH` do executável
4. `/etc/ld.so.cache`
5. Diretórios padrão (`/lib`, `/usr/lib`)

`LD_PRELOAD` e `/etc/ld.so.preload` forçam o carregamento de uma biblioteca **antes** das demais.

### Erro clássico
```
error while loading shared libraries: libfoo.so.1: cannot open shared object file: No such file or directory
```
Solução: `ldd programa` para ver o que falta → instalar o pacote que contém a biblioteca → se está em diretório fora do padrão, adicionar a `ld.so.conf.d/` e rodar `ldconfig` (ou usar `LD_LIBRARY_PATH` como solução temporária).

### Pegadinhas
- Alterou `ld.so.conf`? É preciso rodar **`ldconfig`**.
- `ldconfig -p` **imprime** o cache; `ldconfig` sem opção **atualiza** o cache.
- `LD_LIBRARY_PATH` é temporária; `ld.so.conf` + `ldconfig` é permanente.
- `ldd` mostra dependências; **não** instala nada.

---

## 102.4: Utilização do Sistema de Pacotes Debian

**Objetivo:** gerenciar pacotes usando as ferramentas Debian.

**Áreas de conhecimento:**
- instalar, atualizar e remover pacotes binários Debian;
- encontrar pacotes que contêm arquivos ou bibliotecas específicos, instalados ou não;
- obter informações de pacotes: versão, conteúdo, dependências, integridade e status de instalação;
- conhecer o `apt`.

**Arquivos, termos e utilitários:** `/etc/apt/sources.list`, `dpkg`, `dpkg-reconfigure`, `apt-get`, `apt-cache`.

### Visão geral

| Ferramenta | Nível | Resolve dependências? | Usa repositórios? |
|---|---|---|---|
| `dpkg` | Baixo nível; trabalha com arquivos `.deb` locais | **Não** (só informa o problema) | Não |
| `apt-get` / `apt-cache` / `apt` | Alto nível | **Sim** | Sim |

- Arquivo `.deb`: nome no formato `nome_versão_arquitetura.deb`. É um arquivo `ar` contendo `debian-binary`, `control.tar.*` e `data.tar.*`.
- Banco de dados do `dpkg`: `/var/lib/dpkg/` (`status`, `available`, `info/`).

### `dpkg`

| Comando | Função |
|---|---|
| `dpkg -i pacote.deb` | Instala (ou atualiza) um `.deb` |
| `dpkg -r pacote` | Remove, **mantém** arquivos de configuração |
| `dpkg -P pacote` | **Purge**: remove inclusive as configurações |
| `dpkg -l [padrão]` | Lista pacotes e seus estados |
| `dpkg -L pacote` | Lista os arquivos **instalados** pelo pacote |
| `dpkg -S /caminho/arquivo` | Descobre **a qual pacote instalado** o arquivo pertence |
| `dpkg -s pacote` | Mostra status e informações do pacote instalado |
| `dpkg -I pacote.deb` | Mostra informações de um `.deb` (não instalado) |
| `dpkg -c pacote.deb` | Lista o conteúdo de um `.deb` |
| `dpkg -V pacote` | Verifica a integridade dos arquivos instalados (compara com os checksums do pacote) |
| `dpkg --configure -a` | Termina a configuração de pacotes deixados pela metade |
| `dpkg -x pacote.deb dir` | Extrai o conteúdo do `.deb` para um diretório |
| `dpkg --get-selections` | Lista pacotes e suas marcas de seleção |
| `dpkg --add-architecture arch` | Habilita uma arquitetura adicional (multiarch) |

**Estados em `dpkg -l`** (duas ou três letras no início):

| Código | Significado |
|---|---|
| `ii` | Instalado corretamente |
| `rc` | Removido, mas com **arquivos de configuração remanescentes** |
| `un` | Desconhecido/não instalado |
| `pn` | Purgado (não instalado) |

### `dpkg-reconfigure`
Reexecuta as perguntas de configuração (debconf) de um pacote **já instalado**.

```bash
dpkg-reconfigure tzdata
dpkg-reconfigure locales
dpkg-reconfigure keyboard-configuration
dpkg-reconfigure -p low pacote     # define a prioridade das perguntas
```

### `/etc/apt/sources.list`
Define os repositórios. Também são lidos arquivos em `/etc/apt/sources.list.d/` (`.list`; em versões novas também o formato deb822 `.sources`).

```
deb http://deb.debian.org/debian bookworm main contrib non-free
deb-src http://deb.debian.org/debian bookworm main
```

- `deb` → pacotes **binários**; `deb-src` → pacotes de **código-fonte**.
- Formato: `tipo URI distribuição componente(s)`.
- Componentes: Debian → `main`, `contrib`, `non-free` (nas versões recentes também `non-free-firmware`). Ubuntu → `main`, `restricted`, `universe`, `multiverse`.
- **Depois de editar o `sources.list`, rode `apt-get update`.**

### `apt-get` (alto nível, instalação e atualização)

| Comando | Função |
|---|---|
| `apt-get update` | Baixa/atualiza o **índice** dos repositórios (não instala nada) |
| `apt-get upgrade` | Atualiza os pacotes instalados **sem remover** nenhum pacote e sem instalar pacotes novos |
| `apt-get dist-upgrade` | Atualiza lidando com mudanças de dependência (pode instalar e remover pacotes); equivalente a `full-upgrade` no `apt` |
| `apt-get install pacote` | Instala (com dependências) |
| `apt-get install --reinstall pacote` | Reinstala |
| `apt-get install pacote=versão` | Instala uma versão específica |
| `apt-get remove pacote` | Remove, **mantém** a configuração |
| `apt-get purge pacote` | Remove **e apaga** a configuração |
| `apt-get autoremove` | Remove dependências que não são mais necessárias |
| `apt-get clean` | Apaga **todos** os `.deb` do cache (`/var/cache/apt/archives/`) |
| `apt-get autoclean` | Apaga só os `.deb` **obsoletos** (que não podem mais ser baixados) |
| `apt-get -f install` | Tenta **corrigir** dependências quebradas |
| `apt-get source pacote` | Baixa o código-fonte |
| `apt-get build-dep pacote` | Instala dependências de compilação |
| `apt-get download pacote` | Só baixa o `.deb` para o diretório atual |
| `apt-get check` | Verifica dependências quebradas |

Opções úteis: `-y` (assume sim), `-s` / `--simulate` (simula), `-d` (só baixa).

### `apt-cache` (consulta ao índice local)

| Comando | Função |
|---|---|
| `apt-cache search termo` | Pesquisa pacotes por nome/descrição |
| `apt-cache show pacote` | Mostra informações do pacote |
| `apt-cache policy pacote` | Versão instalada, versão candidata e origens |
| `apt-cache depends pacote` | Dependências do pacote |
| `apt-cache rdepends pacote` | Quem **depende** do pacote |
| `apt-cache showpkg pacote` | Informações detalhadas (versões, dependências reversas) |
| `apt-cache pkgnames` | Lista os nomes de todos os pacotes |
| `apt-cache stats` | Estatísticas do cache |

O `apt-cache` consulta o **índice local**; para ele refletir o que há nos repositórios, rode antes `apt-get update`.

### `apt` (conhecimento)
Interface mais amigável e interativa que combina funções do `apt-get` e do `apt-cache`.

```bash
apt update
apt upgrade
apt full-upgrade
apt install pacote
apt install ./pacote.deb        # instala .deb local resolvendo dependências
apt remove pacote
apt purge pacote
apt autoremove
apt search termo
apt show pacote
apt list --installed
apt list --upgradable
```

### Como encontrar o pacote de um arquivo

| Situação | Ferramenta |
|---|---|
| Arquivo de um pacote **já instalado** | `dpkg -S /caminho/arquivo` |
| Arquivo de um pacote **não instalado** | `apt-file search /caminho/arquivo` (exige `apt-file update` e o pacote `apt-file`) |
| Listar arquivos de um pacote não instalado | `apt-file list pacote` |

### Outros pontos
- Configuração do APT: `/etc/apt/apt.conf` e `/etc/apt/apt.conf.d/`; preferências/pinning em `/etc/apt/preferences`.
- Segurança: os repositórios são verificados por chaves GPG (chaves em `/etc/apt/trusted.gpg.d/` ou em keyrings referenciados no `sources.list`).
- Evitar atualização de um pacote: `apt-mark hold pacote`.
- Instalar `.deb` local com dependências: `apt install ./arquivo.deb`. Se usou `dpkg -i` e faltaram dependências, rode `apt-get -f install`.

### Pegadinhas
- `dpkg` **não** resolve dependências; `apt-get` resolve.
- `remove` **mantém** configuração; `purge` **apaga**.
- `apt-get update` **só atualiza o índice**; `apt-get upgrade` é que atualiza pacotes.
- `dpkg -S` só encontra arquivos de pacotes **instalados**; para pacotes não instalados use `apt-file`.
- `dpkg -L` lista arquivos de um pacote; `dpkg -S` faz o caminho inverso (arquivo → pacote).
- `clean` apaga todo o cache; `autoclean` apaga só o obsoleto.
- Estado `rc` = removido com configuração remanescente.

---

## 102.5: Utilização do Sistema de Pacotes RPM e YUM

**Objetivo:** gerenciar pacotes usando **RPM, YUM e Zypper**.

**Áreas de conhecimento:**
- instalar, reinstalar, atualizar e remover pacotes com RPM, YUM e Zypper;
- obter informações de pacotes RPM: versão, status, dependências, integridade e assinaturas;
- determinar quais arquivos um pacote fornece e de qual pacote um arquivo vem;
- conhecer o **dnf**.

**Arquivos, termos e utilitários:** `rpm`, `rpm2cpio`, `/etc/yum.conf`, `/etc/yum.repos.d/`, `yum`, `zypper`.

### Visão geral

| Ferramenta | Nível | Resolve dependências? |
|---|---|---|
| `rpm` | Baixo nível; trabalha com arquivos `.rpm` e com o banco local | **Não** |
| `yum` / `dnf` | Alto nível (Red Hat, CentOS, Fedora, Alma, Rocky) | **Sim**, via repositórios |
| `zypper` | Alto nível (SUSE/openSUSE) | **Sim**, via repositórios |

- Nome do arquivo: `nome-versão-release.arquitetura.rpm` (por exemplo `bash-5.1.8-6.el9.x86_64.rpm`).
- Banco de dados RPM: `/var/lib/rpm` (em distribuições mais recentes pode estar em `/usr/lib/sysimage/rpm`).

### `rpm` — instalação e remoção

| Comando | Função |
|---|---|
| `rpm -i pacote.rpm` | Instala |
| `rpm -ivh pacote.rpm` | Instala com saída verbosa (`-v`) e barra de progresso (`-h`) |
| `rpm -Uvh pacote.rpm` | **Upgrade**: atualiza, ou **instala** se ainda não existir |
| `rpm -Fvh pacote.rpm` | **Freshen**: atualiza **apenas se já estiver instalado** |
| `rpm -ivh --replacepkgs pacote.rpm` | **Reinstala** o mesmo pacote |
| `rpm -e pacote` | Remove (usa o **nome do pacote**, não do arquivo) |
| `rpm -Uvh --oldpackage pacote.rpm` | Permite instalar versão mais antiga (downgrade) |

Opções que merecem atenção: `--test` (simula), `--nodeps` (ignora dependências), `--force`, `--replacefiles`, `--nosignature`, `--nodigest`, `--prefix` / `--relocate`.

**Arquivos de configuração em upgrades:** quando a configuração foi modificada pelo usuário e o pacote traz uma nova, o RPM preserva uma das versões e grava a outra com sufixo **`.rpmnew`** (nova versão do pacote) ou **`.rpmsave`** (cópia da configuração antiga).

### `rpm` — consultas (`-q`)

| Comando | Função |
|---|---|
| `rpm -q pacote` | Verifica se está instalado e mostra a versão |
| `rpm -qa` | Lista **todos** os pacotes instalados |
| `rpm -qi pacote` | Informações do pacote |
| `rpm -ql pacote` | Lista os **arquivos** do pacote |
| `rpm -qc pacote` | Lista os arquivos de **configuração** |
| `rpm -qd pacote` | Lista a **documentação** |
| `rpm -qR pacote` | Lista as **dependências** (requisitos) |
| `rpm -q --provides pacote` | O que o pacote **fornece** |
| `rpm -q --whatprovides arquivo` | Qual pacote instalado fornece o recurso |
| `rpm -q --whatrequires pacote` | Quem **requer** o pacote |
| `rpm -qf /caminho/arquivo` | **Qual pacote instalado** contém o arquivo |
| `rpm -q --changelog pacote` | Histórico de mudanças |
| `rpm -q --scripts pacote` | Scripts de instalação/remoção |
| `rpm -qa --last` | Pacotes instalados, do mais recente ao mais antigo |

**Consultar um arquivo `.rpm` (ainda não instalado)** — acrescente **`p`**:
`rpm -qpi pacote.rpm`, `rpm -qpl pacote.rpm`, `rpm -qpR pacote.rpm`, `rpm -qpc pacote.rpm`.

### `rpm` — verificação e assinaturas

| Comando | Função |
|---|---|
| `rpm -V pacote` | Verifica arquivos do pacote instalado contra o banco RPM |
| `rpm -Va` | Verifica **todos** os pacotes |
| `rpm -K pacote.rpm` (ou `--checksig`) | Verifica **assinatura e checksum** do arquivo `.rpm` |
| `rpm --import chave.gpg` | Importa uma chave GPG pública |
| `rpm -qa gpg-pubkey*` | Lista as chaves importadas |

Saída do `rpm -V`: nove caracteres, onde cada letra indica uma diferença e um **ponto** indica que passou.

| Letra | Significado |
|---|---|
| `S` | Tamanho (Size) |
| `M` | Modo (permissões/tipo) |
| `5` | Digest MD5/checksum |
| `D` | Dispositivo (major/minor) |
| `L` | Link simbólico |
| `U` | Usuário (dono) |
| `G` | Grupo |
| `T` | Data de modificação (mTime) |
| `P` | Capabilities |

Um `c` antes do nome do arquivo indica **arquivo de configuração**. A palavra `missing` indica arquivo ausente. Sem nenhuma saída = tudo íntegro.

### `rpm2cpio` — extrair arquivos sem instalar
```bash
rpm2cpio pacote.rpm | cpio -idmv      # extrai (i), cria diretórios (d), preserva mtime (m), verboso (v)
rpm2cpio pacote.rpm | cpio -t         # apenas lista o conteúdo
```
A extração é feita no diretório atual, **sem instalar** o pacote.

### YUM

- **Arquivo de configuração global:** `/etc/yum.conf` (opções como `gpgcheck`, `keepcache`, `cachedir`, `logfile`, `exclude`, `installonly_limit`).
- **Repositórios:** arquivos **`/etc/yum.repos.d/*.repo`**.

```ini
[nome-do-repo]
name=Descrição do repositório
baseurl=http://servidor/repo/
# ou mirrorlist= / metalink=
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-exemplo
```

| Comando | Função |
|---|---|
| `yum install pacote` | Instala (com dependências) |
| `yum install ./pacote.rpm` (ou `yum localinstall`) | Instala `.rpm` local resolvendo dependências pelos repositórios |
| `yum reinstall pacote` | Reinstala |
| `yum remove pacote` (ou `erase`) | Remove |
| `yum update` | Atualiza todos os pacotes (ou o pacote indicado) |
| `yum upgrade` | Como `update`, mas também trata pacotes obsoletos |
| `yum check-update` | Lista atualizações disponíveis |
| `yum downgrade pacote` | Volta para versão anterior |
| `yum search termo` | Pesquisa em nome/resumo |
| `yum info pacote` | Informações do pacote |
| `yum list [installed \| available \| updates]` | Lista pacotes |
| `yum provides arquivo` (ou `whatprovides`) | **Qual pacote** fornece o arquivo (aceita curingas, por exemplo `*/bin/foo`) |
| `yum deplist pacote` | Dependências do pacote |
| `yum repolist [all]` | Lista os repositórios habilitados (ou todos) |
| `yum clean all` | Limpa o cache |
| `yum makecache` | Recria o cache de metadados |
| `yum history` | Histórico de transações (permite desfazer) |
| `yum groupinstall "Grupo"` | Instala um grupo de pacotes |

Opções: `-y`, `--enablerepo=repo`, `--disablerepo=repo`.

### DNF (conhecimento)
- Sucessor do YUM (Fedora e RHEL 8 em diante). Em RHEL 8+ o comando `yum` é um **atalho** para o `dnf`.
- Configuração: `/etc/dnf/dnf.conf`; os mesmos arquivos `.repo` em `/etc/yum.repos.d/`.
- Sintaxe praticamente igual à do YUM: `dnf install`, `dnf remove`, `dnf upgrade`, `dnf provides`, `dnf search`, `dnf repolist`...

### Zypper (SUSE/openSUSE)

| Comando | Atalho | Função |
|---|---|---|
| `zypper refresh` | `ref` | Atualiza os metadados dos repositórios |
| `zypper install pacote` | `in` | Instala |
| `zypper remove pacote` | `rm` | Remove |
| `zypper update` | `up` | Atualiza os pacotes instalados |
| `zypper dist-upgrade` | `dup` | Atualização de distribuição (versão/lançamento) |
| `zypper search termo` | `se` | Pesquisa |
| `zypper info pacote` | — | Informações do pacote |
| `zypper repos` | `lr` | Lista repositórios |
| `zypper addrepo URL nome` | `ar` | Adiciona repositório |
| `zypper removerepo nome` | `rr` | Remove repositório |
| `zypper modifyrepo` | `mr` | Altera (habilita/desabilita) repositório |
| `zypper clean` | — | Limpa o cache |
| `zypper search --provides arquivo-ou-recurso` | — | Descobre qual pacote fornece o recurso (antigo `what-provides`) |

Configuração: `/etc/zypp/zypp.conf`; repositórios em `/etc/zypp/repos.d/`.

### Equivalências Debian × RPM

| Tarefa | Debian | RPM / YUM |
|---|---|---|
| Instalar arquivo local (baixo nível) | `dpkg -i` | `rpm -ivh` / `rpm -Uvh` |
| Remover (baixo nível) | `dpkg -r` | `rpm -e` |
| Listar instalados | `dpkg -l` | `rpm -qa` |
| Arquivos de um pacote | `dpkg -L` | `rpm -ql` |
| Pacote dono de um arquivo | `dpkg -S` | `rpm -qf` |
| Info de pacote instalado | `dpkg -s` | `rpm -qi` |
| Info de arquivo de pacote | `dpkg -I` | `rpm -qpi` |
| Verificar integridade | `dpkg -V` | `rpm -V` |
| Atualizar índice | `apt-get update` | `yum makecache` / `zypper ref` |
| Instalar com dependências | `apt-get install` | `yum install` / `zypper in` |
| Remover | `apt-get remove` | `yum remove` / `zypper rm` |
| Pesquisar | `apt-cache search` | `yum search` / `zypper se` |
| Quem fornece um arquivo (não instalado) | `apt-file search` | `yum provides` |
| Configuração de repositórios | `/etc/apt/sources.list` | `/etc/yum.repos.d/*.repo` |

### Pegadinhas
- `rpm -e` usa o **nome do pacote** (`rpm -e bash`), e não o nome do arquivo `.rpm`; `rpm -i`/`-U` usam o **arquivo**.
- `rpm -U` instala **mesmo que não exista** versão anterior; `rpm -F` **só atualiza** se já estiver instalado.
- `rpm -qf` → arquivo → pacote **instalado**. `rpm -ql` → pacote → arquivos.
- `rpm -qp...` consulta o **arquivo `.rpm`**; sem o `p` consulta o **banco de dados**.
- `rpm -K` verifica a assinatura do **arquivo**; `rpm -V` verifica os arquivos **instalados**.
- `rpm` **não** resolve dependências; YUM/DNF/Zypper resolvem.
- Repositórios do YUM ficam em `/etc/yum.repos.d/`, e não em `yum.conf`.
- `rpm2cpio` **não instala**: só converte o `.rpm` para fluxo `cpio`.

---

## 102.6: Linux Virtualizado

**Objetivo:** entender as implicações da virtualização e da computação em nuvem em um sistema Linux **convidado** (guest).

**Áreas de conhecimento:**
- entender o conceito geral de máquinas virtuais e contêineres;
- entender elementos comuns de VMs em uma nuvem **IaaS**: instâncias de computação, armazenamento em blocos e rede;
- entender propriedades únicas de um Linux que precisam ser alteradas quando ele é **clonado** ou usado como **template**;
- entender como imagens de sistema são usadas para implantar VMs, instâncias de nuvem e contêineres;
- entender extensões do Linux que o integram a um produto de virtualização;
- conhecer o **cloud-init**.

**Termos:** máquina virtual, contêiner Linux, contêiner de aplicação, drivers de convidado, chaves de host SSH, ID de máquina do D-Bus.

### Máquinas virtuais (VMs)
- O **hypervisor** (VMM) cria e gerencia VMs; cada VM tem **seu próprio kernel** e sistema operacional completo sobre hardware virtualizado.

| Tipo de hypervisor | Descrição | Exemplos |
|---|---|---|
| **Tipo 1** (bare metal) | Roda direto sobre o hardware | Xen, VMware ESXi, Hyper-V, KVM |
| **Tipo 2** (hospedado) | Roda como programa sobre um SO | VirtualBox, VMware Workstation |

- **Virtualização completa:** o convidado roda sem modificações (com suporte de hardware: Intel VT-x → flag `vmx`; AMD-V → flag `svm`).
- **Paravirtualização:** o convidado conhece a virtualização e usa drivers especiais (Xen PV, **virtio**).
- **Emulação** (por exemplo QEMU puro) simula outro hardware; é mais lenta que a virtualização assistida por hardware.
- **KVM:** módulo do kernel Linux (`kvm`, `kvm_intel`, `kvm_amd`; dispositivo `/dev/kvm`); normalmente usado com **QEMU**. Gerenciamento com **libvirt** (`virsh`, `virt-manager`, `virt-install`).
- **Xen:** o domínio de gerenciamento é o **Dom0**; as VMs convidadas são **DomU**.

### Contêineres
- **Compartilham o kernel do host**; não têm kernel próprio. São mais leves e iniciam mais rápido que VMs.
- Isolamento por **namespaces** (pid, net, mnt, uts, ipc, user...) e limite de recursos por **cgroups**.

| Tipo | Ideia | Exemplos |
|---|---|---|
| **Contêiner de sistema** (Linux container) | Roda um userland completo, com init, como se fosse uma "VM leve" | LXC/LXD, systemd-nspawn |
| **Contêiner de aplicação** | Empacota **um processo/aplicação** e suas dependências | Docker, Podman |

- **Imagem de contêiner:** modelo imutável, em camadas; o contêiner é a **instância em execução** da imagem. Imagens vêm de **registries** (por exemplo Docker Hub).
- Podman: sem daemon central e com suporte a execução sem root (rootless).

### Elementos de uma nuvem IaaS

| Elemento | Descrição |
|---|---|
| **Instância de computação** | VM em execução na nuvem, de um certo tipo/tamanho (CPU, RAM) |
| **Armazenamento em blocos** | Volumes anexáveis à instância, vistos como discos (`/dev/vdb`, `/dev/xvdf`...) |
| **Armazenamento de objetos** | Dados acessados por API (não é disco) |
| **Rede** | Redes/sub-redes virtuais, IP público/flutuante, grupos de segurança (firewall) |
| **Imagem** | Modelo usado para criar instâncias |
| **Par de chaves SSH** | Acesso inicial às instâncias, normalmente sem senha |
| **Metadados / user-data** | Dados de configuração entregues à instância no boot |

Modelos de serviço: **IaaS** (infraestrutura), **PaaS** (plataforma), **SaaS** (software).
## Tipos de serviço em nuvem

| | **IaaS** | **PaaS** | **SaaS** |
|---|---|---|---|
| **Nome completo** | Infrastructure as a Service | Platform as a Service | Software as a Service |
| **Significado** | O provedor entrega infraestrutura virtualizada (VMs, redes, armazenamento) sob demanda | O provedor entrega uma plataforma pronta para desenvolver e executar aplicações | O provedor entrega um aplicativo pronto, usado pela internet |
| **Quem usa** | Administradores de sistemas e equipes de infraestrutura | Desenvolvedores | Usuários finais |
| **Nível de controle do cliente** | Alto | Médio | Baixo |
| **Exemplos** | AWS EC2, Azure Virtual Machines, Google Compute Engine, DigitalOcean | Heroku, Google App Engine, Azure App Service | Gmail, Microsoft 365, Google Workspace, Dropbox |

## Como lembrar

- **IaaS:** você recebe a "máquina vazia" e instala o resto. Ex.: criar uma VM Linux e configurar o Apache.
- **PaaS:** você só envia o código, e o provedor cuida do sistema e do ambiente.
- **SaaS:** você só usa o software, sem se preocupar com nada abaixo dele.

Quanto mais você sobe de IaaS para SaaS, **menos controle** e **menos responsabilidade de administração** o cliente tem. Em troca, o provedor assume mais camadas.

### Imagens
- Formatos de disco de VM: **qcow2** (QEMU/KVM), **raw**, **VMDK** (VMware), **VDI** (VirtualBox), **VHD/VHDX** (Hyper-V), além de **ISO** para instalação.
- Ferramenta do QEMU para manipular imagens: `qemu-img`.
- Uma imagem "genérica" de nuvem é implantada e personalizada no primeiro boot (normalmente pelo **cloud-init**).

### Clonagem e templates — o que precisa ser único
Uma máquina clonada ou criada a partir de template **não pode manter** identidades que deveriam ser únicas:

| Item | Onde | Como renovar |
|---|---|---|
| **Chaves de host SSH** | `/etc/ssh/ssh_host_*` | Apagar e regenerar: `ssh-keygen -A` (gera as que faltam) ou `dpkg-reconfigure openssh-server` (Debian) |
| **Machine ID** (D-Bus/systemd) | `/etc/machine-id` e `/var/lib/dbus/machine-id` | Esvaziar o arquivo (`truncate -s 0 /etc/machine-id`) para ser regenerado no próximo boot, ou usar `systemd-machine-id-setup` / `dbus-uuidgen --ensure` |
| **Hostname** | `/etc/hostname` | `hostnamectl set-hostname` |
| **Rede** (endereço MAC, IP estático) | Configuração de rede | Ajustar para a nova máquina |
| **Chaves autorizadas, histórico, logs** | `~/.ssh/authorized_keys`, `~/.bash_history`, `/var/log/` | Limpar conforme a política |

Chaves de host SSH duplicadas fazem várias máquinas terem a **mesma identidade**: risco de segurança (ataque *man-in-the-middle*) e confusão nos clientes (`known_hosts`). Quando as chaves de host mudam, o cliente avisa `REMOTE HOST IDENTIFICATION HAS CHANGED`.

A ferramenta `virt-sysprep` (libguestfs) automatiza a "generalização" de imagens (conhecimento).

### Extensões e drivers de convidado
Pacotes/módulos que integram o Linux ao hypervisor e melhoram desempenho e integração (rede, disco, relógio, mouse, área de transferência, desligamento).

| Plataforma | Componentes |
|---|---|
| KVM/QEMU | Drivers **virtio** (`virtio_net`, `virtio_blk`, `virtio_scsi`, `virtio_balloon`), `qemu-guest-agent` |
| VirtualBox | **Guest Additions** (`vboxguest`, `vboxsf`) |
| VMware | **open-vm-tools** (`vmtoolsd`) |
| Hyper-V | Módulos `hv_*` (Linux Integration Services, incluídos no kernel) |
| SPICE | `spice-vdagent` |

Detectar se o sistema roda virtualizado: `systemd-detect-virt`, `lscpu` (campo *Hypervisor vendor*), `dmesg | grep -i hypervisor`, `dmidecode -s system-product-name`. Dentro de um convidado o `/proc/cpuinfo` mostra a flag `hypervisor`.

### cloud-init (conhecimento)
- Executa no **primeiro boot** (e conforme configurado nos seguintes) e personaliza a instância com base em **metadados** e **user-data** vindos do provedor (datasource).
- Pode: definir hostname, criar usuários, instalar chaves SSH, instalar pacotes, executar comandos (`runcmd`), configurar a rede e redimensionar o disco/sistema de arquivos.
- **user-data** pode ser um arquivo `#cloud-config` (YAML) ou um script.
- Configuração: `/etc/cloud/cloud.cfg` e `/etc/cloud/cloud.cfg.d/`.
- Logs: `/var/log/cloud-init.log` e `/var/log/cloud-init-output.log`. Estado: `cloud-init status`.
- Muitos provedores entregam os metadados por um endereço especial, como `169.254.169.254`.

```yaml
#cloud-config
hostname: servidor01
users:
  - name: admin
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...
packages:
  - nginx
runcmd:
  - systemctl enable --now nginx
```

### Pegadinhas
- **VM** tem kernel próprio; **contêiner** compartilha o kernel do host.
- Contêiner de **sistema** (LXC) ≠ contêiner de **aplicação** (Docker/Podman).
- Ao clonar uma VM, **regenere** as chaves de host SSH e o **machine-id**.
- `qcow2` é formato do QEMU/KVM; `VMDK` é do VMware; `VDI` é do VirtualBox; `VHD/VHDX` é do Hyper-V.
- Armazenamento em **blocos** aparece como disco; **objetos** não.
- cloud-init é **conhecimento**; basta saber o que ele faz e onde fica a configuração.

---
