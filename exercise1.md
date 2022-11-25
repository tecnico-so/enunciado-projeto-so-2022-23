# Projeto base (ponto de partida)

O projeto base é o TecnicoFS (Técnico _File System_), um sistema de ficheiros simplificado em modo utilizador.
É implementado como uma biblioteca, que pode ser usada por qualquer processo cliente que pretenda ter uma instância privada de um sistema de ficheiros no qual pode manter os seus dados.

## Interface de programação

O TecnicoFS oferece uma interface de programação (API) inspirada na API de sistema de ficheiros POSIX.
No entanto, para simplificar o projeto, a API do TecnicoFS oferece apenas um subconjunto de funções com interface simplificada:

- `int tfs_open(char const *name, int flags);`
- `int tfs_close(int fhandle);`
- `ssize_t tfs_write(int fhandle, void const *buffer, size_t len);`
- `ssize_t tfs_read(int fhandle, void *buffer, size_t len);`

(Nota: o tipo de dados `ssize_t` é definido no _standard_ POSIX para representar tamanhos em _bytes_, podendo também ter o valor `-1` para representar erro.
É, por exemplo, o tipo do retorno das funções `read` e `write` da API de sistema de ficheiros POSIX.)

Além destas funções, existem as funções de inicialização e destruição do sistema de ficheiros, `tfs_init` e `tfs_destroy`.

O código fonte do TecnicoFS encontra-se disponível [neste repositório](https://github.com/tecnico-so/projeto-so-2022-23).
A descrição detalhada de cada função pode ser encontrada na documentação no código fonte do TecnicoFS.

## Estado do sistema de ficheiros

Tal como em FS tradicionais modernos, o conteúdo do FS encontra-se referenciado numa estrutura de dados principal chamada tabela de _i-nodes_, global ao FS.
Cada _i-node_ representa uma diretoria ou um ficheiro no TecnicoFS, que tem um identificador único chamado _i-number_.
O _i-number_ de uma diretoria/ficheiro corresponde ao índice do _i-node_ correspondente na tabela de _i-nodes_.
O _i-node_ consiste numa estrutura de dados que descreve os atributos da diretoria/ficheiro (aquilo que normalmente se chamam os seus _metadados_) e que referencia o conteúdo da diretoria/ficheiro (ou seja, os _dados_).

Além da tabela de _i-nodes_, existe uma região de dados, organizada em blocos de tamanho fixo.
Esta região mantém os dados de todos os ficheiros do FS, sendo esses dados referenciados a partir do _i-node_ de cada ficheiro (na tabela de _i-nodes_).
No caso de ficheiros normais, é na região de dados que é mantido o conteúdo do ficheiro (por exemplo, a sequência de caracteres que compõem um ficheiro de texto).
No caso de diretorias, a região de dados mantém a respetiva tabela, que representa o conjunto de ficheiros (ficheiros normais e subdiretorias) que existem nessa diretoria.

Para facilitar a alocação e libertação de _i-nodes_ e blocos, existe um vetor de alocação associado à tabela de _i-nodes_ e à região de dados, respetivamente.

Além das estruturas de dados mencionadas acima, que mantêm o estado durável do sistema de ficheiros, o TecnicoFS mantém uma tabela de ficheiros abertos.
Essencialmente, esta tabela conhece os ficheiros atualmente abertos pelo processo cliente do TecnicoFS e, para cada ficheiro aberto, indica onde está o cursor atual.
A tabela de ficheiros abertos é descartada quando o sistema é desligado ou termina abruptamente (ou seja, não é durável).

Nas aulas teóricas da 2ª semana, o código base será apresentado e discutido.
Recomendamos a todos os estudantes que frequentem essas aulas antes de começarem a desenvolver a solução.

## Simplificações

Além de uma API simplificada, o desenho e implementação do TecnicoFS adotam algumas simplificações fundamentais, que sumarizamos de seguida:

- Em vez de uma árvore de diretorias, o TecnicoFS tem apenas uma diretoria (a raiz "/"), dentro da qual podem existir ficheiros (e.g., "/f1", "/f2", etc.) mas não outras subdiretorias.
O texto entre aspas nos exemplos anteriores é chamado o **caminho de acesso** ao ficheiro.
- O conteúdo dos ficheiros e da diretoria raiz é limitado a um bloco (cuja dimensão é determinada em tempo de compilação e que é configurada para 1KB, por omissão).
Como consequência, o _i-node_ respetivo tem um campo simples que indica qual o índice desse bloco.
- Assume-se que existe um único processo cliente, que é o único que pode aceder ao sistema de ficheiros.
Consequentemente, existe apenas uma tabela de ficheiros abertos e não há permissões nem controlo de acesso.
- A implementação das funções assume que estas são chamadas por um cliente sequencial, ou seja, a implementação pode resultar em erros caso uma ou mais funções sejam chamadas concorrentemente por duas ou mais tarefas (_threads_) do processo cliente.
Por outras palavras, não é _thread-safe_.
- As estruturas de dados que, em teoria, deveriam ser duráveis, não são mantidas em memória secundária.
Quando o TecnicoFS é terminado, o conteúdo destas estruturas de dados é perdido.

# 1º Exercício

No 1º exercício pretende-se estender a versão base do TecnicoFS com as funcionalidades que são descritas de seguida.

## 1.1 Cópia de um sistema de ficheiros externo

Pretende-se que seja oferecida e implementada uma nova função:

- `int tfs_copy_from_external_fs(char const *source_path, char const *dest_path);`

Esta função copia o conteúdo de um ficheiro `source_path` existente no sistema de ficheiros do sistema operativo no qual o TecnicoFs corre, para o ficheiro _destPath_ guardado no TecnicoFS.
Por outras palavras, importa o conteúdo de um ficheiro do sistema de ficheiros nativo para dentro do TecnicoFS (limitado à dimensão de 1 bloco).

Devolve 0 em caso de sucesso, -1 em caso de erro.

Caso o ficheiro identificado por `dest_path` não exista, este deve ser criado.
Caso contrário, o conteúdo prévio do ficheiro já existente deve ser totalmente substituído pelo novo conteúdo.

Além de implementar esta funcionalidade no TecnicoFS, cada grupo deve desenvolver também pelo menos um programa de teste que demonstre e verifique a correção da solução proposta.

## 1.2 Nomes alternativos e Atalhos

Pretende-se permitir a criação e apagamento de **nomes alternativos** (_hard links_) e **atalhos** (_soft links_) para ficheiros.
Ambos os tipos de _links_ permitem criar ligações alternativas (i.e., pontos de acessos alternativos) para ficheiros pré-existentes no sistemas de ficheiros.
Contudo, estes dois tipos de _links_ têm semânticas diferentes, tal como ilustrado de seguida.

_Nomes alternativos._ Ao criar um _hard link_ "/path\_A/hard\_link" para um ficheiro "/path\_B/file\_name", é acrescentada uma entrada na lista do diretório "/path\_A" cujo nome é "hard\_link" e que aponta para o _i-node_ associado ao ficheiro "/path\_B/file\_name".
Nesse _i-node_ é também incrementado o contador de _hard links_.
Um ficheiro é efetivamente apagado só quando o contador de _hard links_ do relativo _i-node_ atinge o valor `0`.
Nota: este contador é inicializado com o valor `1` ao criar o ficheiro no sistema de ficheiro (o que corresponde, na prática, à criação de um _hard link_ para o ficheiro).

_Atalhos._ Ao criar um _soft link_ "/path\_A/soft\_link" para um ficheiro "/path\_B/file\_name", é acrescentada uma entrada na lista do diretório "/path\_A" cujo nome é "soft\_link" e cujo _conteúdo_ é "/path\_B/file\_name".
A diferença de um _hard link_ para um _soft link_ é que este último não aponta para o _i-node_ associado ao ficheiro "/path\_B/file\_name", mas armazena apenas uma referência para o nome do ficheiro para o qual esse _link_ aponta.
Consequentemente, caso o ficheiro "/path\_B/file\_name" venha a ser apagado após a criação de um _soft link_ para ele, esse ficheiro ("/path\_B/file\_name") é imediatamente apagado do sistema de ficheiros e, portanto, as aplicações que tentem usar esse _soft link_ irão incorrer em erros.

Para a criação de _hard/soft link_ deverão ser implementadas as seguintes funções:

`int tfs_link(char const *target_file, char const *source_file)` // cria um _hard link_

`int tfs_sym_link(char const *target_file, char const *source_file)` // cria um _soft link_

Pretende-se também implementar uma função que permite apagar ficheiros e _links_:

`int tfs_unlink(char const *target)`

Simplificações/observações:

- Podem ser criados _hard links_ apenas para ficheiros (ou diretorias), e nunca para _soft links_;
- Os _soft links_ guardam a referência para o _target_ no bloco de dados.

Cada grupo deverá também desenvolver pelo menos um programa de teste que demonstre e verifique a correção da solução proposta para cada função.

## 1.3 Suporte a chamadas concorrentes por cliente multitarefa

É desejável eliminar a restrição do programa cliente ser _single thread_, permitindo que seja constituído por múltiplas tarefas concorrentes (criadas por `pthread_create`). Pretende-se rever a implementação das funções do TecnicoFS para as tornar corretas quando executadas concorrentemente (i.e., _thread-safe_).

Para cumprir este requisito, devem ser usados trincos lógicos (`pthread_mutex`) ou trincos de leitura-escrita (`pthread_rwlock`), ficando a sua escolha ao critério dos estudantes.

A solução deve maximizar o paralelismo, recorrendo a sincronização fina.
Uma solução baseada em trincos que colocam em exclusão mútua a totalidade ou grande parte das estruturas de dados do TecnicoFS será fortemente penalizada na avaliação.

Caso sintam que a interface de funções que gerem o estado do TecnicoFS (definida em `fs/state.h`) não é a mais apropriada para resolver este requisito, podem adaptar essa interface para uma alternativa mais apropriada.
Recomendamos que, antes de começarem a implementar, discutam o desenho da vossa solução com o docente de laboratório.

Além da adaptação do TecnicoFS para o tornar _thread-safe,_ cada grupo deve **compor uma bateria de, pelo menos, três programas cliente paralelos**.

----

## Submissão e avaliação

A submissão é feita através do Fénix **até ao dia 19/Dezembro/2022 às 17h00**.

Os estudantes devem submeter um ficheiro no formato `zip` com o código fonte e o ficheiro `Makefile`.
O arquivo submetido não deve incluir outros ficheiros (tais como binários).
Além disso, o comando `make clean` deve limpar todos os ficheiros resultantes da compilação do projeto.

Recomendamos que os alunos se assegurem que o projeto compila/corre corretamente no [ambiente de referência](https://github.com/tecnico-so/lab_ambiente).
Ao avaliar os projetos submetidos, em caso de dúvida sobre o funcionamento do código submetido, os docentes usarão o ambiente de referência para fazer a validação final.
O uso de outros ambientes para o desenvolvimento/teste do projeto (e.g., macOS, Windows/WSL) é permitido, mas o corpo docente não dará apoio técnico a dúvidas relacionadas especificamente com esses ambientes.

A avaliação será feita de acordo com o método de avaliação descrito na página Fénix da cadeira.

_Bom trabalho!_
