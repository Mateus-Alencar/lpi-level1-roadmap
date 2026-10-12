# LPIC-1 — Links simbólicos e links físicos

## 1. Conceitos fundamentais

No Linux, um link permite acessar um arquivo por outro nome ou caminho.

- **Hard link (link físico):** um nome adicional que referencia o mesmo inode do arquivo.
- **Symbolic link (link simbólico ou symlink):** um arquivo especial que armazena o caminho para outro arquivo ou diretório.

### O que é um inode?

O inode armazena metadados do arquivo, como permissões, proprietário, tamanho, datas e referências aos blocos de dados. O nome do arquivo fica em uma entrada de diretório associada ao inode.

Comandos úteis:

```bash
ls -i arquivo.txt   # Mostra o inode
stat arquivo.txt    # Mostra metadados e quantidade de links
```

## 2. Hard link — link físico

Criar um hard link:

```bash
ln arquivo.txt copia.txt
```

Características:

- O link e o arquivo original têm o **mesmo inode**.
- Ambos acessam os mesmos dados; não são cópias independentes.
- Alterações no conteúdo por um nome são visíveis pelo outro.
- Remover um dos nomes não apaga os dados enquanto houver outro hard link válido.
- Normalmente não pode atravessar sistemas de arquivos diferentes.
- Em geral, não se permite criar hard links para diretórios comuns.

Exemplo:

```bash
echo "Estudando Linux" > arquivo.txt
ln arquivo.txt backup.txt
echo "LPIC-1" > arquivo.txt
cat backup.txt   # Exibe: LPIC-1
```

Se `arquivo.txt` for removido, `backup.txt` continuará acessível.

## 3. Symbolic link — link simbólico

Criar um link simbólico:

```bash
ln -s arquivo.txt atalho.txt
```

Características:

- Tem **inode próprio**, diferente do inode do destino.
- Armazena o caminho do destino.
- Pode apontar para arquivos e diretórios.
- Pode apontar para destinos em outros sistemas de arquivos.
- Pode ser criado mesmo se o destino ainda não existir.
- Se o destino for removido ou o caminho deixar de resolver, o link pode ficar **quebrado** (*dangling link*).

Exemplo:

```bash
ln -s inexistente.txt atalho.txt
```

O link pode ser criado, embora o destino não exista.

Para ver o destino:

```bash
ls -l atalho.txt
readlink atalho.txt
```

Em `ls -l`, um symlink costuma aparecer com `l` no início e uma seta, por exemplo:

```text
lrwxrwxrwx ... atalho.txt -> arquivo.txt
```

## 4. Comparação rápida

| Característica | Hard link | Symbolic link |
|---|---|---|
| Inode | Mesmo do arquivo original | Inode próprio |
| Armazena o caminho do destino | Não | Sim |
| Pode apontar para diretórios | Normalmente não | Sim |
| Pode atravessar sistemas de arquivos | Não | Sim |
| Destino removido | Continua acessível se outro hard link existir | Pode ficar quebrado |
| Destino precisa existir ao criar | Sim | Não |
| Comando | `ln origem destino` | `ln -s origem destino` |

## 5. Remoção e comandos importantes

Exemplo:

```bash
ln dados.txt fisico.txt
ln -s dados.txt simbolico.txt
rm dados.txt
```

Depois de remover `dados.txt`:

- `fisico.txt` continua funcionando, pois referencia o mesmo inode e os dados ainda têm uma referência.
- `simbolico.txt` fica quebrado, pois aponta para o caminho `dados.txt`, que deixou de existir.

Comandos para memorizar:

| Comando | Função |
|---|---|
| `ln origem destino` | Cria um hard link |
| `ln -s origem destino` | Cria um link simbólico |
| `ls -i arquivo` | Mostra o inode |
| `ls -l` | Exibe detalhes e identifica links simbólicos |
| `stat arquivo` | Exibe metadados, inode e quantidade de links |
| `readlink link` | Exibe o caminho armazenado no symlink |
| `unlink nome` | Remove um nome de arquivo ou um link simbólico |

**Atenção:** `unlink` remove uma referência de nome. Isso não significa que todos os dados serão apagados: eles podem continuar existindo se houver outros hard links.

## 6. Pegadinhas comuns na LPIC-1

1. **Mesmo inode = hard link**, normalmente.
2. **Inode próprio + caminho armazenado = link simbólico**.
3. `ln` sem `-s` cria hard link; `ln -s` cria symlink.
4. Um symlink pode apontar para um destino inexistente; um hard link não.
5. Um hard link pode continuar funcionando após a remoção do nome original.
6. Um symlink pode ficar quebrado se o destino ou o caminho deixar de existir.
7. Hard links normalmente não atravessam sistemas de arquivos; symlinks podem.
8. `ls -l` mostra o destino de um symlink; `ls -i` ajuda a comparar inodes.

## 7. Prática rápida em uma VM

Execute em um diretório de testes:

```bash
echo "conteudo inicial" > original.txt
ln original.txt fisico.txt
ln -s original.txt simbolico.txt

ls -li original.txt fisico.txt simbolico.txt
stat original.txt
cat simbolico.txt

rm original.txt
cat fisico.txt
cat simbolico.txt
```

Observe que `original.txt` e `fisico.txt` compartilham o inode. O symlink tem outro inode e, após a remoção de `original.txt`, deixa de encontrar seu destino.

---

## Resumo para memorizar

- **Hard link:** outro nome para o mesmo inode.
- **Symlink:** arquivo especial que guarda um caminho.
- **Criar hard link:** `ln origem destino`.
- **Criar symlink:** `ln -s origem destino`.
- **Ver inode:** `ls -i`.
- **Ver metadados:** `stat`.
- **Ver destino do symlink:** `readlink`.
