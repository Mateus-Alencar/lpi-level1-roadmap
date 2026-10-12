# Sticky Bit no Linux

> **LPIC-1 – Exame 101 | Tópico 104.5:** Gerenciar permissões e propriedade de arquivos

## O que é

O **sticky bit** é uma permissão especial que, quando aplicada a um **diretório**, restringe quem pode **apagar ou renomear** os arquivos dentro dele.

Sem o sticky bit, qualquer usuário com permissão de escrita (`w`) no diretório pode apagar **qualquer** arquivo dentro dele, mesmo que o arquivo pertença a outro usuário. Com o sticky bit ativo, só podem remover ou renomear um arquivo:

- o **dono do arquivo**;
- o **dono do diretório**;
- o **root**.

## Exemplo clássico: o diretório `/tmp`

O `/tmp` precisa ser gravável por todos os usuários, mas ninguém deve conseguir apagar os arquivos temporários dos outros. O sticky bit resolve exatamente isso.

```bash
$ ls -ld /tmp
drwxrwxrwt 15 root root 4096 out 11 10:00 /tmp
```

Repare no **`t`** no lugar do `x` das permissões de "outros" (`drwxrwxrwt`). Em octal, as permissões são `1777`.

### Demonstração prática

```bash
# Como usuário "ana": cria um arquivo no /tmp
ana$ touch /tmp/arquivo_da_ana

# Como usuário "bruno": tenta apagar o arquivo da ana
bruno$ rm /tmp/arquivo_da_ana
rm: cannot remove '/tmp/arquivo_da_ana': Operation not permitted
```

Mesmo com `rwx` para todos no diretório, o `bruno` não consegue apagar o arquivo da `ana`. Já a própria `ana` (dona do arquivo) e o `root` conseguem.

## Como aplicar e remover

**Modo simbólico:**

```bash
chmod +t /srv/compartilhado     # adiciona o sticky bit
chmod o+t /srv/compartilhado    # equivalente, explícito para "outros"
chmod -t /srv/compartilhado     # remove o sticky bit
```

**Modo octal** (o sticky bit é o dígito `1` na frente das permissões normais):

```bash
chmod 1777 /srv/compartilhado   # rwxrwxrwt
chmod 1770 /srv/compartilhado   # rwxrwx--T
```

## Lendo o resultado no `ls -l`

| Saída | Significado |
|---|---|
| `drwxrwxrwt` | Sticky bit ativo **e** `x` ativo para "outros" |
| `drwxrwxrwT` | Sticky bit ativo, mas **sem** `x` para "outros" (configuração incomum) |
| `drwxrwxrwx` | Sem sticky bit |

- **`t` minúsculo** = sticky + execução.
- **`T` maiúsculo** = sticky **sem** execução.

## Comparação com os outros bits especiais

| Bit | Octal | Efeito principal | Símbolo no `ls -l` |
|---|---|---|---|
| SUID | 4 | Executa com as permissões do **dono** do arquivo | `s` na posição do dono |
| SGID | 2 | Executa com o **grupo** do arquivo; em diretórios, novos arquivos herdam o grupo | `s` na posição do grupo |
| Sticky | 1 | Só o dono do arquivo/diretório (ou root) apaga ou renomeia | `t` na posição de "outros" |

## Localizando diretórios com sticky bit

```bash
find / -type d -perm -1000 2>/dev/null
```

## Pontos que caem na prova

- O sticky bit é aplicado em **diretórios**; em **arquivos** é ignorado no Linux moderno (historicamente mantinha programas na memória/swap).
- Em octal, o bit especial vai no **dígito extra à esquerda**: `1` (sticky), `2` (SGID), `4` (SUID). Podem ser somados (ex.: `3777` = SGID + sticky).
- `t` minúsculo vs. `T` maiúsculo indica se o `x` de "outros" está ativo.
- O exemplo de referência é o `/tmp`, com permissões `1777`.

## Resumo rápido

```text
chmod +t DIR      →  ativa o sticky bit
chmod 1777 DIR    →  rwxrwxrwt (padrão do /tmp)
ls -ld DIR        →  t / T no fim das permissões
find / -type d -perm -1000   →  lista diretórios com sticky bit
```