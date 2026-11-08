# LPIC-1 — Tópico 103: Comandos GNU e Unix

Resumo de revisão (exame 101-500). Ordem numérica: 103.1 → 103.8.

---

# 103.1 — Trabalhar na linha de comando

## Shell e comandos básicos

| Comando | Função | Opções importantes |
|---|---|---|
| `pwd` | Diretório atual | `-L` (lógico, padrão), `-P` (físico, resolve links) |
| `cd` | Muda de diretório | `cd` / `cd ~` = home; `cd -` = volta ao `OLDPWD` |
| `echo` | Imprime texto | `-n` sem quebra de linha; `-e` interpreta `\n`, `\t`; `-E` não interpreta (padrão) |
| `uname` | Info do sistema | `-a` tudo; `-s` kernel; `-n` hostname; `-r` release do kernel; `-v` versão; `-m` hardware (arquitetura); `-p` processador; `-o` SO |
| `type` | Como o shell interpreta um nome (builtin, alias, função, arquivo) | `-a` todas as ocorrências; `-t` só o tipo |
| `which` | Localiza executável no `PATH` | — |
| `command -v` | Alternativa portátil ao `which` | — |
| `history` | Histórico de comandos | ver abaixo |
| `exit` | Sai do shell | — |

## Variáveis

```bash
VAR=valor          # variável local (só no shell atual) — sem espaços ao redor do =
export VAR         # exporta para o ambiente (processos filhos herdam)
export VAR=valor   # cria e exporta
unset VAR          # remove
echo $VAR  ou  echo ${VAR}
```

- `env` → lista variáveis **de ambiente** (exportadas). Também executa comando com ambiente alterado: `env VAR=x comando`, `env -i comando` (ambiente vazio), `env -u VAR comando`.
- `set` → lista **todas** as variáveis (locais + ambiente) e funções do shell. Também liga/desliga opções (`set -o noclobber`).
- `printenv` → variáveis de ambiente.
- Filho **nunca** altera variáveis do pai.

Variáveis comuns: `PATH`, `HOME`, `PWD`, `OLDPWD`, `SHELL`, `USER`, `LANG`, `PS1` (prompt), `EDITOR`, `HISTSIZE`, `HISTFILESIZE`.

Variáveis especiais: `$?` (código de saída do último comando; 0 = sucesso), `$$` (PID do shell), `$!` (PID do último processo em background), `$0` (nome do script), `$1…$9` (parâmetros), `$#` (qtd de parâmetros), `$@`/`$*` (todos os parâmetros).

## Aspas e escape

| Forma | Efeito |
|---|---|
| `'texto'` | **Literal**: nada é expandido (`$`, `` ` ``, `\` ficam como estão) |
| `"texto"` | Expande `$var`, `$(cmd)`, `` `cmd` `` e `\` (parcial); **não** expande globbing (`*`) |
| `\c` | Escapa o caractere seguinte |
| `$(cmd)` ou `` `cmd` `` | **Substituição de comando** (preferir `$(...)`, aceita aninhamento) |

## Encadeamento de comandos

| Operador | Significado |
|---|---|
| `a ; b` | Executa b depois de a, sempre |
| `a && b` | b só se a tiver sucesso (código 0) |
| `a \|\| b` | b só se a **falhar** (código ≠ 0) |
| `a &` | Executa a em background |

## Histórico (bash)

- Arquivo: `~/.bash_history`. Tamanho em memória: `HISTSIZE`; no arquivo: `HISTFILESIZE`.
- O histórico da sessão é **gravado ao sair do shell** (ou com `history -w`).
- `history N` → últimos N; `history -c` → limpa a memória; `history -d N` → apaga a linha N; `history -w` → grava arquivo; `history -a` → anexa as novas linhas; `history -r` → lê do arquivo.
- Atalhos: `!!` (último comando), `!N` (comando número N), `!-N` (N-ésimo anterior), `!texto` (último que começa com texto), `!?texto` (contém texto), `^velho^novo` (repete trocando).
- `Ctrl+R` → busca reversa incremental.
- `HISTCONTROL=ignorespace:ignoredups` (ignora comandos que começam com espaço e duplicados); `HISTTIMEFORMAT` mostra data/hora.

## Edição na linha de comando (readline)

`Ctrl+A` início · `Ctrl+E` fim · `Ctrl+U` apaga até o início · `Ctrl+K` apaga até o fim · `Ctrl+W` apaga palavra anterior · `Ctrl+L` limpa tela · `Ctrl+C` interrompe (SIGINT) · `Ctrl+D` EOF / sai do shell · `Ctrl+Z` suspende (SIGTSTP) · `Tab` completa.

## Arquivos de configuração do bash

| Tipo de shell | Arquivos lidos |
|---|---|
| **Login** (`bash -l`, `su -`, login no console/ssh) | `/etc/profile` (+ `/etc/profile.d/*.sh`), depois **o primeiro que existir** entre `~/.bash_profile`, `~/.bash_login`, `~/.profile` |
| **Interativo não-login** (abrir terminal gráfico, `bash`) | `/etc/bash.bashrc` (Debian) ou `/etc/bashrc` (RHEL), e `~/.bashrc` |
| Ao sair de login shell | `~/.bash_logout` |

- Convenção: `~/.bash_profile` costuma fazer `source ~/.bashrc`.
- `source arquivo` ou `. arquivo` → executa no **shell atual** (aplica mudanças sem relogar).
- `/etc/skel` → modelo copiado para a home de novos usuários.

## Alias e funções

```bash
alias ll='ls -l'     # lista aliases se sem argumentos
unalias ll           # remove (-a remove todos)
minhafunc() { echo "oi $1"; }
unset -f minhafunc
```

## Ajuda

- `man comando` → páginas de manual. Seções:

| Seção | Conteúdo |
|---|---|
| 1 | Comandos de usuário |
| 2 | Chamadas de sistema |
| 3 | Funções de biblioteca |
| 4 | Arquivos especiais / dispositivos (`/dev`) |
| 5 | Formatos de arquivos e configuração |
| 6 | Jogos |
| 7 | Miscelânea |
| 8 | Administração do sistema |
| 9 | Rotinas do kernel |

- `man 5 passwd` → seção específica. `man -k termo` = `apropos termo` (busca por palavra-chave). `man -f cmd` = `whatis cmd` (descrição curta). `man -a` mostra todas as seções. `man -w` mostra o caminho do arquivo. `MANPATH` define onde procurar.
- Dentro do man (usa `less`): `/texto`, `n`, `N`, `q`.
- `info comando` → documentação GNU em hipertexto. `/usr/share/doc/` → docs dos pacotes. `comando --help`.

### Pegadinhas 103.1
- `env` vs `set`: `env` só ambiente; `set` tudo.
- Aspas simples **não** expandem variável; duplas expandem.
- `export` não funciona "para trás": filho não altera o pai.
- Login shell lê só **um** dos três (`.bash_profile`, `.bash_login`, `.profile`).
- `uname -r` = release do kernel; `uname -m` = arquitetura.

---

# 103.2 — Processar fluxos de texto usando filtros

Filtros leem de **stdin** (ou arquivos) e escrevem em **stdout**; combinam-se com pipes.

## Visualização

| Comando | Função / opções |
|---|---|
| `cat` | Concatena e exibe. `-n` numera todas; `-b` numera só as não vazias; `-A` mostra `$` no fim de linha e `^I` nos tabs; `-E` `$` no fim; `-T` tabs como `^I`; `-s` comprime linhas em branco |
| `tac` | Inverte a ordem das **linhas** |
| `rev` | Inverte os **caracteres** de cada linha |
| `head` | Primeiras 10 linhas. `-n N` / `-N`; `-n -N` = tudo **menos** as últimas N; `-c N` bytes |
| `tail` | Últimas 10 linhas. `-n N`; `-n +N` = **a partir** da linha N; `-c N` bytes; `-f` acompanha o arquivo crescendo; `-F` acompanha pelo **nome** (reabre se o arquivo for rotacionado/recriado) |
| `less` | Paginador (rola nos dois sentidos, não carrega tudo). `Espaço`/`f` avança, `b` volta, `/` `?` busca, `n`/`N` próxima/anterior, `g`/`G` início/fim, `q` sai, `-N` numera linhas, `F` segue como `tail -f` |
| `more` | Paginador mais simples (tradicionalmente só avança) |
| `zcat` / `bzcat` / `xzcat` | Mostram conteúdo de `.gz` / `.bz2` / `.xz` sem descompactar o arquivo (`zcat` = `gzip -dc`). Também `zless`, `zgrep` |

## Seleção e junção de colunas

| Comando | Detalhes |
|---|---|
| `cut` | `-d 'x'` delimitador (padrão **TAB**); `-f 1,3` / `-f 2-` campos; `-c 1-5` caracteres; `-b` bytes; `--complement` inverte seleção; `-s` ignora linhas sem delimitador; `--output-delimiter=` |
| `paste` | Junta arquivos **lado a lado** (padrão TAB). `-d ','` delimitador; `-s` serial (cada arquivo vira uma linha) |
| `join` | Junta arquivos por **campo comum** (como um JOIN SQL); entradas devem estar **ordenadas** pelo campo. `-1 N` / `-2 N` campo de cada arquivo; `-t ':'` separador; `-a 1` mostra não pareadas do arquivo 1; `-v 1` só as não pareadas; `-e` valor p/ vazios; `-o` formato de saída |

## Ordenação e contagem

**`sort`** (ordem alfabética conforme locale por padrão):

| Opção | Efeito |
|---|---|
| `-n` | Numérica |
| `-r` | Reversa |
| `-k N` / `-k N,M` | Chave (campo) N, ou de N até M |
| `-t ':'` | Delimitador de campos (padrão: transição de branco para não-branco) |
| `-u` | Remove duplicadas (únicas) |
| `-f` | Ignora maiúsc./minúsc. |
| `-b` | Ignora espaços iniciais |
| `-h` | Numérica "humana" (1K, 2M, 3G) |
| `-M` | Meses (JAN…DEZ) |
| `-V` | Versões (1.2 < 1.10) |
| `-c` | Só verifica se já está ordenado |
| `-o arq` | Grava em arquivo (seguro para sobrescrever o próprio arquivo de entrada) |
| `-s` | Estável |

**`uniq`**: remove linhas **adjacentes** repetidas → normalmente usado após `sort`. `-c` conta ocorrências; `-d` só as repetidas (uma de cada); `-D` todas as repetidas; `-u` só as que não repetem; `-i` ignora caixa; `-f N` ignora N campos; `-s N` ignora N caracteres.

**`wc`**: sem opção mostra linhas, palavras e bytes. `-l` linhas; `-w` palavras; `-c` bytes; `-m` caracteres; `-L` comprimento da maior linha. `wc -l < arq` mostra só o número (sem nome do arquivo).

## Transformação de texto

**`tr`** (só lê stdin; **não** aceita arquivo como argumento):
```bash
tr 'a-z' 'A-Z'          # converte
tr '[:lower:]' '[:upper:]'
tr -d '\r'              # apaga caracteres
tr -s ' '               # comprime repetições (squeeze)
tr -c 'a-z' '_'         # -c complementa o conjunto
```

| Comando | Função |
|---|---|
| `expand` | TAB → espaços (`-t N`, padrão 8) |
| `unexpand` | Espaços → TAB (`-a` todos; padrão só os iniciais) |
| `fmt` | Reformata parágrafos. `-w N` largura (padrão 75); `-u` espaçamento uniforme |
| `pr` | Formata para impressão (paginação). `-l N` linhas/página (66), `-h "título"`, `-d` espaço duplo, `-t` sem cabeçalho, `-N` N colunas |
| `nl` | Numera linhas. **Padrão `-b t`: só linhas não vazias**; `-b a` todas; `-n ln`, `rn` ou `rz` (alinhamento); `-w N` largura; `-s 'sep'` separador |
| `od` | Dump octal (padrão: octal em palavras de 2 bytes). `-t x1` hex byte; `-c` caracteres; `-A x`, `d`, `o` ou `n` (base do offset); `-N n` limita bytes |
| `hexdump -C` / `xxd` | Dump hexadecimal + ASCII (`xxd -r` reverte) |
| `split` | Divide arquivo. **Padrão: 1000 linhas**, prefixo `x` → `xaa`, `xab`… `-l N` linhas; `-b N` bytes (aceita `K`,`M`); `-d` sufixo numérico; `-a N` tamanho do sufixo. Uso: `split -l 100 arq prefixo_` |
| `tee` | Copia stdin para stdout **e** arquivos. `-a` anexa |

## Somas de verificação

```bash
md5sum arq > arq.md5
sha256sum arq      # saída: HASH  nome
sha512sum arq
sha256sum -c arq.sha256     # verifica (mostra OK / FAILED)
```
Também `sha1sum`. Usados para checar integridade de downloads.

## sed — editor de fluxo

Usa **BRE** por padrão; `-E` (ou `-r`) para ERE. Não altera o arquivo, a menos que `-i`.

```bash
sed 's/velho/novo/'        # troca a 1ª ocorrência de cada linha
sed 's/velho/novo/g'       # todas na linha
sed 's/velho/novo/2'       # só a 2ª ocorrência
sed 's/velho/novo/I'       # ignora caixa
sed -n '2,4p'              # imprime só linhas 2 a 4 (-n suprime saída padrão)
sed '3d'  /  '2,5d'  /  '$d'  /  '/^#/d'  /  '/^$/d'    # deleta
sed '/padrão/!d'           # ! nega o endereço
sed -i 's/a/b/' arq        # edita no arquivo; -i.bak guarda backup
sed -e 's/a/b/' -e 's/c/d/'  # vários comandos (ou separados por ;)
sed -f script.sed arq      # comandos vindos de arquivo
sed 'y/abc/xyz/'          # transliteração (como tr)
sed 's#/usr/bin#/opt#'     # delimitador alternativo
sed 's/.*/[&]/'            # & = texto casado
sed '2i\texto'  /  '2a\texto'  /  '2c\texto'   # insere antes / anexa depois / substitui linha
sed -n '/ini/,/fim/p'      # intervalo por padrões
sed '=' arq                # imprime nº da linha
```

### Pegadinhas 103.2
- `uniq` só remove **adjacentes** → precisa de `sort` antes (ou `sort -u`).
- `nl` **não** numera linhas em branco por padrão (`cat -n` numera).
- `cut` não reordena campos (`-f3,1` sai como `1,3`).
- `tr` não aceita nome de arquivo (use `<`).
- `tail -f` × `tail -F`: o segundo sobrevive a rotação de logs.
- `sed` sem `-i` só imprime; o arquivo original fica intacto.
- `join` exige entrada ordenada.
- `head -n -3` ≠ `head -n 3`.

---

# 103.3 — Gerenciamento básico de arquivos

## Globbing (curingas do shell — expandidos pelo shell, não pelo comando)

| Padrão | Casa com |
|---|---|
| `*` | Qualquer sequência (inclusive vazia); **não** casa arquivos ocultos (começam com `.`) |
| `?` | Exatamente 1 caractere |
| `[abc]` / `[a-z]` | 1 caractere do conjunto/intervalo |
| `[!abc]` ou `[^abc]` | 1 caractere **fora** do conjunto |
| `[[:digit:]]` | Classes POSIX (`alpha`, `alnum`, `upper`, `lower`, `digit`, `space`, `punct`) |
| `{a,b,c}` | **Brace expansion** (não é globbing): `arq{1,2}.txt` → `arq1.txt arq2.txt`; `{1..5}` |

## Comandos de arquivo

**`ls`**: `-l` longo; `-a` todos (inclui `.` e `..`); `-A` todos menos `.` e `..`; `-d` mostra o próprio diretório; `-R` recursivo; `-h` tamanhos legíveis; `-t` por data de modificação (mais novo primeiro); `-r` inverte; `-S` por tamanho; `-i` inode; `-F` marca tipo (`/` dir, `*` exec, `@` link); `-1` uma por linha; `-u` atime; `-c` ctime; `--full-time`.

Primeiro caractere do `ls -l`: `-` arquivo, `d` diretório, `l` link simbólico, `c` dispositivo de caractere, `b` dispositivo de bloco, `p` pipe nomeado (FIFO), `s` socket. Colunas: tipo+permissões, nº de links, dono, grupo, tamanho, data, nome.

**`cp`**: `-r`/`-R` recursivo (necessário para diretórios); `-a` arquivamento (= `-dR --preserve=all`: preserva tudo, links como links); `-p` preserva modo, dono e datas; `-i` pergunta antes de sobrescrever; `-f` força; `-u` só se origem for mais nova; `-v` verboso; `-n` não sobrescreve; `-l` cria hard links; `-s` cria links simbólicos; `-t DIR` destino primeiro.

**`mv`**: renomeia/move. `-i`, `-f`, `-n`, `-u`, `-v`, `-b` (backup). Dentro do mesmo filesystem mantém o inode (só muda a entrada do diretório).

**`rm`**: `-r` recursivo; `-f` força (sem erro se não existir); `-i` pergunta; `-v`; `-d` remove diretório vazio. Para arquivo começando com `-`: `rm -- -arq` ou `rm ./-arq`.

**`mkdir`**: `-p` cria pais e não dá erro se existe; `-m 755` modo; `-v`. **`rmdir`**: só diretório **vazio**; `-p` remove também os pais vazios.

**`touch`**: cria arquivo vazio ou atualiza datas. `-a` só atime; `-m` só mtime; `-c` não cria; `-t [[CC]AA]MMDDhhmm[.ss]`; `-d "2026-11-04 10:00"`; `-r ref` copia datas de outro arquivo.

**`file`**: identifica o tipo pelo conteúdo (números mágicos, `/usr/share/misc/magic`). `-i` MIME; `-b` sem nome; `-L` segue links.

**`stat`**: inode, tamanho, blocos, permissões, links, atime/mtime/ctime. `-c FORMATO`, `-f` filesystem.

### Os três tempos
- **atime** = último acesso (leitura) · **mtime** = última modificação do **conteúdo** · **ctime** = última mudança do **inode** (permissões, dono, links; também muda quando o conteúdo muda). ctime **não** é data de criação.

## find

```bash
find [caminho...] [expressão]       # caminho padrão: .   ação padrão: -print
```

| Critério | Significado |
|---|---|
| `-name 'padrão'` / `-iname` | Nome (glob, **entre aspas**); `-iname` ignora caixa |
| `-path`, `-regex` | Caminho completo; regex sobre o caminho completo |
| `-type f d l b c p s` | Tipo |
| `-size +10M` / `-10k` / `100c` | Maior que / menor que / exatamente. Unidades: `c` bytes, `k`, `M`, `G`, `b` (blocos de **512 bytes — padrão se omitir unidade**). Arredonda para cima |
| `-user`, `-group`, `-nouser`, `-nogroup` | Dono/grupo (ou órfãos) |
| `-perm 644` | Permissão **exata** |
| `-perm -644` | **Todos** os bits indicados (pode ter mais) |
| `-perm /644` | **Algum** dos bits |
| `-mtime n` / `-atime` / `-ctime` | Em **dias** (n×24h). `+n` = mais de n dias; `-n` = menos de n dias; `n` = exatamente n |
| `-mmin n` / `-amin` / `-cmin` | Mesmo, em **minutos** |
| `-newer ref` | Modificado depois de `ref` |
| `-empty` | Arquivo/dir vazio |
| `-maxdepth N` / `-mindepth N` | Limita profundidade (colocar **antes** dos outros testes) |
| `-links N`, `-inum N`, `-samefile` | Por links, inode |

Operadores: `-a` (E, implícito), `-o` (OU), `!` ou `-not`, `\( ... \)` agrupamento.

Ações:
```bash
-print            # padrão
-print0           # separa por NUL (para xargs -0)
-ls
-delete
-exec cmd {} \;   # executa cmd 1 vez por arquivo
-exec cmd {} +    # junta vários arquivos numa só execução (como xargs)
-ok cmd {} \;     # igual -exec, mas pede confirmação
```
Exemplos:
```bash
find / -name '*.log' -mtime +7 -exec rm {} \;
find /home -user joao -size +100M
find . -type f -perm /111          # algum bit de execução
find . -maxdepth 1 -type d
```
`-mtime -1` = modificado nas últimas 24h; `-mtime +1` = há **2 dias ou mais** (fração ignorada).

## locate / updatedb

- `locate` consulta um **banco de dados** (`mlocate.db`/`plocate.db`, em `/var/lib/...`), por isso é rápido, mas pode estar **desatualizado** e não achar arquivos recentes.
- `updatedb` (root) atualiza o banco (normalmente via cron/timer diário). Configuração em `/etc/updatedb.conf` (`PRUNEPATHS`, `PRUNEFS`).
- Opções: `-i` ignora caixa; `-c` só conta; `-n N` limita; `-r` regex; `-b` só nome base; `-e` só os que ainda existem; `-S` estatísticas.
- Padrão sem curinga casa em **qualquer parte** do caminho.

## whereis / which / type
- `which` → executável no `PATH`. `whereis` → binário, fonte e man (`-b`, `-m`, `-s`). `type` → leva em conta builtins, aliases e funções.

## Links

| | Hard link | Link simbólico |
|---|---|---|
| Criar | `ln origem destino` | `ln -s alvo nome_do_link` |
| Inode | **Mesmo inode** do original | Inode próprio |
| Entre filesystems | **Não** | Sim |
| Para diretórios | **Não** | Sim |
| Se o original for removido | Dados continuam | Link fica **quebrado** (dangling) |
| `ls -l` | Contagem de links aumenta | Mostra `l...  nome -> alvo` |

- Caminho relativo em link simbólico é relativo ao **local do link**, não ao diretório atual.
- `ln -f` sobrescreve; `readlink` mostra o alvo; `ls -i` mostra inodes.

## dd

```bash
dd if=ENTRADA of=SAIDA bs=TAMANHO count=N [skip=N] [seek=N] [conv=...] [status=progress]
dd if=/dev/zero of=teste bs=1M count=10
dd if=/dev/sda of=mbr.bak bs=512 count=1      # backup do MBR (512 bytes)
dd if=imagem.iso of=/dev/sdX bs=4M status=progress
```
- `if`/`of` sem opção → stdin/stdout. `bs` bloco; `count` nº de blocos; `skip` blocos pulados na **entrada**; `seek` blocos pulados na **saída**.
- `conv=notrunc` (não trunca saída), `noerror` (continua após erro de leitura), `sync` (preenche com zeros), `ucase`/`lcase`.
- Sintaxe `chave=valor`, sem traços. `kill -USR1 PID` faz o dd mostrar progresso.

## Arquivamento e compressão

### tar
| Opção | Função |
|---|---|
| `-c` | Cria |
| `-x` | Extrai |
| `-t` | Lista |
| `-r` | Acrescenta ao fim |
| `-u` | Acrescenta só o que é mais novo |
| `-A` | Concatena arquivos tar |
| `-d` | Compara com o sistema (diff) |
| `-f arq` | Arquivo (obrigatório na prática; `-f -` = stdin/stdout) |
| `-v` | Verboso |
| `-z` / `-j` / `-J` | gzip / bzip2 / xz |
| `-a` | Escolhe compressão pela extensão |
| `-C dir` | Muda de diretório antes |
| `-p` | Preserva permissões |
| `--exclude=padrão` | Exclui |
| `--strip-components=N` | Remove N níveis do caminho ao extrair |

```bash
tar -cvf a.tar dir/         tar -czvf a.tar.gz dir/      tar -cjvf a.tar.bz2 dir/     tar -cJvf a.tar.xz dir/
tar -xvf a.tar -C /destino  tar -tvf a.tar               tar -xf a.tar caminho/arq    # extrai só um item
```
- Remove a `/` inicial dos caminhos (a menos que `-P`). Não se acrescenta (`-r`/`-u`) a arquivo **comprimido**.
- O GNU tar detecta a compressão ao **ler** (`tar xf` basta).

### Compressores (substituem o original, a menos que `-k` ou `-c`)

| | gzip | bzip2 | xz |
|---|---|---|---|
| Extensão | `.gz` | `.bz2` | `.xz` |
| Descomprimir | `gunzip` / `gzip -d` | `bunzip2` / `bzip2 -d` | `unxz` / `xz -d` |
| Ver sem extrair | `zcat` | `bzcat` | `xzcat` |
| Taxa de compressão | menor | média | **maior** (e mais lento) |

Opções comuns: `-k` mantém original; `-c` escreve na saída padrão; `-1` … `-9` nível (padrão 6 no gzip e xz, 9 no bzip2); `-t` testa; `-l` lista (gzip/xz); `-v`; `-r` recursivo (só gzip); `-T N` threads (xz).

### cpio
Lê a **lista de arquivos pela entrada padrão** (tipicamente `find` ou `ls`).

| Modo | Função |
|---|---|
| `-o` (copy-out) | Cria arquivo (escreve em stdout / `-O arq`) |
| `-i` (copy-in) | Extrai (lê stdin / `-I arq`) |
| `-p` (pass-through) | Copia árvore para outro diretório |

Complementos: `-t` lista; `-d` cria diretórios; `-m` preserva mtime; `-u` sobrescreve; `-v`; `-H newc|tar` formato; `-0` entrada separada por NUL.
```bash
find . -depth | cpio -o > backup.cpio
cpio -idv < backup.cpio
cpio -tv < backup.cpio
find . | cpio -pdm /destino
```

### Pegadinhas 103.3
- `cp` de diretório exige `-r`.
- `-name` precisa de aspas para o shell não expandir o glob.
- `find -size 10` = 10 blocos de 512 bytes (não 10 bytes).
- `-perm -mode` (todos os bits) × `-perm /mode` (qualquer bit).
- `-exec ... \;` × `+`.
- `locate` não acha arquivo criado depois do último `updatedb`.
- Hard link: mesmo inode e mesmo filesystem; não vale para diretório.
- `dd`: `skip` = entrada, `seek` = saída.
- ctime ≠ data de criação.
- `cpio` precisa receber a lista de arquivos via stdin.

---

# 103.4 — Fluxos, pipes e redirecionamentos

## Descritores padrão

| FD | Nome | Padrão |
|---|---|---|
| 0 | stdin | teclado |
| 1 | stdout | terminal |
| 2 | stderr | terminal |

## Redirecionamentos

| Sintaxe | Efeito |
|---|---|
| `cmd > arq` | stdout → arquivo (**sobrescreve**) |
| `cmd >> arq` | stdout → arquivo (**anexa**) |
| `cmd 2> arq` / `2>> arq` | stderr → arquivo (sobrescreve / anexa) |
| `cmd &> arq` ou `cmd > arq 2>&1` | stdout **e** stderr → arquivo |
| `cmd &>> arq` | idem, anexando |
| `cmd 2>&1` | stderr vai para onde stdout está apontando |
| `cmd 1>&2` ou `>&2` | stdout → stderr |
| `cmd < arq` | stdin vem do arquivo |
| `cmd << FIM` | **here document**: entrada até a linha `FIM` |
| `cmd <<- FIM` | here doc que ignora **TABs** iniciais |
| `cmd << 'FIM'` | here doc sem expansão de variáveis |
| `cmd <<< "texto"` | **here string** |
| `cmd > /dev/null 2>&1` | descarta tudo |
| `cmd >| arq` | sobrescreve mesmo com `noclobber` |

- **A ordem importa:** `cmd > arq 2>&1` (ambos no arquivo) ≠ `cmd 2>&1 > arq` (stderr no terminal, só stdout no arquivo).
- `set -o noclobber` impede `>` de sobrescrever arquivo existente (`>|` força).
- `> arq` sozinho cria/esvazia o arquivo.

## Pipes

- `a | b` → stdout de a vira stdin de b. **Só stdout** passa; stderr continua no terminal.
- `a |& b` (bash) → stdout **e** stderr passam pelo pipe (= `a 2>&1 | b`).
- `mkfifo nome` → pipe nomeado (FIFO).

## tee

```bash
cmd | tee arq            # tela + arquivo (sobrescreve)
cmd | tee -a arq         # anexa
cmd | tee a b            # vários arquivos
cmd 2>&1 | tee log       # inclui stderr
echo x | sudo tee /etc/arq    # truque: redirecionamento > roda com as permissões do usuário, tee com as do sudo
```

## xargs

Monta linhas de comando a partir da entrada padrão (padrão: comando `echo`; separa por espaços/novas linhas; executa quantas vezes necessário).

| Opção | Efeito |
|---|---|
| `-n N` | No máximo N argumentos por execução |
| `-L N` | N linhas por execução |
| `-I {}` | Define marcador de substituição (uma execução por linha de entrada) |
| `-0` | Separador NUL (par de `find -print0`) |
| `-d 'c'` | Delimitador customizado |
| `-p` | Pergunta antes de cada execução |
| `-t` | Mostra o comando antes de executar |
| `-r` | Não executa se a entrada for vazia (GNU executa 1 vez por padrão) |
| `-P N` | Execução em paralelo |

```bash
find . -name '*.tmp' -print0 | xargs -0 rm
cat lista.txt | xargs -I {} cp {} /backup/
echo a b c | xargs -n1
```

## Substituição de comandos
`$(cmd)` / `` `cmd` `` → substitui pela saída. Process substitution (bash): `diff <(sort a) <(sort b)`.

### Pegadinhas 103.4
- `2>&1` **depois** do `>` arquivo.
- Pipe não carrega stderr (a menos que `2>&1` ou `|&`).
- `>` apaga antes de executar: `sort arq > arq` **zera** o arquivo (use `sort -o arq arq`).
- `xargs` com nomes com espaço → `-print0 | xargs -0`.
- `wc -l < arq` × `wc -l arq` (este mostra o nome).
- `<<` (here doc) × `<<<` (here string).

---

# 103.5 — Criar, monitorar e finalizar processos

## Jobs (controle de tarefas do shell)

| Ação | Como |
|---|---|
| Rodar em background | `cmd &` (mostra `[job] PID`; `$!` guarda o PID) |
| Suspender o processo em foreground | `Ctrl+Z` (envia SIGTSTP) |
| Retomar em background | `bg %N` |
| Trazer para foreground | `fg %N` |
| Listar | `jobs` (`-l` com PID, `-p` só PIDs, `-r` rodando, `-s` parados) |
| Matar job | `kill %N` |

Especificadores: `%N` (número), `%%` ou `%+` (atual), `%-` (anterior), `%texto` (começa com), `%?texto` (contém).

- Jobs pertencem ao shell/terminal: ao fechar o terminal, o shell envia **SIGHUP** e os filhos morrem.
- `nohup cmd &` → ignora SIGHUP; se stdout for terminal, grava em `nohup.out`.
- `disown` → remove o job da tabela do shell (`-h` marca para não receber SIGHUP mantendo na lista; `-a` todos; `-r` só os rodando).
- **screen** e **tmux** → multiplexadores de terminal; sessões sobrevivem à desconexão.
  - screen: `screen` (nova), `Ctrl+A d` (detach), `screen -ls`, `screen -r` (reanexar), `screen -S nome`.
  - tmux: `tmux` / `tmux new -s nome`, `Ctrl+B d` (detach), `tmux ls`, `tmux attach` (`a`) / `attach -t nome`.

## ps

Aceita três estilos: UNIX (`-ef`), BSD (`aux`, sem traço) e GNU (`--forest`).

| Comando | Resultado |
|---|---|
| `ps` | Processos do **terminal/shell atual** |
| `ps -e` / `-A` | Todos os processos |
| `ps -ef` | Todos, formato completo: `UID PID PPID C STIME TTY TIME CMD` |
| `ps aux` | Todos, com donos e sem terminal: `USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND` |
| `ps -u user` | Do usuário |
| `ps -p PID` | PID específico |
| `ps -C nome` | Pelo nome do comando |
| `ps -l` | Formato longo (mostra `F`, `S`, `PRI`, `NI`) |
| `ps --forest` / `-H` | Árvore hierárquica |
| `ps -o pid,ppid,ni,%cpu,%mem,cmd` | Colunas escolhidas |
| `ps aux --sort=-%mem` | Ordena (− = decrescente) |

**Estados (STAT/S):** `R` executando/pronto · `S` dormindo (interrompível) · `D` dormindo ininterruptível (normalmente I/O; não responde a sinais) · `T` parado · `Z` **zumbi** (terminou, pai não coletou o status) · `I` thread ociosa do kernel. Sufixos: `s` líder de sessão, `+` foreground, `<` alta prioridade, `N` baixa prioridade, `l` multithread.

## Outros comandos de processos

| Comando | Função |
|---|---|
| `pstree` | Árvore de processos (`-p` mostra PIDs, `-u` mudanças de usuário, `-a` argumentos) |
| `pgrep padrão` | Lista **PIDs** por nome (regex). `-l` nome, `-a` linha de comando completa, `-u user`, `-f` casa na linha de comando inteira, `-x` nome exato, `-c` conta, `-n` mais novo, `-o` mais antigo |
| `pkill padrão` | Envia sinal por nome (mesmos filtros do pgrep; `pkill -9 -u joao`) |
| `pidof nome` | PID(s) de um programa |
| `killall nome` | Envia sinal a **todos** os processos com esse nome (`-i` interativo, `-u user`, `-l` lista sinais) |

## Sinais e kill

| Nº | Nome | Significado |
|---|---|---|
| 1 | SIGHUP | Terminal fechado; em daemons costuma **recarregar configuração** |
| 2 | SIGINT | Interrupção (`Ctrl+C`) |
| 3 | SIGQUIT | Sair com core dump (`Ctrl+\`) |
| 9 | SIGKILL | Mata **imediatamente**; **não pode ser capturado nem ignorado** |
| 15 | SIGTERM | Término educado — **padrão do `kill`** |
| 18 | SIGCONT | Continua processo parado |
| 19 | SIGSTOP | Para o processo (**não pode ser capturado**) |
| 20 | SIGTSTP | Suspensão pelo terminal (`Ctrl+Z`; pode ser capturado) |

```bash
kill PID             # SIGTERM (15)
kill -9 PID   |   kill -KILL PID   |   kill -s KILL PID
kill -l              # lista sinais (kill -l 9 → KILL)
kill %1              # job
```
- Usuário comum só mata **seus** processos; root mata qualquer um.
- **Zumbi** não pode ser morto (já está morto): resolve-se encerrando/ajustando o **pai**. **Órfãos** são adotados pelo PID 1 (init/systemd).
- Ordem prática: tentar `15` primeiro; usar `9` só se não responder.

## top

Cabeçalho: hora/uptime/usuários, **load average** (médias de 1, 5 e 15 min), tarefas (running, sleeping, stopped, zombie), CPU (`us` usuário, `sy` sistema, `ni` nice, `id` ocioso, `wa` espera de I/O, `hi`, `si`, `st` roubado por virtualização), memória e swap.

Colunas: `PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND`.

Teclas interativas:

| Tecla | Ação |
|---|---|
| `k` | Mata processo (pede PID e sinal) |
| `r` | **renice** (pede PID e valor) |
| `M` | Ordena por memória |
| `P` | Ordena por CPU |
| `N` | Ordena por PID |
| `T` | Ordena por tempo de CPU |
| `R` | Inverte ordenação |
| `1` | Mostra cada CPU separadamente |
| `u` | Filtra por usuário |
| `c` | Alterna nome/linha de comando completa |
| `H` | Mostra threads |
| `d` ou `s` | Intervalo de atualização (padrão 3 s) |
| `f` | Escolhe campos |
| `W` | Salva configuração |
| `h` / `?` | Ajuda |
| `q` | Sai |

Opções: `-d seg`, `-n N` (nº de atualizações e sai), `-b` (batch, para scripts), `-p PID[,PID]`, `-u user`, `-H`.

## uptime, free, watch

- `uptime` → hora atual, tempo ligado, usuários e load average (1, 5, 15 min). Mesmo dado em `/proc/loadavg`.
- `free` → colunas `total used free shared buff/cache available`. Opções: `-b -k -m -g` unidades, `-h` legível, `-s N` repete a cada N s, `-t` total, `-w` wide. **`available`** = estimativa do que está disponível para novos programas sem usar swap. Fonte: `/proc/meminfo`.
- `watch [-n seg] [-d] 'cmd'` → repete o comando (padrão **2 s**), `-d` destaca diferenças, `-t` sem cabeçalho. Sai com `Ctrl+C`. Use aspas se houver pipe.

### Pegadinhas 103.5
- `ps` sozinho mostra só o terminal atual; `ps -e` ou `ps aux` mostram tudo.
- `kill` sem sinal = **15**, não 9.
- SIGKILL e SIGSTOP não podem ser capturados; `Ctrl+Z` é SIGTSTP (capturável).
- `nohup` ≠ `&`: `&` só põe em background; sem `nohup`/`disown`, SIGHUP ainda mata o processo.
- Zumbi (`Z`) não se mata com `kill -9`.
- `pgrep` mostra PID; `pkill` envia sinal; `killall` age por nome exato.
- Load average **não** é % de CPU.
- `top` → `k` mata, `r` renice.

---

# 103.6 — Modificar a prioridade de execução de um processo

## Conceito
- **Niceness (NI)** vai de **−20** (maior prioridade) a **+19** (menor prioridade). Padrão: **0**.
- Quanto **mais "nice"** (valor alto), mais o processo cede CPU.
- Processos filhos **herdam** o nice do pai.
- No `top`, `PR` ≈ 20 + NI (para processos normais); `rt` indica tempo real.

## nice — inicia o processo com nice ajustado

```bash
nice cmd                 # sem -n: adiciona +10 (padrão)
nice -n 5 cmd            # soma 5 ao nice herdado
nice -n -5 cmd           # reduz (só root)
nice --5 cmd             # sintaxe antiga para -5 (nice -5 = +5)
nice                     # sem argumentos: imprime o nice atual
```

## renice — altera processo **já em execução**

```bash
renice -n 10 -p 1234        # define nice = 10 para o PID (valor absoluto)
renice 10 1234              # sintaxe antiga
renice -n 5 -u joao         # todos os processos do usuário
renice -n 5 -g grupo        # grupo de processos
```
- `renice` **define** o valor (absoluto); `nice -n` **soma** ao valor herdado.
- Usuário comum: só em **seus** processos e só pode **aumentar** o nice (diminuir prioridade); não pode voltar atrás nem usar valores negativos. **Root** pode tudo.

## Onde ver
- `ps -l` (coluna `NI`, e `PRI`), `ps -o pid,ni,cmd`, `top` (colunas `PR` e `NI`; tecla `r` para renice).

### Pegadinhas 103.6
- Valor **menor** = prioridade **maior**.
- `nice` sem `-n` aplica **+10**.
- Usuário comum não reduz nice (nem volta ao valor anterior).
- `nice -n` é relativo; `renice` é absoluto.

---

# 103.7 — Procurar em arquivos de texto usando expressões regulares

## Metacaracteres

| Elemento | BRE (básica) | ERE (estendida) | Significado |
|---|---|---|---|
| `.` | `.` | `.` | Um caractere qualquer |
| `*` | `*` | `*` | Zero ou mais do anterior |
| `+` | `\+` | `+` | Um ou mais do anterior |
| `?` | `\?` | `?` | Zero ou um |
| `{n}` `{n,}` `{n,m}` | `\{n,m\}` | `{n,m}` | Repetições |
| Alternância (ou) | barra invertida + barra vertical (GNU) | barra vertical sozinha | `a`/`b`: casa a ou b |
| Grupo | `\( \)` | `( )` | Agrupamento / captura |
| Retrovisor | `\1` … `\9` | `\1` … `\9` | Repete o que o grupo casou |
| `^` | `^` | `^` | Início da linha |
| `$` | `$` | `$` | Fim da linha |
| `[abc]` `[a-z]` | igual | igual | Conjunto |
| `[^abc]` | igual | igual | Negação |
| `\<` `\>` | igual | igual | Início / fim de palavra |
| `\b` `\B` | igual | igual | Borda de palavra / não-borda |
| `\w` `\W` | igual | igual | `[A-Za-z0-9_]` / negação |
| `\s` `\S` | igual | igual | Espaço em branco / negação |

Regra: em **BRE** `+ ? | { } ( )` são literais e precisam de `\` para virar operador (extensão GNU); em **ERE** ocorre o inverso (`\+` seria um `+` literal).

Classes POSIX (dentro de colchetes duplos): `[[:alpha:]] [[:digit:]] [[:alnum:]] [[:upper:]] [[:lower:]] [[:space:]] [[:punct:]] [[:blank:]]`.

Regras dentro de `[]`: `]` deve vir **primeiro**; `-` por último (ou primeiro); `^` só nega se for o primeiro; metacaracteres perdem o significado (`[.]` = ponto literal).

Outros: `\.` `\*` `\$` escapam o literal. Repetições são **gulosas** (casam o máximo possível). `.*` casa qualquer sequência.

## grep

`grep [opções] padrão [arquivos]` — usa **BRE** por padrão.

| Opção | Efeito |
|---|---|
| `-E` | ERE (= `egrep`) |
| `-F` | Texto **fixo**, sem regex (= `fgrep`) |
| `-P` | Perl-compatível |
| `-i` | Ignora caixa |
| `-v` | **Inverte** (linhas que NÃO casam) |
| `-c` | Conta **linhas** que casam |
| `-n` | Mostra nº da linha |
| `-o` | Mostra só o trecho casado (um por linha) |
| `-l` / `-L` | Só nomes de arquivos com / sem correspondência |
| `-r` / `-R` | Recursivo (`-R` segue links simbólicos) |
| `-w` | Palavra inteira |
| `-x` | Linha inteira |
| `-h` / `-H` | Oculta / mostra o nome do arquivo |
| `-A n` / `-B n` / `-C n` | n linhas depois / antes / ao redor |
| `-e padrão` | Vários padrões (`-e a -e b`) |
| `-f arq` | Padrões vindos de arquivo |
| `-q` | Silencioso (só código de saída) |
| `-s` | Suprime erros de arquivo inexistente |
| `-m N` | Para após N linhas |
| `--include=GLOB` | Filtra arquivos em busca recursiva |

Código de saída: **0** encontrou · **1** não encontrou · **2** erro.

```bash
grep -v '^#' arq                    # sem comentários
grep -v '^\s*$' arq                 # sem linhas vazias
grep -E '^(root|joao):' /etc/passwd
grep -E '[0-9]{3}-[0-9]{4}' arq
grep -c '^$' arq                    # conta linhas vazias
grep -rn 'erro' /var/log/ --include='*.log'
grep -E '(ab)\1' arq                # retrovisor
```
- Sempre coloque o padrão **entre aspas simples** para o shell não interpretar `*`, `$`, `|`.
- `sed` usa BRE por padrão (`-E`/`-r` para ERE); `\1` na substituição referencia grupos.

### Pegadinhas 103.7
- `grep -c` conta **linhas**, não ocorrências (`-o | wc -l` conta ocorrências).
- `egrep` = `grep -E`; `fgrep` = `grep -F`.
- No BRE, `a|b` é literal; use `a\|b` ou `grep -E`.
- `^` e `$` fora do início/fim do padrão podem ser literais em BRE.
- `*` no começo de uma BRE é literal.
- `-w` (palavra) × `-x` (linha inteira).
- `-l` lista arquivos; `-L` lista os que **não** casam.
- `.*` é guloso.

---

# 103.8 — Edição básica de arquivos com o vi

## Modos

| Modo | Como entrar | Função |
|---|---|---|
| **Comando** (normal) | `Esc` | Navegar, apagar, copiar, colar (modo inicial) |
| **Inserção** | `i a I A o O s S c...` | Digitar texto |
| **Última linha** (ex) | `:` (comandos), `/` `?` (busca) | Salvar, sair, substituir |
| **Visual** (vim) | `v` (caractere), `V` (linha), `Ctrl+V` (bloco) | Seleção |
| Substituição | `R` | Sobrescreve texto |

## Entrar em modo de inserção

| Tecla | Efeito |
|---|---|
| `i` / `a` | Antes / depois do cursor |
| `I` / `A` | No **início** / no **fim** da linha |
| `o` / `O` | Nova linha **abaixo** / **acima** |
| `s` / `S` | Substitui o caractere / a linha inteira |
| `cw`, `cc`, `C` | Altera palavra / linha / até o fim da linha |
| `r` | Troca **um** caractere (e volta ao modo comando) |
| `R` | Modo de substituição (sobrescrita) |

## Navegação

| Tecla | Movimento |
|---|---|
| `h j k l` | Esquerda, baixo, cima, direita (também setas) |
| `w` / `b` / `e` | Próxima palavra / palavra anterior / fim da palavra |
| `0` / `^` / `$` | Início da linha / primeiro não-branco / fim da linha |
| `gg` ou `1G` | Início do arquivo |
| `G` | Fim do arquivo |
| `nG` ou `:n` | Linha n |
| `H` `M` `L` | Topo / meio / base da tela |
| `Ctrl+F` / `Ctrl+B` | Página para frente / trás |
| `Ctrl+D` / `Ctrl+U` | Meia página para baixo / cima |
| `(` `)` / `{` `}` | Sentença / parágrafo anterior e próximo |
| `%` | Pula para o par de `( [ {` |
| `f<c>` / `t<c>` | Vai até (ou antes) do caractere c na linha; `;` repete |

## Editar: apagar, copiar, colar

| Tecla | Efeito |
|---|---|
| `x` / `X` | Apaga caractere sob / antes do cursor |
| `dd` | Apaga (recorta) a linha |
| `dw` | Apaga palavra |
| `d$` ou `D` | Apaga até o fim da linha |
| `d0` | Apaga até o início da linha |
| `dG` / `dgg` | Apaga até o fim / início do arquivo |
| `yy` | Copia a linha |
| `yw` / `y$` | Copia palavra / até o fim da linha |
| `p` / `P` | Cola **depois** (abaixo) / **antes** (acima) |
| `u` | Desfaz |
| `Ctrl+R` | Refaz |
| `U` | Desfaz alterações na linha |
| `.` | Repete a última alteração |
| `J` | Junta a linha seguinte |
| `~` | Inverte maiúscula/minúscula |

**Contadores** multiplicam: `3dd` (3 linhas), `5yy`, `2p`, `d3w`, `4x`, `10j`. Truques: `xp` troca dois caracteres; `ddp` troca duas linhas.

## Busca e substituição

| Comando | Efeito |
|---|---|
| `/texto` | Busca para frente |
| `?texto` | Busca para trás |
| `n` / `N` | Próxima / anterior (mesmo sentido / sentido oposto) |
| `*` | Busca a palavra sob o cursor |
| `:s/a/b/` | Substitui a 1ª ocorrência **na linha atual** |
| `:s/a/b/g` | Todas na linha atual |
| `:%s/a/b/g` | Todas no **arquivo** |
| `:%s/a/b/gc` | Com confirmação |
| `:5,10s/a/b/g` | Nas linhas 5 a 10 |
| `:g/padrão/d` | Apaga todas as linhas que casam |

As buscas aceitam expressões regulares (BRE).

## Salvar e sair (modo última linha)

| Comando | Efeito |
|---|---|
| `:w` | Salva |
| `:w arq` | Salva como |
| `:q` | Sai (falha se houver alterações não salvas) |
| `:q!` | Sai **descartando** alterações |
| `:wq` | Salva e sai |
| `:x` / `ZZ` | Salva (se houver mudança) e sai |
| `ZQ` | Sai sem salvar |
| `:w!` | Força gravação |
| `:e!` | Recarrega o arquivo descartando alterações |
| `:e arq` | Abre outro arquivo |
| `:r arq` | Insere o conteúdo de um arquivo |
| `:r !cmd` | Insere a saída de um comando |
| `:!cmd` | Executa comando do shell |
| `:set number` / `:set nonumber` | Mostra / oculta numeração de linhas |
| `:set ignorecase` | Busca sem diferenciar caixa |

## Linha de comando do vi
```bash
vi arq            vi +N arq          vi + arq (abre na última linha)
vi +/padrão arq   (abre na 1ª ocorrência)
vi -r arq         (recupera após queda; usa o arquivo .arq.swp)
view arq          (somente leitura; o mesmo que vi -R)
vimtutor          (tutorial interativo)
```
- A variável `EDITOR` (e `VISUAL`) define o editor padrão de programas como `crontab -e`, `visudo` (usa `EDITOR`/`VISUAL`), `git`.

## Outros editores (apenas reconhecimento)
- **vim** = vi melhorado (modo visual, realce de sintaxe, múltiplos undos); em muitas distros `vi` é um link/alias para `vim`.
- **nano**: simples; atalhos exibidos no rodapé: `Ctrl+O` grava, `Ctrl+X` sai, `Ctrl+K` recorta linha, `Ctrl+U` cola, `Ctrl+W` busca, `Ctrl+G` ajuda.
- **emacs**: `Ctrl+X Ctrl+S` salva, `Ctrl+X Ctrl+C` sai.

### Pegadinhas 103.8
- O vi **abre em modo comando**: digitar texto direto executa comandos.
- `Esc` sempre volta ao modo comando.
- `dd` **recorta** (a linha vai para o buffer e `p` cola); `yy` copia.
- `:q` falha com alterações; `:q!` descarta; `:x` e `ZZ` salvam e saem.
- `:s/a/b/` atua só na linha atual; `:%s/a/b/g` no arquivo todo.
- `p` cola **depois**, `P` **antes**.
- `a` (depois do cursor) × `A` (fim da linha); `i` × `I` (início da linha); `o` × `O`.

---

# Revisão rápida — comandos × função

| Preciso… | Comando |
|---|---|
| Ver só algumas colunas | `cut -d: -f1,3` |
| Ordenar e remover repetidas | `sort \| uniq` ou `sort -u` |
| Contar repetições | `sort \| uniq -c` |
| Trocar texto em fluxo | `sed 's/a/b/g'` |
| Trocar/apagar caracteres | `tr` |
| Dividir arquivo | `split -l N` |
| Numerar linhas | `nl` / `cat -n` |
| Achar arquivo por atributo (tempo, tamanho, dono) | `find` |
| Achar arquivo rápido por nome | `locate` (após `updatedb`) |
| Copiar bloco bruto / disco | `dd` |
| Empacotar | `tar` / `cpio` |
| Comprimir | `gzip` / `bzip2` / `xz` |
| Gravar saída e ver na tela | `tee` |
| Transformar entrada em argumentos | `xargs` |
| Rodar após fechar o terminal | `nohup` / `disown` / `screen` / `tmux` |
| Listar processos | `ps` / `pstree` / `top` |
| Achar PID por nome | `pgrep` / `pidof` |
| Matar | `kill` / `pkill` / `killall` |
| Mudar prioridade | `nice` (novo) / `renice` (em execução) |
| Buscar padrão em texto | `grep` |
| Editar | `vi` |

## Números para decorar
- FDs: **0** stdin · **1** stdout · **2** stderr
- Sinais: **1** HUP · **2** INT · **9** KILL · **15** TERM (padrão) · **18** CONT · **19** STOP · **20** TSTP
- Nice: **−20 a 19**, padrão **0**; `nice` sem `-n` = **+10**
- `split`: **1000 linhas** por arquivo · `head`/`tail`: **10 linhas** · `watch`: **2 s** · `top`: **3 s**
- Seções do man: **1** comandos · **2** syscalls · **3** biblioteca · **4** dispositivos · **5** arquivos de config · **6** jogos · **7** misc · **8** admin · **9** kernel