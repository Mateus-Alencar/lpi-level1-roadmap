# LPIC-1 – Tópico 104: Dispositivos, Sistemas de Arquivos Linux e FHS

Resumo de revisão para a prova 102 (v5.0), cobrindo 104.1, 104.2, 104.3, 104.5, 104.6 e 104.7. O foco está no que costuma cair: comandos, opções, arquivos de configuração e as pegadinhas clássicas.

## 104.1 – Criar partições e sistemas de arquivos

### MBR x GPT

- **MBR (DOS)**: tabela no primeiro setor (512 bytes): 446 bytes de código de boot, 64 bytes de tabela de partições (4 entradas de 16 bytes) e 2 bytes de assinatura (0x55AA).
- MBR: até **4 partições primárias**, ou 3 primárias + 1 **estendida** com partições **lógicas** dentro dela. Lógicas são numeradas a partir de **5** (sda5, sda6...). Limite de **2 TiB** por disco/partição.
- **GPT**: usada com UEFI (também funciona com BIOS). Por padrão até **128 partições**, discos acima de 2 TiB (limite teórico de 8 ZiB), identifica partições por **GUID**, guarda **cópia de backup** da tabela no fim do disco e mantém um **MBR protetor** no início.
- **ESP (EFI System Partition)**: partição FAT32 (tipo EF00 no gdisk), normalmente montada em `/boot/efi`.
- **BIOS boot partition** (GPT + BIOS/GRUB): tipo EF02 no gdisk.

### Ferramentas de particionamento

- `fdisk`: clássico para MBR; versões atuais também trabalham com GPT. `fdisk -l` lista as tabelas de todos os discos.
- `gdisk`: específico para **GPT**.
- `parted`: MBR e GPT; **as alterações são aplicadas imediatamente** (não há "w" para gravar).
- `cfdisk`: interface em ncurses.
- As alterações do `fdisk`/`gdisk` só são gravadas ao usar **`w`**. `q` sai sem salvar.
- `partprobe` (ou `partx -u`) pede ao kernel que releia a tabela de partições sem reiniciar.

Comandos internos do `fdisk`:

- `m` ajuda, `p` imprime a tabela, `n` nova partição, `d` apaga, `t` muda o tipo, `l` lista os tipos, `a` marca como bootável (MBR), `g` cria nova tabela **GPT**, `o` cria nova tabela **DOS/MBR**, `w` grava e sai, `q` sai sem gravar.

Códigos de tipo mais cobrados:

- MBR/fdisk: `83` Linux, `82` Linux swap, `8e` Linux LVM, `fd` Linux RAID, `5` ou `f` estendida, `ef` EFI.
- GPT/gdisk: `8300` Linux, `8200` swap, `8E00` LVM, `FD00` RAID, `EF00` ESP, `EF02` BIOS boot.

Outros comandos úteis: `lsblk` (árvore de dispositivos, `-f` mostra FS, label e UUID) e `blkid` (UUID, LABEL, TYPE).

### Sistemas de arquivos

- **ext2**: sem journal.
- **ext3**: ext2 + journal.
- **ext4**: journal, *extents*, alocação otimizada, suporta volumes e arquivos muito grandes; padrão em muitas distribuições.
- **XFS**: journaling, alto desempenho com arquivos grandes, pode ser **ampliado (`xfs_growfs`), mas não reduzido**.
- **Btrfs**: copy-on-write, **snapshots**, **subvolumes**, checksums, compressão, suporte a vários dispositivos (RAID) no próprio FS.
- **VFAT/FAT32**: compatível com Windows, usado na ESP e em mídias removíveis. Sem permissões Unix.
- **exFAT**: para pendrives/cartões grandes, sem permissões Unix. **NTFS**: acesso via `ntfs-3g`/driver do kernel.

### Criação de sistemas de arquivos (`mkfs`)

- Forma geral: `mkfs -t TIPO /dev/sdXN` ou `mkfs.TIPO /dev/sdXN`.
- `mkfs.ext2/3/4` (e `mke2fs -t ext4`): `-L rótulo`. `mke2fs -j` cria ext3.
- `mkfs.xfs`: `-L rótulo`, `-f` força sobrescrever um FS existente.
- `mkfs.btrfs`: `-L rótulo`.
- `mkfs.vfat -F 32 -n RÓTULO` (também `mkfs.fat`). `mkfs.exfat`.
- Rótulos depois da criação: `e2label` ou `tune2fs -L` (ext), `xfs_admin -L` (XFS), `btrfs filesystem label`.

### Btrfs – o que saber

- `btrfs subvolume create /mnt/sub`, `btrfs subvolume list /mnt`, `btrfs subvolume snapshot origem destino` (adicione `-r` para snapshot somente leitura).
- `btrfs filesystem show` mostra os dispositivos do FS. `btrfs device add /dev/sdc /mnt` adiciona um dispositivo ao FS (multi-device). `btrfs balance` redistribui os dados.
- Compressão: opção de montagem `-o compress=zstd` (ou `lzo`, `zlib`).
- `btrfs scrub start /mnt` verifica checksums; `btrfs check` verifica o FS desmontado.

### Swap

- Preparar: `mkswap /dev/sdXN` (aceita `-L`). Ativar: `swapon /dev/sdXN`. Desativar: `swapoff`.
- `swapon -a` ativa todos os swaps do `/etc/fstab`. `swapon --show` ou `swapon -s` e `cat /proc/swaps` listam os swaps ativos. `free` mostra o uso.
- **Arquivo de swap**: criar com `dd if=/dev/zero of=/swapfile bs=1M count=1024` (ou `fallocate -l 1G /swapfile`), `chmod 600 /swapfile`, `mkswap /swapfile`, `swapon /swapfile`.
- Linha no fstab: `/swapfile none swap defaults 0 0` (tipo `swap`, ponto de montagem `none`).
- O tipo da partição (82/8200) é recomendado, mas quem define o uso como swap é o `mkswap`.

## 104.2 – Manutenção da integridade de sistemas de arquivos

### Espaço em disco: `df` e `du`

- `df`: espaço **por sistema de arquivos montado**. `-h` (potências de 1024), `-H` (potências de 1000), `-T` mostra o tipo, `-i` mostra **inodes**, `-t tipo` filtra.
- `du`: espaço **usado por diretórios/arquivos**. `-s` resumo, `-h` legível, `-c` total geral, `-a` inclui arquivos, `-d N` ou `--max-depth=N` limita a profundidade, `-x` não cruza outros sistemas de arquivos.
- **Pegadinha**: o disco pode estar "cheio" com espaço livre se os **inodes acabaram**. Confirme com `df -i` (IUse% = 100%).

### `fsck`

- `fsck` é um *front-end* que chama `fsck.ext4`/`e2fsck`, `fsck.xfs`, etc. **Nunca execute em sistema de arquivos montado** (risco de corrupção); desmonte ou use modo somente leitura. Para o `/`, use modo de recuperação/live CD ou a verificação no boot.
- Opções principais: `-t tipo`, `-A` (verifica todos do fstab), `-N` (não executa, só mostra), `-a` ou `-p` (reparo automático seguro), `-y` (responde sim a tudo), `-n` (responde não a tudo, somente leitura), `-f` (força verificação mesmo se estiver "limpo"), `-r`.
- Ordem de verificação no boot: **6º campo do `/etc/fstab`**. `0` = não verifica, `1` = raiz (primeiro), `2` = demais.
- Códigos de saída (somados): `0` sem erros, `1` erros corrigidos, `2` reinício necessário, `4` erros não corrigidos, `8` erro operacional, `16` erro de uso, `32` cancelado, `128` erro de biblioteca compartilhada.
- Arquivos recuperados vão para o diretório `lost+found` na raiz do sistema de arquivos.
- `e2fsck -b 32768 /dev/sdXN` usa um **superbloco de backup**. `-c` procura blocos defeituosos (`badblocks`).

### Ferramentas ext2/3/4

- `tune2fs`: ajusta parâmetros de um FS ext.
  - `-l` lista o superbloco; `-c N` máximo de montagens entre verificações (`0` ou `-1` desativa); `-i 2w` intervalo de tempo (d, w, m); `-L` rótulo; `-U` UUID; `-m %` percentual reservado ao root; `-r N` blocos reservados; `-e` comportamento em erro (`continue`, `remount-ro`, `panic`).
  - `-j` adiciona journal (**converte ext2 em ext3**); `-O ^has_journal` remove o journal.
- `dumpe2fs -h /dev/sdXN`: exibe informações do superbloco (somente leitura).
- `debugfs`: depurador interativo do ext (leitura; `-w` para escrita).

### Ferramentas XFS

- `xfs_repair`: reparo (FS **desmontado**). `-n` só verifica, sem alterar. `-L` zera o log (último recurso). `fsck.xfs` **não faz nada** de fato.
- `xfs_info`: informações do FS montado. `xfs_db`: depuração. `xfs_metadump`: copia os metadados para análise. `xfs_fsr`: desfragmentação. `xfs_admin`: rótulo e UUID. `xfs_growfs`: amplia o FS montado.

### Btrfs

- `btrfs check` (desmontado) e `btrfs scrub` (montado).

## 104.3 – Controle da montagem e desmontagem dos sistemas de arquivos

### `mount` e `umount`

- `mount` sem argumentos lista o que está montado. `mount -t tipo -o opções dispositivo diretório`.
- Identificar o dispositivo: `/dev/sdb1`, `UUID=...`, `LABEL=...` (também `-U` e `-L` na linha de comando) ou `PARTUUID=...`. Os links em `/dev/disk/by-uuid`, `by-label`, `by-id` e `by-path` são estáveis.
- `mount -a`: monta tudo do `/etc/fstab` **exceto** linhas com `noauto`.
- `mount -o remount,rw /` remonta sem desmontar. `-r` ou `-o ro` somente leitura; `-w` ou `-o rw` leitura e escrita.
- `mount -o loop arquivo.iso /mnt` monta uma imagem. `mount --bind /origem /destino` (ou `-B`) espelha um diretório em outro local.
- `umount /dev/sdb1` ou `umount /mnt` (dispositivo **ou** ponto de montagem).
- Erro **"target is busy"**: algo está usando o ponto de montagem. Descubra com `lsof /mnt` ou `fuser -vm /mnt`. `umount -l` (lazy) desanexa agora e limpa depois; `umount -f` força (útil em NFS).
- Pontos de montagem comuns: `/mnt` (montagens temporárias) e `/media` (mídias removíveis).

### Ver o que está montado

- `findmnt` (árvore de montagens), `lsblk`, `blkid`, `/proc/mounts` e `/proc/self/mounts`. `/etc/mtab` costuma ser um link para `/proc/self/mounts`.

### `/etc/fstab` – 6 campos

```
<dispositivo>   <ponto>   <tipo>   <opções>   <dump>   <pass>
UUID=abcd-1234  /dados    ext4     defaults   0        2
```

1. Dispositivo (caminho, UUID, LABEL ou PARTUUID).
2. Ponto de montagem (para swap: `none`).
3. Tipo do sistema de arquivos.
4. Opções de montagem.
5. **dump**: `0` = não entra no backup do `dump`, `1` = entra.
6. **pass**: ordem do `fsck` (`0` não verifica, `1` raiz, `2` demais).

Opções de montagem mais cobradas:

- `defaults` = `rw, suid, dev, exec, auto, nouser, async`.
- `noauto` não monta com `mount -a`/no boot. `auto` é o contrário.
- `ro` / `rw`. `exec` / `noexec` (proíbe execução de binários). `suid` / `nosuid` (ignora SUID/SGID). `dev` / `nodev` (ignora arquivos de dispositivo).
- `user`: usuário comum pode montar, **e só quem montou pode desmontar**; implica `noexec,nosuid,nodev`. `users`: qualquer usuário pode montar e desmontar. `owner`: o dono do dispositivo pode montar.
- `sync` / `async`. `atime` / `noatime` / `relatime` (padrão atual).
- `uid=`, `gid=`, `umask=` (FS sem permissões Unix, como vfat). `nofail` (não falha o boot se ausente). `_netdev` (precisa de rede).

### Unidades de montagem do systemd

- O systemd converte as linhas do fstab em unidades `.mount` (via *fstab-generator*), mas também é possível criar unidades manualmente em `/etc/systemd/system/`.
- **O nome do arquivo deve refletir o caminho**: `/mnt/dados` vira `mnt-dados.mount`. `systemd-escape -p --suffix=mount /mnt/dados` gera o nome.

```
[Unit]
Description=Dados

[Mount]
What=/dev/disk/by-uuid/abcd-1234
Where=/mnt/dados
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target
```

- Ativar: `systemctl daemon-reload` e `systemctl enable --now mnt-dados.mount`.
- **Automount**: unidade `mnt-dados.automount` (seção `[Automount]` com `Where=`) ou, no fstab, `x-systemd.automount` (com `x-systemd.idle-timeout=`). Monta sob demanda, no primeiro acesso.

## 104.5 – Controlar permissões e propriedades de arquivos

### Leitura do `ls -l`

```
-rwxr-xr--  1  joao  dev  1024  jan 10  script.sh
```

- Primeiro caractere: `-` arquivo, `d` diretório, `l` link simbólico, `c` dispositivo de caractere, `b` dispositivo de bloco, `p` FIFO, `s` socket.
- Depois: 3 trios (dono, grupo, outros) com `r`=4, `w`=2, `x`=1.

### Efeito em arquivos x diretórios

- **Arquivo**: `r` ler, `w` alterar o conteúdo, `x` executar.
- **Diretório**: `r` listar os nomes, `x` entrar (`cd`) e acessar o que está dentro, `w` criar, apagar e renomear entradas (**só funciona junto com `x`**).
- **Apagar um arquivo depende das permissões do diretório**, não do arquivo.
- O `root` ignora as permissões de leitura/escrita, mas só executa arquivos que tenham ao menos um bit `x`.

### `chmod`

- **Simbólico**: `u`, `g`, `o`, `a` + operador `+`, `-`, `=` + `r`, `w`, `x`, `X`, `s`, `t`.
  - `chmod u+x,g-w,o= arq`, `chmod a+r arq`, `chmod g=u arq` (copia as permissões do dono para o grupo), `chmod -R u+rwX,go+rX dir`.
- **Octal**: soma 4+2+1 em cada trio.
  - `755` = `rwxr-xr-x`, `644` = `rw-r--r--`, `600` = `rw-------`, `700` = `rwx------`, `640` = `rw-r-----`, `777` = `rwxrwxrwx`.
- `-R` aplica recursivamente. Só o **dono** (ou root) pode usar `chmod`.

### Permissões especiais

- **SUID (4000)**: o arquivo executável roda com a identidade do **dono**. Aparece como `s` no lugar do `x` do dono (`rwsr-xr-x`, ex.: `/usr/bin/passwd`). Sem `x` aparece `S` maiúsculo. Em diretórios é ignorado no Linux.
- **SGID (2000)**: em arquivo executável, roda com o **grupo** do arquivo (`rwxr-sr-x`). **Em diretório, os novos arquivos herdam o grupo do diretório** (usado em pastas de equipe).
- **Sticky bit (1000)**: em diretório, **só o dono do arquivo, o dono do diretório ou o root podem apagar/renomear** os arquivos. Aparece como `t` (`rwxrwxrwt`) ou `T` sem `x`. Exemplo clássico: `/tmp` (`1777`).
- Simbólico: `chmod u+s`, `chmod g+s`, `chmod +t`. Octal: dígito extra à esquerda, `chmod 4755 arq`, `chmod 2775 dir`, `chmod 1777 dir`.

### `chown` e `chgrp`

- `chown usuario arq`, `chown usuario:grupo arq`, `chown :grupo arq`, `chown -R usuario:grupo dir`. A forma `usuario.grupo` é antiga, mas funciona.
- `chgrp grupo arq`.
- **Só o root altera o dono.** Um usuário comum pode trocar o grupo de um arquivo seu, desde que seja **membro** do novo grupo.
- `chown` segue o link simbólico por padrão; `-h` altera o próprio link.
- Grupo de trabalho compartilhado: coloque os usuários no mesmo grupo, use `chmod g+rw` e SGID no diretório. Consulte com `id` e `groups`; `newgrp grupo` muda o grupo primário da sessão.

### `umask`

- Define as permissões **removidas** dos novos arquivos. Base: **666 para arquivos** e **777 para diretórios**.
- `umask 022` → arquivos `644`, diretórios `755`. `umask 027` → `640` / `750`. `umask 077` → `600` / `700`. `umask 002` → `664` / `775`.
- Arquivos novos **nunca** nascem com `x`, mesmo com `umask 000` (ficam `666`).
- `umask` mostra o valor atual, `umask -S` mostra em formato simbólico. Configuração permanente: `/etc/profile`, `~/.bashrc`, `~/.profile`.

## 104.6 – Criar e alterar links simbólicos e hardlinks

### Comparação

- **Hardlink**: outro **nome** para o mesmo inode. Mesmo conteúdo, dono e permissões. Deve estar no **mesmo sistema de arquivos**. **Não pode apontar para diretórios**. Não "quebra": se o original for apagado, os dados continuam acessíveis pelo outro nome, e só são liberados quando o contador de links chega a 0.
- **Link simbólico (soft)**: arquivo especial com **inode próprio**, que guarda o **caminho** do alvo. Pode **cruzar sistemas de arquivos**, pode apontar para **diretórios** e pode **quebrar** (*dangling*) se o alvo for removido ou movido.

### Comandos

- `ln ALVO NOME_DO_LINK`: cria **hardlink**. `ln -s ALVO NOME_DO_LINK`: cria **link simbólico**. Ordem dos argumentos: igual ao `cp` (origem primeiro, destino depois).
- Se o nome do link for omitido, o link é criado no diretório atual com o mesmo nome do alvo.
- `ln -sf` substitui um link existente. `ln -n` trata um link para diretório como arquivo comum.
- **Pegadinha**: em `ln -s` com caminho **relativo**, o alvo é interpretado **a partir da pasta onde o link está**, e não da pasta onde você digitou o comando. Isso é uma causa comum de links quebrados.
- Identificação:
  - `ls -l`: tipo `l` e `link -> alvo`. Permissões de um link simbólico aparecem como `lrwxrwxrwx` (o que vale é a permissão do alvo). O tamanho do link é o comprimento do caminho que ele guarda.
  - `ls -i`: mostra o **inode**. Hardlinks compartilham o mesmo número.
  - `ls -l` (2ª coluna) e `stat`: mostram o **contador de links**.
  - `readlink link` e `readlink -f link` (caminho final resolvido). `file link` indica "symbolic link to ...".
  - `find / -samefile arq` ou `find / -inum N`: acha hardlinks. `find -type l`: lista links simbólicos. `find -xtype l`: lista links **quebrados**. `test -h` / `test -L`: verifica se é link simbólico.
- `rm link` remove **apenas o link**. `chmod` em um link simbólico altera o **alvo**.
- Exemplos no sistema: `/bin` → `usr/bin` em distribuições modernas e `/etc/alternatives`.

## 104.7 – Encontrar arquivos de sistema e conhecer sua localização correta (FHS)

### Hierarquia (FHS)

- `/` raiz. `/bin` binários essenciais. `/sbin` binários de administração. `/lib`, `/lib64` bibliotecas essenciais. `/boot` kernel, initramfs e GRUB. `/dev` dispositivos. `/etc` configurações do sistema. `/home` diretórios dos usuários. `/root` home do root. `/media` mídias removíveis. `/mnt` montagens temporárias. `/opt` software opcional de terceiros. `/srv` dados de serviços (web, FTP). `/tmp` temporários. `/lost+found` itens recuperados pelo `fsck`.
- **Sistemas de arquivos virtuais**: `/proc` (processos e kernel), `/sys` (dispositivos e drivers), `/dev` (arquivos de dispositivo) e `/run` (dados voláteis de execução, em tmpfs).
- `/usr`: dados compartilháveis e (em geral) somente leitura.
  - `/usr/bin`, `/usr/sbin`, `/usr/lib`, `/usr/include`, `/usr/share` (independente de arquitetura: man, doc), `/usr/src` (fontes), `/usr/local` (software instalado localmente pelo administrador, fora do gerenciador de pacotes).
- `/var`: dados variáveis.
  - `/var/log` logs, `/var/spool` filas (e-mail, impressão, cron), `/var/cache`, `/var/lib` (estado de programas, ex.: bancos do dpkg/rpm), `/var/tmp` (temporários que **sobrevivem ao reboot**), `/var/mail`.
- `/tmp` costuma ser limpo no boot; `/var/tmp` não. `/var/run` é link para `/run`.
- **Devem ficar na mesma partição da raiz (`/`)**: `/etc`, `/bin`, `/sbin`, `/lib` e `/dev`, pois são necessários no boot. **Podem ser partições separadas**: `/boot`, `/home`, `/usr`, `/var`, `/tmp`, `/opt`.
- `/opt` (pacotes de terceiros) x `/usr/local` (instalações locais do administrador): não confundir.

### `find`

Sintaxe: `find [caminho] [testes] [ação]`. Se não houver ação, usa `-print`.

- Nome: `-name 'padrão'` (diferencia maiúsculas, **use aspas** para o shell não expandir o curinga), `-iname` (ignora maiúsculas), `-path`, `-regex` (casa o caminho inteiro).
- Tipo: `-type f` arquivo, `d` diretório, `l` link simbólico, `b`, `c`, `p`, `s`.
- Dono: `-user`, `-group`, `-nouser`, `-nogroup`.
- Tamanho: `-size +100M` (maior que), `-size -1k` (menor que), `-size 10c` (exatamente 10 bytes). Unidades: `c` bytes, `k` KiB, `M`, `G`. **Sem sufixo, a unidade é bloco de 512 bytes** (pegadinha). O tamanho é arredondado para cima na unidade usada.
- Tempo (dias): `-mtime` (conteúdo modificado), `-atime` (acesso), `-ctime` (alteração do inode/metadados).
  - `-mtime +7` = há **mais de** 7 dias; `-mtime -7` = há **menos de** 7 dias; `-mtime 7` = exatamente 7 (dias completos).
  - Em minutos: `-mmin`, `-amin`, `-cmin`. `-newer arq`: mais novo que `arq`.
- Permissão: `-perm 644` (**exatamente** 644), `-perm -644` (**todos** esses bits ligados), `-perm /644` (**qualquer** um desses bits).
  - Arquivos SUID: `find / -perm -4000 -type f`. SUID **ou** SGID: `find / -perm /6000`.
- Outros: `-empty`, `-links N`, `-inum N`, `-samefile arq`, `-maxdepth N`, `-mindepth N` (coloque antes dos demais testes).
- Lógica: `-a` (e, padrão), `-o` (ou), `!` ou `-not` (negação), com parênteses escapados `\( ... \)`.
- Ações:
  - `-exec comando {} \;`: executa uma vez **por arquivo**. `-exec comando {} +`: agrupa vários arquivos por execução (mais eficiente). `-ok`: pede confirmação. `-delete`: remove. `-ls`: listagem longa.
  - Com `xargs`: `find . -name '*.log' -print0 | xargs -0 rm` (seguro com espaços nos nomes).

### `locate` e `updatedb`

- `locate` consulta um **banco de dados** (mlocate/plocate, ex.: `/var/lib/mlocate/mlocate.db`), por isso é rápido, mas **não encontra arquivos criados depois da última atualização** até rodar `updatedb` (normalmente agendado pelo cron/timer diário).
- Por padrão procura o texto em qualquer parte do **caminho completo**. Opções: `-i` ignora maiúsculas, `-c` conta, `-n N` limita resultados, `-b` só o nome base, `-r` regex, `-e` só os que ainda existem, `-S` estatísticas do banco.
- Configuração: `/etc/updatedb.conf` (`PRUNEPATHS`, `PRUNEFS`, `PRUNENAMES`) define o que **não** é indexado.

### `which`, `whereis`, `type`

- `which comando`: mostra o executável que seria usado, procurando nos diretórios do `$PATH` (retorna o primeiro; `-a` mostra todos). Não enxerga *builtins* nem funções do shell de forma confiável.
- `whereis comando`: localiza **binário, código-fonte e manual** em locais padrão. `-b` só binário, `-m` só manual, `-s` só fonte.
- `type comando` (builtin do bash): informa se é **alias, builtin, função, palavra reservada ou arquivo**. `type -a` mostra todas as ocorrências.
- `$PATH`: lista de diretórios separados por `:`, consultados em ordem.

## Pegadinhas de prova (resumo rápido)

- `fdisk` grava só com **`w`**; `parted` aplica **na hora**.
- MBR: 4 primárias (ou 3 + estendida), lógicas a partir do **5**, limite de **2 TiB**. GPT: até 128 partições, **GUID**.
- XFS: ampliar sim (`xfs_growfs`), reduzir **não**. Reparo com `xfs_repair`, **desmontado**.
- `tune2fs -j` converte ext2 em ext3. `tune2fs -l` e `dumpe2fs -h` mostram o superbloco.
- `fsck`: nunca em FS montado. 6º campo do fstab = ordem do fsck (`1` raiz, `2` demais).
- Disco cheio com espaço sobrando: `df -i` (inodes).
- `defaults` = `rw,suid,dev,exec,auto,nouser,async`. `user` só deixa **quem montou** desmontar; `users` deixa **qualquer um**.
- Unidade `.mount` deve ter o **nome igual ao caminho** (`mnt-dados.mount`).
- Apagar arquivo depende do **diretório** (`w` + `x`). Sticky bit: só o dono (ou root) apaga.
- SGID em diretório: **herança do grupo**. SUID: executa como o **dono**.
- `umask 022` → **644** (arquivos) e **755** (diretórios). Arquivos novos nunca saem com `x`.
- `ln -s` com caminho relativo é relativo à **pasta do link**. Hardlink: mesmo FS, sem diretórios, mesmo inode.
- `find -size 10` sem sufixo = 10 blocos de **512 bytes**. `-mtime +7` = **mais de** 7 dias. `-perm -4000` x `-perm /6000`.
- `locate` não vê arquivo novo até o `updatedb`. `which` só olha o `$PATH`; `type` reconhece alias e builtin; `whereis` acha binário, fonte e manual.
- `/etc`, `/bin`, `/sbin`, `/lib` e `/dev` ficam na partição raiz. `/var/tmp` sobrevive ao reboot; `/tmp` costuma ser limpo.