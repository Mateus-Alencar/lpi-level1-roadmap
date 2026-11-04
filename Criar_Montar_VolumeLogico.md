# LPIC-1 — Criar e Montar um Volume Lógico `/Temp`

## 1. Verificações iniciais

```bash
# Lista discos, partições e volumes lógicos
lsblk

# Lista discos e tabelas de partições
sudo fdisk -l

# Lista discos, partições, sistemas de arquivos, UUIDs e pontos de montagem
lsblk -f

# Verifica os sistemas de arquivos montados e o espaço disponível
df -hT

# Verifica os pontos de montagem
findmnt

# Lista os Physical Volumes (PV) do LVM
sudo pvs

# Lista os Volume Groups (VG) e o espaço livre disponível
sudo vgs

# Lista os Logical Volumes (LV) existentes
sudo lvs
```

## 2. Criar o volume lógico Temp

```bash
# Cria um volume lógico de 10 GB chamado Temp dentro do ubuntu-vg
sudo lvcreate -L 10G -n Temp ubuntu-vg

# Verifica os volumes lógicos criados
sudo lvs

# Verifica a estrutura de discos, partições e volumes lógicos
lsblk
```

## 3. Criar o sistema de arquivos

```bash
# Formata o volume lógico Temp com o sistema de arquivos ext4
sudo mkfs.ext4 /dev/ubuntu-vg/Temp

# Verifica o sistema de arquivos e seu UUID
sudo blkid /dev/ubuntu-vg/Temp

# Lista os sistemas de arquivos e UUIDs
lsblk -f
```

## 4. Criar o ponto de montagem e montar o volume

```bash
# Cria o diretório /Temp que será utilizado como ponto de montagem
sudo mkdir /Temp

# Monta o volume lógico Temp no diretório /Temp
sudo mount /dev/ubuntu-vg/Temp /Temp

# Verifica a estrutura de dispositivos e pontos de montagem
lsblk

# Verifica o ponto de montagem específico
findmnt /Temp

# Verifica o espaço disponível e o tipo de sistema de arquivos
df -hT /Temp
```

## 5. Configurar montagem persistente no /etc/fstab

```bash
# Obtém o UUID do volume lógico Temp
sudo blkid /dev/ubuntu-vg/Temp

# Abre o arquivo /etc/fstab para edição
sudo nano /etc/fstab

# Adicionar ao /etc/fstab a seguinte linha, substituindo pelo UUID real:
# UUID=SEU-UUID /Temp ext4 defaults 0 2

# Testa as configurações do /etc/fstab sem reiniciar o sistema
sudo mount -a

# Verifica se o volume está montado corretamente
findmnt /Temp

# Verifica a estrutura final
lsblk -f
```

## 6. Resultado esperado

```text
NAME                      TYPE MOUNTPOINTS
sda                       disk
├─sda1                    part
├─sda2                    part /boot
└─sda3                    part
  ├─ubuntu--vg-ubuntu--lv  lvm  /
  └─ubuntu--vg-Temp        lvm  /Temp
```

## Conceitos importantes

- `PV`: Physical Volume — volume físico utilizado pelo LVM.
- `VG`: Volume Group — grupo que reúne os PVs.
- `LV`: Logical Volume — volume lógico criado dentro do VG.
- `mkfs.ext4`: cria um sistema de arquivos ext4.
- `mount`: associa um sistema de arquivos a um diretório.
- `/etc/fstab`: arquivo responsável por definir montagens persistentes.
- `UUID`: identificador único do sistema de arquivos.
- `lsblk`: lista dispositivos de bloco.
- `pvs`, `vgs` e `lvs`: comandos de verificação do LVM.
