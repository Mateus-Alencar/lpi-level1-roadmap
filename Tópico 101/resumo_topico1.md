# LPIC-1 — Tópico 101 (Arquitetura do Sistema) — Guia de Estudo

Cobre: **101.1** Hardware · **101.2** Boot do sistema · **101.3** Runlevels/targets, desligar e reiniciar.

---

## 101.1 — Determinar e configurar as definições de hardware

### Sistemas de arquivos virtuais e udev

| Diretório | Conteúdo |
|---|---|
| `/proc` | **Processos e parâmetros do kernel** (`cpuinfo`, `interrupts`, `ioports`) |
| `/sys` | **Dispositivos e drivers** (`/sys/bus`, `/sys/class`, `/sys/devices`) |
| `/dev` | Nós de dispositivo, **criados dinamicamente pelo udev** (sobre devtmpfs) |

- Regras do udev: `/etc/udev/rules.d/` e `/usr/lib/udev/rules.d/`.
- `udevadm monitor`: acompanha eventos do kernel e do udev em tempo real.

Arquivos úteis em `/proc`:

| Arquivo | Informação |
|---|---|
| `/proc/interrupts` | IRQs |
| `/proc/ioports` | Portas de E/S |
| `/proc/iomem` | Memória mapeada |
| `/proc/dma` | Canais DMA |
| `/proc/modules` | Módulos carregados (é de onde o `lsmod` lê) |

**D-Bus**: barramento de mensagens (IPC) que permite que processos conversem entre si e recebam eventos do sistema (ex.: o gerenciador de arquivos é avisado de que um pendrive foi conectado). Há o barramento do **sistema** (*system bus*) e o da **sessão** do usuário (*session bus*). Para observar: `dbus-monitor`.

### Fluxo de carregamento de um módulo

```
Hardware conectado
        ↓
Kernel detecta o dispositivo e gera um uevent (com MODALIAS)
        ↓
udev recebe o evento
        ↓
udev consulta o banco de aliases (modules.alias) e aciona o modprobe
        ↓
modprobe carrega o módulo necessário (e suas dependências, via modules.dep)
        ↓
Driver assume o controle do dispositivo
        ↓
udev cria/ajusta o(s) nó(s) em /dev (e links simbólicos)
```

### Configuração de módulos

- **`/etc/modprobe.d/*.conf`**: configura opções e bloqueios (`options`, `blacklist`, `install`). Não carrega módulos por si só. O `/etc/modprobe.conf` é o formato antigo.
- **`/etc/modules-load.d/`**: lista módulos a carregar **no boot**.
- `blacklist mod`: impede só o carregamento **automático** (por alias). O módulo ainda pode ser carregado manualmente ou como dependência de outro.
- `install mod /bin/false`: bloqueio total.
- `depmod -a`: deve ser rodado após adicionar módulos manualmente (gera o `modules.dep`).

### Tabela-resumo de comandos

| Comando | Função | Opções mais importantes | Exemplo |
|---------|--------|--------------------------|---------|
| `lsmod` | Lista os módulos carregados (lê `/proc/modules`). Colunas: `Module`, `Size`, `Used by` | — | `lsmod` |
| `modinfo` | Mostra informações de um módulo (autor, licença, descrição, parâmetros, dependências) | `-p` parâmetros · `-F <campo>` só um campo (`author`, `license`, `description`, `depends`, `filename`) · `-k <versão>` outro kernel | `modinfo e1000e`<br>`modinfo -p snd_hda_intel` |
| `modprobe` | Carrega módulos **pelo nome, resolvendo dependências**; também remove | `-r` remove (e dependências sem uso) · `-a` carrega vários · `-v` verboso · `-n` simula (*dry-run*) · `-c` mostra a configuração · `--show-depends` mostra dependências | `modprobe e1000e`<br>`modprobe -r e1000e` |
| `insmod` | Carrega um `.ko` **direto**, sem resolver dependências (exige o **caminho** do arquivo) | — | `insmod ./meu_driver.ko` |
| `rmmod` | Remove um módulo, sem resolver dependências. Falha se estiver em uso (`Used by` > 0) | `-f` força · `-v` verboso | `rmmod e1000e` |
| `depmod` | Gera/atualiza `modules.dep` (dependências entre módulos) | `-a` todos os módulos | `depmod -a` |
| `lspci` | Lista dispositivos **PCI/PCIe** | `-k` **driver em uso** e módulos disponíveis · `-v`/`-vv` detalhes (também mostram driver/módulo, IRQ, memória) · `-nn` IDs numéricos (**não** mostra o módulo) · `-s <slot>` filtra por slot | `lspci -k` |
| `lsusb` | Lista dispositivos **USB** (**não** tem `-k`) | `-t` árvore · `-v` detalhes · `-d <VID:PID>` filtra | `lsusb -t` |
| `lsblk` | Lista dispositivos de **bloco** (discos, partições) | `-f` sistema de arquivos/UUID · `-o` colunas · `-a` todos · `-p` caminho completo · `-d` só discos | `lsblk -f` |
| `blkid` | Mostra **UUID**, LABEL e tipo de sistema de arquivos | — | `blkid /dev/sda1` |
| `lshw` | Informações detalhadas de todo o hardware (melhor como root) | `-short` resumo · `-class <classe>` filtra (`network`, `disk`...) | `lshw -short` |
| `lscpu` | Resumo da CPU (modelo, núcleos, cache...) | — | `lscpu` |
| `dmidecode` | Dados do firmware (DMI/SMBIOS): fabricante, memória, BIOS | `-t <tipo>` filtra (ex.: `memory`) | `dmidecode -t memory` |
| `dmesg` | Mensagens do buffer do kernel (boot e detecção de hardware) | `-H` legível · `-w` acompanha em tempo real · `-T` timestamps legíveis | `dmesg -w` |
| `udevadm` | Interage com o udev | `monitor` eventos em tempo real · `info` atributos · `trigger` reenvia eventos · `control --reload` recarrega regras · `test` simula regras | `udevadm monitor` |
| `setpci` | Lê/altera registradores de configuração PCI | — | `setpci -s 00:1f.2 COMMAND` |
| `hwinfo` | Detalha o hardware por categoria (nem sempre instalado) | `--short` resumo | `hwinfo --short` |

### Armadilhas — 101.1

1. **`lsmod` lê `/proc/modules`**, e não `/sys/module`.
2. **`modprobe` × `insmod`:** só o `modprobe` resolve dependências e usa o **nome** do módulo; o `insmod` exige o **caminho do `.ko`**.
3. **`rmmod` não resolve dependências:** falha se o módulo estiver em uso.
4. **`lspci` × `lsusb`:** PCI/PCIe × USB. Para o **driver em uso** de um dispositivo PCI, `lspci -k`.
5. **`blacklist` não impede** carregamento manual nem por dependência; `install ... /bin/false` impede.
6. **`/etc/modules-load.d/`** carrega módulos **no boot**; **`/etc/modprobe.d/`** só **configura**.
7. **`/proc/bus/usb/`** é obsoleto; use `/sys/bus/usb/` e `lsusb`.
8. **`hald` está obsoleto;** foi substituído por **udev** + **D-Bus**.
9. **`/dev`** é gerenciado por **udev** (sobre devtmpfs), e **não** é mantido manualmente.
10. **Bloco (`b`) × caractere (`c`):** `/dev/sda` é bloco; `/dev/tty` é caractere.
11. **NVMe:** partições são `nvme0n1p1` (com **`p`**), e não `nvme0n1`1.
12. **Coldplug × hotplug:** o USB é **hotplug**; CPU e RAM são **coldplug**.
13. **`dmesg` pode exigir root** em sistemas com `kernel.dmesg_restrict=1`.
14. **`udevadm trigger`** reenvia eventos; **`udevadm monitor`** apenas observa.

---

## 101.2 — Início (boot) do sistema

### BIOS/MBR × UEFI/GPT

| | BIOS / MBR | UEFI / GPT |
|---|---|---|
| Quem inicia | Código do BIOS | Firmware UEFI |
| Onde fica o bootloader | **Primeiro setor (512 bytes)**: 446 de código + 64 de tabela de partições + 2 de assinatura `0x55AA` | Arquivo `.efi` na **ESP** (FAT, montada em `/boot/efi`) |
| Ferramenta | `grub-install /dev/sda` | `efibootmgr` (`-v` lista, `-o` ordem, `-b` entrada) |

- UEFI normalmente usa GPT, mas **não exige** MBR.
- Em UEFI não existe "bootloader nos 446 bytes iniciais": isso é característica do BIOS/MBR.
- **MBR × GPT** é particionamento (disco); **BIOS × UEFI** é firmware (placa-mãe). Não confundir.

### Fluxo de boot

1. Firmware (BIOS ou UEFI)
2. **Bootloader** (GRUB)
3. O bootloader carrega o **kernel (`vmlinuz`) + initramfs** na memória
4. O kernel descompacta o initramfs e executa o `/init` dele
5. O `/init` carrega os módulos necessários e monta a **raiz real**
6. `switch_root` e inicialização do **systemd** (PID 1)
7. O systemd inicia os serviços até o target padrão, e então o login

> Quem carrega o kernel é o **bootloader**, não o BIOS/UEFI. Quem inicia os serviços é o **systemd** (ou outro init), não o kernel.

### Initramfs

- Mini sistema raiz temporário em RAM, com os módulos necessários para montar a raiz real (RAID, LVM, criptografia, XFS, NVMe...).
- **Quem o carrega:** o bootloader, nunca o firmware.
- Gerar:
  - Debian/Ubuntu: `update-initramfs -u` (ou `mkinitramfs`)
  - RHEL/Fedora: `dracut`
  - Antigo: `mkinitrd`
- Inspecionar: `lsinitramfs` (Debian), `lsinitrd` (RHEL).
- Fica em `/boot` (`initrd.img-*` ou `initramfs-*.img`), **separado** do `vmlinuz`.

### GRUB 2

| Item | Detalhe |
|---|---|
| `/etc/default/grub` | Configuração principal (é o que se edita) |
| `/etc/grub.d/` | Scripts que compõem o menu |
| `/boot/grub/grub.cfg` | **Gerado, não editar à mão** |
| Regenerar | `grub-mkconfig -o /boot/grub/grub.cfg` |
| Atalho Debian | `update-grub` |
| RHEL/Fedora | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| Instalar | `grub-install /dev/sda` |
| Tecla `e` no menu | Edita **só aquele boot**; `Ctrl+X` inicia |
| Parâmetro `quiet` | Apenas suprime mensagens do kernel |

> O `-o` **exige o caminho de saída**. Sem ele, a configuração apenas é impressa no terminal.

### Parâmetros do kernel (linha `linux` do GRUB)

Valem **somente para aquele boot**.

| Parâmetro | Efeito |
|---|---|
| `systemd.unit=rescue.target` | Modo rescue (monousuário) |
| `systemd.unit=emergency.target` | Modo de emergência (raiz somente leitura, mínimo) |
| `systemd.unit=multi-user.target` | Modo texto multiusuário |
| `single` / `1` | Monousuário |

### Logs e análise do boot

| Comando | Mostra |
|---|---|
| `dmesg` | Mensagens do kernel |
| `journalctl -k` | Mensagens do kernel registradas no journal (systemd) |
| `journalctl -b` | Logs do boot **atual** |
| `journalctl -b -1` | Logs do boot **anterior** |
| `systemd-analyze` | **Tempo** total do boot |
| `systemd-analyze blame` | **Tempo** por unidade, em ordem |
| `systemd-analyze critical-chain` | **Cadeia crítica** de unidades que atrasaram o boot |

> **Regra de bolso:** `journalctl` = **logs** · `systemd-analyze` = **tempo**.

### Armadilhas — 101.2

1. Em UEFI o bootloader é um `.efi` na **ESP**, não nos 446 bytes do MBR.
2. O **bootloader** (não o firmware) carrega kernel e initramfs.
3. `grub.cfg` é **gerado**: edite `/etc/default/grub` e regenere.
4. `grub-mkconfig -o` sem o caminho de saída não grava nada.
5. A tecla `e` do GRUB e os parâmetros na linha do kernel valem **só para aquele boot**.
6. O initramfs é um arquivo **separado** do `vmlinuz`.
7. `journalctl` = logs · `systemd-analyze` = tempo.

---

## 101.3 — Alternar runlevels/boot targets, desligar e reiniciar

### Runlevels × targets

| Runlevel | Função | Target systemd |
|---|---|---|
| 0 | **Desligar** (halt) | `poweroff.target` |
| 1 | Monousuário | `rescue.target` |
| 2, 3, 4 | Multiusuário (texto) | `multi-user.target` |
| 5 | Multiusuário gráfico | `graphical.target` |
| 6 | **Reiniciar** | `reboot.target` |

- **0 desliga** e **6 reinicia** (as bancas gostam de inverter).
- `runlevelN.target` são links para os targets acima.
- Runlevel 2 no Debian/Ubuntu é multiusuário **com rede**; nos Red Hat clássicos, é sem rede.
- Runlevel 4 é "não usado" por padrão.

### Comandos e arquivos por sistema de init

```bash
# =========================================================
# systemd
# Arquivos principais:
#   /lib/systemd/system/         (ou /usr/lib/systemd/system/) unidades instaladas por pacotes
#   /etc/systemd/system/         unidades e overrides do administrador (maior prioridade)
#   /run/systemd/system/         unidades geradas em tempo de execução (somem no reboot)
#   /etc/systemd/system/default.target   link simbólico para o target padrão
#   /etc/systemd/system/<target>.wants/  links criados pelo "systemctl enable"
#   /etc/systemd/system/<unidade>.d/override.conf  drop-in criado por "systemctl edit"
# Prioridade: /etc > /run > /lib
# =========================================================
systemctl get-default                # mostra o target padrão de boot
systemctl set-default multi-user.target   # define o target padrão (permanente, a partir do próximo boot)
                                     # por baixo: ln -sf /usr/lib/systemd/system/X.target /etc/systemd/system/default.target
systemctl isolate rescue.target      # muda para o target agora (temporário, só nesta sessão)
systemctl enable --now sshd          # habilita no boot E inicia agora
systemctl mask <svc>                 # bloqueia qualquer início (link para /dev/null)
systemctl daemon-reload              # relê os arquivos de unidade após editá-los

# =========================================================
# SysVinit
# Arquivos principais:
#   /etc/inittab        runlevel padrão (initdefault) e processos do init
#   /etc/init.d/        scripts reais dos serviços (start|stop|restart|status)
#   /etc/rc0.d ... /etc/rc6.d/   links simbólicos para os scripts (S = start, K = kill, 00-99 = ordem)
#   /etc/rc.d/ (famílias Red Hat) pode conter o init.d e os rcX.d
# Formato do inittab: id:runlevels:ação:processo  (runlevel padrão: id:N:initdefault:)
# =========================================================
runlevel          # mostra runlevel anterior e atual (ex: N 5, onde N = sem anterior)
who -r            # também mostra o runlevel atual
init 3 / telinit 3   # muda para o runlevel 3 agora (temporário)
telinit q         # manda o init reler o /etc/inittab (não muda de runlevel)

# =========================================================
# Upstart
# Arquivos principais:
#   /etc/init/*.conf                  definição dos jobs (não confundir com /etc/init.d/)
#   /etc/init/*.override              sobrescreve um job (ex: "manual" desativa o início)
#   /etc/init/rc-sysinit.conf         variável DEFAULT_RUNLEVEL (runlevel padrão)
# Compatível com os scripts do SysV (/etc/init.d/ e rcX.d)
# =========================================================
initctl list                     # lista todos os jobs e seus estados
initctl reload-configuration     # relê os arquivos de /etc/init/

# =========================================================
# Desligar/reiniciar
# Arquivo relacionado:
#   /etc/nologin    criado nos últimos 5 min do shutdown; bloqueia login de usuários comuns
# =========================================================
shutdown -h now                  # desliga imediatamente
shutdown -h 23:30                # desliga às 23:30
shutdown -r +10 "mensagem"       # reinicia em 10 minutos e avisa os usuários logados
shutdown -c                      # cancela um desligamento agendado
shutdown -k now                  # SÓ envia o aviso, não desliga
halt / poweroff / reboot         # para o sistema / desliga cortando energia / reinicia
                                 # (com systemd são links simbólicos para o systemctl)
systemctl poweroff               # desliga
systemctl reboot                 # reinicia
init 0 / telinit 0               # desliga (runlevel 0)
init 6 / telinit 6               # reinicia (runlevel 6)
```

### Armadilhas — 101.3

1. `shutdown -k` **não desliga**: só avisa.
2. `telinit 6` é **reboot**, não runlevel 5.
3. `init 1` é **monousuário**, não gráfico.
4. `isolate`, `telinit` e `init N` são **temporários**; `set-default` é permanente.
5. `systemctl enable` **não** define o target padrão e **não inicia** o serviço agora (use `enable --now` ou `start`).
6. `disable` não impede início manual; só `mask` impede.
7. `systemctl reload` **não** derruba o processo; `restart` derruba.
8. `/etc/init.d/` (scripts SysV) é diferente de `/etc/init/` (jobs do Upstart).
9. Links em `rcX.d`: prefixo `K` = parar (*kill*), `S` = iniciar (*start*); os números definem a ordem.
10. `telinit q` relê o `/etc/inittab`, não muda de runlevel.

### Apoio: sinais e processos

```bash
# Sinais: 1 = SIGHUP (relê configuração), 9 = SIGKILL (à força), 15 = SIGTERM (padrão, educado)
kill -15 PID                     # pede o término do processo de forma limpa (SIGTERM)
kill -9 PID                      # mata à força, sem chance de tratar o sinal (SIGKILL)
killall nome                     # envia sinal a todos os processos com esse nome exato
pkill -u usuario padrao          # envia sinal aos processos do usuário cujo nome casa com o padrão
```

---

## Revisão rápida  

```bash
dmesg
journalctl -b
systemctl get-default
systemctl set-default graphical.target
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
grub-mkconfig -o /boot/grub/grub.cfg
shutdown -h 23:30
shutdown -r +10
```

Parâmetro do kernel:

```
systemd.unit=rescue.target
```
