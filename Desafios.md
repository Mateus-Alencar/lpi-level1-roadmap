## Prompt  
Atue como um instrutor sênior de Linux e especialista na certificação LPIC-1, focado especificamente no Exame 101 (Versão 5.0). 

Eu quero que você gere um bloco de 5 desafios práticos de Linux para eu treinar. Siga estritamente as diretrizes abaixo:

1. Formato de Cenário Real: Em vez de pedir comandos diretamente, crie um cenário curto de problema ou necessidade real de um servidor (ex: "O sistema não monta a partição X no boot...", "Um processo zumbi está consumindo recursos...").
2. Nível de Dificuldade (Investigativo): Eu gosto de questões que tragam um desafio onde eu precise pesquisar, ler man pages ou correlacionar conceitos para responder. Evite comandos óbvios de nível básico/iniciante.
3. Escopo Restrito (Exame 101): Traga apenas tópicos cobertos na primeira prova da LPI nível 1. Sendo eles:
   - Tópico 101: Arquitetura do Sistema (Hardware, Boot, SysV/systemd).
   - Tópico 102: Instalação do Linux e Gerenciamento de Pacotes (Debian/RPM e bibliotecas compartilhadas).
   - Tópico 103: Comandos GNU e Unix (Linha de comando, filtros de texto, fluxos, redirecionamentos e processos).
   - Tópico 104: Discos, Sistemas de Arquivos Linux e Padrão FHS (Partições, integridade, montagem e permissões).
4. Sem Respostas de Início: Liste apenas os 5 enunciados/missões e pare de gerar texto. Não inclua os comandos de resposta ou explicações na primeira mensagem. Eu vou tentar resolver e te enviar as minhas respostas para você avaliar depois.

Agora dê foco no tópico 103.8 Edição básica de arquivos com o vi

Para começar, gere os primeiros 2 desafios investigativos focados no Exame 101.

## Desafio 1 - O servidor que "esqueceu" o disco no boot
Após uma manutenção, um servidor Debian passou a cair em modo de emergência a cada reinicialização. Ele funcionava bem antes de o administrador substituir o disco de dados por um novo. No journalctl aparece a falha de montagem de /dados, e o /etc/fstab referencia a partição por um identificador do disco antigo. Você precisa:
- Descobrir o identificador único do sistema de arquivos da nova partição sem usar fdisk -l; 
- Ajustar a entrada para que o boot não fique preso caso esse disco esteja ausente no futuro, e para que o sistema não tente executar binários nem interpretar dispositivos nesse ponto de montagem. 

Quais comandos e opções de montagem resolvem isso? Explique também o significado dos dois últimos campos da linha do fstab.  

## Desafio 2 – Biblioteca fantasma
#### (Resolvido)
Um binário proprietário instalado em /opt/app/bin/coletor falha ao iniciar com a mensagem `error while loading shared libraries: libmetrics.so.3: cannot open shared object file`. A biblioteca existe em /opt/app/lib. Você não pode alterar variáveis de ambiente globais em /etc/environment nem reinstalar o software. Descreva:  

(a) confirmar exatamente quais dependências estão faltando,   
(b) tornar a biblioteca visível ao carregador dinâmico de forma permanente para todo o sistema,   
(c) verificar se ela entrou no cache.  

Cite o arquivo/diretório de configuração envolvido e como o cache é atualizado. Existe uma alternativa temporária, válida só para uma execução? Qual?  

Resposta:  
a - `ldd /opt/app/lib` esse comando irá listar as dependências compartilhadas dessa biblioteca   
b - `echo "/opt/app/bin/coletor" > /etc/ld.so.conf.d/coletor.conf
    ldconfig`  # esse comando irá atualizar o cache com a nova biblioteca informada logo acima  
c - `ldconfig -p | grep libmetrics`  

Há outro possibilidade para executar testes com essa biblioteca antes de definir ela oficialmente.
```bash
LD_LIBRARY_PATH=/opt/app/lib /opt/app/bin/coletor
```

## Desafio 3 – Quem mandou esse pacote aqui?
#### (Pendente)
Em um servidor RHEL-like, o auditor encontrou o arquivo /usr/lib64/libcrypto-legacy.so.1 e quer saber a origem. Além disso, suspeita que o arquivo foi alterado depois da instalação. Sem usar interface gráfica nem buscar manualmente na internet, descreva como você (a) descobre a qual pacote o arquivo pertence, (b) verifica se o arquivo foi modificado em relação ao registrado no banco RPM, e o que cada caractere na saída dessa verificação (tamanho, hash, permissões, etc.) indica, e (c) lista quais scripts de instalação/remoção esse pacote executa. Qual seria o equivalente Debian para (a) e (b)?

## Desafio 4 – O pipeline que só funciona pela metade
#### (Resolvida)  
Um script de rotina roda `/coleta.sh > saida.log 2>&1 | sort | uniq -c > resumo.txt` e o administrador reclama que resumo.txt sempre fica vazio, enquanto saida.log contém tudo (stdout e stderr). Explique por que isso acontece, reescreva o comando para que stdout e stderr passem pelo pipeline e o resultado final vá para resumo.txt, e ainda grave, ao mesmo tempo, a saída bruta (antes do sort) em bruto.log sem interromper o fluxo. Depois, descreva o que muda se o script for iniciado com nohup e por que isso importa se a sessão SSH cair.

**Resposta**:
O `resumo.txt` fica vazio por causa do `> saida.log`, que redireciona o stdout do script para o arquivo antes de ele chegar ao pipe. O `2>&1` faz o stderr seguir o stdout, então os dois vão para `saida.log`. O `sort` não recebe nada pelo pipe e o `uniq -c` não tem o que contar.

Comando corrigido:
```bash
./coleta.sh 2>&1 | tee bruto.log | sort | uniq -c > resumo.txt
```
O `2>&1` vem antes do pipe para que stdout e stderr passem juntos pelo pipeline. O `tee` grava a saída bruta em `bruto.log` e deixa o fluxo continuar.

Com `nohup`, o processo ignora o SIGHUP, o sinal enviado quando a sessão SSH cai. Assim ele continua até o fim mesmo com a conexão fechada. O `nohup` não manda o processo para segundo plano: isso é feito pelo `&` no final. Além disso, ele só protege o primeiro comando do pipeline, então o pipeline inteiro deve ser envolvido em um shell:
```bash
nohup sh -c './coleta.sh 2>&1 | tee bruto.log | sort | uniq -c > resumo.txt' > /dev/null 2>&1 &
```

## Desafio 5 – Cadê o espaço? E as permissões?
#### (Pendente)
O df -h mostra /var com 100% de uso, mas du -sh /var soma bem menos do que o disco reporta. Em paralelo, df -i indica uso de inodes normal. Um processo de log foi reiniciado recentemente. Descreva o que provavelmente está acontecendo e como identificar, sem reiniciar o servidor, quais arquivos ainda estão abertos mas já foram removidos, além de como liberar o espaço sem matar o processo. Em seguida, considere que uma pasta compartilhada /srv/projetos deve permitir que membros do grupo dev criem arquivos, que novos arquivos herdem automaticamente o grupo dev, e que ninguém possa remover arquivos de outro usuário. Quais bits de permissão especiais e qual comando você usaria (em notação octal)? Justifique cada bit.

## Desafio 6 - Migração de caminhos  
Você administra um servidor de backup. O arquivo `/etc/backup/jobs.conf` tem cerca de 3.000 linhas e foi herdado de outro administrador. Os discos de dados foram migrados, e o diretório `/srv/data` agora se chama `/mnt/storage/data`.  
Há pegadinhas:  
- Só as linhas 40 a 120 devem ser alteradas. As demais ainda dependem do caminho antigo.  
- Algumas dessas linhas contêm `/srv/data` dentro de comentários que você quer preservar sem alteração.  
- Você quer confirmar cada substituição antes que ela aconteça.  
- O novo caminho contém barras, e você não quer escapar cada uma delas.  
Tudo isso deve ser feito dentro de uma única sessão do vi, sem sair do editor e sem usar sed externo. Em seguida, você percebe que errou numa das confirmações e precisa desfazer a última alteração.  
Missão: descreva a sequência de comandos do vi que cumpre todos esses requisitos. Explique também como lidar com a linha de comentário (dica: pense na sintaxe de endereçamento do ex e nas flags do :s). Por fim, diga como desfazer o erro e o que muda entre o comportamento do vi clássico e o do vim.

##### Resposta:
`:40,120s#/srv/data#/mnt/storage/data#gc`  
Após executar esse comando no mode de comando do vim, ele irá pedir a autorização para alterar cada linha individualmente, eu só cliquei em y (yes para troca) nas linha que não havia #

40,120s - pega a primeira ocorrência com inicio na linha 40 para alterada pela segunda em todas as linhas até a linha 120
g - global (leva em consideração todas as ocorrências daquela linha)
c - solicitação de confirmação antes de realizar a troca