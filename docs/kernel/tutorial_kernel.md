Tutorial de Kernel
==================

Autor: Douglas Schilling Landgraf

## Introdução


## Processos

Um primeiro conceito a ser introduzido antes de mais nada, é o conceito de PROCESSO. Todo e qualquer sistema operacional usa uma abstração conhecida pelo nome de processo, que pode ser considerada( entre outras coisas ) como uma instância de um programa( um programa em execução ).

Em sistemas operacionais tradicionais, um processo nada mais é do que uma lista de instruções a serem executadas pelo processador. Todo o processo em um sistema operacional recebe uma area de endereçamento que é um conjunto de endereços de memória ao qual o mesmo pode referenciar, sendo que nada além deste espaço diz respeito ao processo e todo acesso a partes externas a área de endereçamento do processo deve ser vetado.

É importante salientar que processos diferem de programas. Processos podem executar N programas sequencialmente enquanto um programa pode ser executado por M processos simultâneamente, portanto, programas são diferentes de processos. Em ambientes mono processados apenas um processo pode rodar por vez(fazer uso do processador), e o sistema operacional deve prover uma ferramenta para alternar o processador entre todos processos que estão ativos, passando a impressão de que vários programas rodam simultânemante. A esta ferramenta da-se o nome de scheduler ou escalonador.

Para prover esta impressão de que vários programas rodam simultaneamente em um único processador, o kernel do sistema operacional deve-se utilizar de um conceito conhecido como preempção, ou seja, dado certo tempo de execução do processo X, o escalonador é chamado para fazer uma troca de processo( oferecendo o processador a outro processo ativo ), assim distribuindo da forma mais justa possível o processador a todos os processos ativos no sistema.

Como um processo possui uma área restrita de endereçamento, quando o mesmo precisa de um recurso que reside em outra área que não seja a sua ele tem que fazer uma solicitação ao Kernel. Suponhamos um processo que necessita acessar o conteúdo de um certo arquivo, para tal tarefa ele faz uso de chamada ao Kernel solicitando tal informação. O Kernel então recebe esta solicitação e o fluxo passa a ocorrer agora em Kernel Space, ou seja, sem qualquer restrição quanto ao espaço de endereçamento. Uma vez que o Kernel tenha atendido a solicitação, ele a devolve ao processo e entao o fluxo volta ao seu contexto original (User Space).

Interessante levar em consideração que o fluxo que ocorre após a chamada, é um fluxo implementado no próprio kernel. A todas estas chamadas possíveis fornecidas pelo Kernel damos o nome de System Calls (syscalls), e cada syscall é composta por um conjunto de instruções aqui chamadas de kernel control path, portando, kernel control path é um conjunto de instruções necessárias para atender uma syscall.

Cada processo no Linux é representado por uma estrutura de controle chamada de process descriptor. Como apenas um processo pode rodar por vez em uma CPU, toda a vez que o processador troca de processo o estado atual do processo é armazenado nesta estrutura para que possa ser continuado quando sua vez de utilizar o processador chegar novamente. Quando um processo não está fazendo uso da CPU o mesmo reside em uma fila, sendo que o Linux gerencia filas diferentes para processos parados em diferentes estados( por exemplo os processos aguardando IO e os processos aguardando sua nova fatia de tempo de CPU ).

Como exemplo pratico podemos citar um processo que faz a requisicao de leitura para um determinado número de bytes de um arquivo no disco rígido(HD). Ele faz o uso de uma chamada de sistema, o Kernel interpreta o que o processo quer e dispara a requisicão para o driver(módulo) do disco que por sua vez dispara para o dispositivo físico. Uma vez disparado, este processo pode ter seu process descriptor atualizado e ficar aguardando em uma fila de processos aguardando IO, liberando assim a CPU para outro processo utilizar. Uma vez que o dispositivo físico( em nosso caso o HD ) terminar de processar a requisicão ele comunica o processador gerando uma interrupcão. Quando o processador recebe esta interrupcao ele pode parar o que esta fazendo e disparar um event handler que nada mais é do que um conjunto de instrucoes para tratar da interrupcao recebida.
Durante este intervalo o dispositivo( HD ) fica aguardando um ack por parte do Kernel.

Como ficou evidenciado acima, um processo pode ter sua fatia de tempo de CPU esgotada a qualquer instante, podendo ter seu fluxo ocorrendo em User Mode( executando o codigo do processo ) ou em Kernel Mode( no momento da formalizacao da requisicao ao disco físico ). Dada esta verdade podemos ter em um dado momento X, Y processos aguardando nas filas tendo seu fluxo parado em Kernel Mode. Da-se o nome de reentrant kernel quando o Kernel suporta estes Y processos com fluxos trancados em Kernel Mode. O Kernel Linux o é.

Analisando mais profundamente este comportamento nos deparamos com outro problema. E se o processo 0 teve sua execucao suspensa durante o atendimento de uma syscall( Kernel Mode ) e o processo 1 assumiu a cpu e alterou algum dado global do Kernel, quando voltarmos ao processo 0 o ambiente pode ter mudado e teremos um belo problema. Com o adendo do conceito de reentrant kernel também temos que introduzir técnicas para prevenir que um processo não interfira no trabalho de outro como visto acima.

Quando o retorno(resultado) de um processamento depende da ordem em que os processos são executados temos uma famosa race condition; Mais a frente tentaremos analisar técnicas para evitar estas famigeradas race conditions e os problemas ocasionados por técnicas mal aplicadas para preveni-las( aka dead-locks ); 

## Arquitetura

Em termos de arquitetura de Kernels, podemos dividi-los superficialmente em dois tipos:

Monolithic:
O Kernel é composto por um, e somente um, programa em que todas as syscalls, controles e afins são implementados.

MicroKernel:
Neste tipo de implementação o Kernel possui programas que o auxiliam trocando mensagens entre si. Aqui por exemplo o sub sistema de IO poderia estar em um processo separado, em que o nosso micro kernel conversaria quando necessitase de alguma operação de IO. Esta implementação é considerada por alguns como a mais elengante, porém envolve um algoritmo um pouco complexo para controlar esta troca de mensagens entre o micro kernel e seus 'auxiliares'.

Existe ums discussão entre qual dos dois modelos de implementação é o melhor e não cabe aqui uma discussão a cerca deste assunto. O Linux em si implementa um kernel monolítico com aspectos de micro kernel. Por exemplo, os módulos de certos dispositivos (drivers) podem ser plugados em nosso kernel monolítico. Isto traz um enorme ganho, pois permite que drivers sejam compilados, plugados ao kernel e testados sem a necessidade de uma reinicialização completa do sistema, a conversa entre o kernel e esse módulos plugáveis é feita através de uma bem definida API, sendo assim quem desenvolve drivers não necessita conhecer a fundo as entranhas do kernel, basta conhecer a bem documentada API.

## Filesystem

O design dos Sistemas Operacionais Unix Like( e aqui se encaixa o nosso Linux ) gira em torno de arquivos. Basicamente tudo em ums SO Unix Like é representado por um arquivo. O Kernel em si não conhece o conteúdo de cada arquivo, ele apenas lê e escreve neles conforme as aplicacões em um nível superior requerem.

Cada arquivo em um ambiente Unix é atrelado a uma estrutura de controle chamada inode. É nesta estrutura que ficam armazenas as informacões a cerca do arquivo. Informacões como: tipo do arquivo( regular, dispositivo, pipe ou dispositivo ), número de hard links associados ao arquivo, tamanho do arquivo, um identificador do dispositvo onde o arquivo está armazenado, um número identificador do inode, o dono e o grupo ao qual o arquivo pertence, direitos de acesso e informacões a cerca dos tempos da ultima alteracao e acesso ao arquivo.

Chamadas de sistema para trabalho com arquivos. Como falado anteriormente, quando um processo deseja acessar alguma informacão que esteja fora de seu espaco de enderecamento, o mesmo deve requisitar ao Kernel a informacao( e o faz atravez de uma chamada de sistema ). Abaixo estao listadas as chamadas de sistema que os processos podem executar para ter acesso ao conteúdo de arquivos: 

~~~
fd = fopen(path, flag, mode)
~~~

Todo processo só pode acessar arquivo previamente "abertos", portanto esta syscall informa ao Kernel que o processo necessita acessar o arquivo indicado pela variável path, qual o modo de abertura( read, write, append ) indicado pela variavel flag e um último parâmetro indicando( em caso de o arquivo não existir e necessitar se criado ) quais as permissões iniciais serão assumidas pelo mesmo. Quando um processo em User Land executa a chamada fopen ele recebe como retorno um descriptor para o arquivo. Nada mais é do que uma instância de um relacionamento com o arquivo, a partir daí o processo faz suas operacões utilizando-se deste descriptor, em nosso caso a variável fd.

Toda a vez que um arquivo é aberto via a fopen também é criado um ponteiro que indica a posicão atual no arquivo. Este ponteiro é iniciado na posicão 0 do arquivo( no início ) e toda a leitura ou escrita no arquivo é feita na posicão atual do ponteiro. O ponteiro pode ser movimentado através de uma outra chamada de sistema chamada de lseek()

~~~ 
lseek(fd, offset, whence);
~~~ 


Neste caso estamos desviando o ponteiro dentro de fd offset bytes a partir da posicão whence; Whence pode indicar o fim do arquivo, a posicão atual ou o inicio do arquivo.

Uma vez posicionados, podemos efetuar tanto uma leitura quanto uma escrita. Por exemplo para lermos count bytes a partir de nossa posicao atual do ponteiro faríamos:

~~~ 
read(fd, buf, count);
~~~ 


fd novamente indica o File Descriptor, buf é um ponteiro onde para uma posicão para onde desejamos que os dados lidos sejam armazenados e count o numero de bytes que queremos ler. Toda a vez que utilizamos um recurso somos responsaveis por libera-lo. Para tanto, após termos efetuados as operacoes desejadas em nosso arquivo devemos fechado:

~~~ 
fclose(fd);
~~~ 


Existem algumas chamadas de sistema para trocar o nome de arquivos ( rename(oldpath, newpath); ) e para delecão de arquivos( unlink(path) ); Lembrando que na estrutura armazenada no inode temos um contador de hard links, portanto um arquivo só pode ser removido efetivamente quando seu numero de referencias for igual a 0; 

## Hello World no Kernel 2.6.x

Desde o inicio dos tempos, o primeiro passo para aprender uma nova linguagem de programação é escrever um programa que exiba na tela a mensagem "Olá Mundo!" e aqui não será diferente. :) 


*Módulo olamundo.c*

~~~ 
#include <linux/init.h>
 #include <linux/module.h>

 static int __init olamundo_init(void) 
 {
       printk(KERN_DEBUG "Ola Mundo!\n");
       return 0;
 }
 static void __exit olamundo_exit(void)
 {
       printk(KERN_DEBUG "Tchau!\n");
 }

 MODULE_AUTHOR("Douglas Landgraf <dougsland@email.com>");
 MODULE_DESCRIPTION("Um modulo para exibir Ola Mundo!");
 MODULE_LICENSE("GPL");

 module_init(olamundo_init);
 module_exit(olamundo_exit);
~~~ 

*Makefile* 

ATENCAO: Não esqueça que Makefile exigem TAB ao invés de ESPAÇOS 

~~~ 
 # Diretório de módulos do Kernel 
 KDIR := /lib/modules/$(shell uname -r)/build

 # Regra default do Makefile
 default:
       $(MAKE) -C $(KDIR) M=$(PWD) modules
       @rm -rf *.mod* Module.symvers *.o

 # Objeto
 obj-m += olamundo.o

 # Limpando objetos e afins 
 clean:
 @rm -rf *~ *.o *.ko
~~~ 

*Testando*

~~~ 
 Criando o diretório para testes:
 shell$> mkdir meus-modulos

 Copiando o módulo olamundo e o Makefile para o diretório meus-modulos
 shell$> cp Makefile olamundo.c meus-modulos

 Entrando no diretório meus-modulos
 shell$> cd meus-modulos

 Compilando
 shell$> make

 Carregando o módulo na memória
 shell$> sudo insmod ./olamundo.ko 

 Exibindo a mensagem de Ola Mundo:
 shell$> dmesg

 Removendo o módulo da memória e exibindo a mensagem Tchau!
 shell$> rmmod olamundo 

 Exibindo a mensagem Tchau:
 shell$> dmesg
~~~ 



## Ferramentas para Controle de Versão


### Git




### Mercurial

Mercurial é um sistema para gerenciamento de versões distribuído utilizado por alguns subsistemas do kernel e projetos opensource.

Todos os comandos começam com hg, uma referência ao elemento químico mercúrio

Autor: Matt Mackall
Página: http://www.selenic.com/mercurial/
Licença: GPLv2
Plataformas suportadas: Linux, *BSD, Mac OS, Solaris, Windows 

#### Instalação

##### Debian e Ubuntu

~~~ 
 shell$> sudo apt-get install mercurial
~~~ 

##### Fedora e Red Hat

~~~ 
 shell$> sudo yum install mercurial
~~~ 

#### Comandos

##### Clone

Cria uma cópia de repositório em um diretório. Exemplo:

~~~ 
 shell$> hg clone http://linuxtv.org/hg/v4l-dvb
 shell$> ls v4l-dvb
~~~ 

##### Commit

Envia as alterações para o repositório local

 Exemplo:

~~~ 
 shell$> vi arquivo.c (algumas alterações ocorreram neste momento)
 shell$> hg commit [arquivo]
~~~ 


##### Status

Exibe o status dos arquivos do repositório local 

Exemplo:

~~~ 
 shell$> hg status [arquivo]
 M arquivo
~~~ 

 Os códigos de status são:

 M = Arquivo alterado (modified)
 A = Adicionado
 C = Sem alterações (clean) (Quando usa-se o parâmetro -c)
 R = Arquivo removido
 ! = Arquivo removido mas continua no repositório
 ? = Arquivo não desconhecido
 I = Arquivo ignorado
 Sem retorno  = Não houveram alterações no arquivo


##### Add

Sinaliza os arquivos que serão adicionados ao repositório local

 Exemplo:

~~~ 
 shell$> hg add arquivo
 shell$> hg commit (envia as alterações para o repositório local)
~~~ 



##### Remove

Sinaliza os arquivos que serão removidos do repositório local

 Exemplo:

~~~ 
 shell$> hg remove arquivo
 shell$> hg commit (envia as alterações para o repositório local)
~~~ 


##### Diff

Exibe as alterações de um determinado arquivo

 Exemplo:

~~~ 
 shell$> hg diff arquivo
~~~ 

Opções:

~~~ 
 -r = revisão 
 shell$> hg diff -r revisão arquivo 
~~~ 

##### ExtDiff


A extensão extdiff permite especificar a ferramenta que sera utilizada para visualizar as alterações feitas em um determinado programa.

Por padrão, a extensão extdiff utiliza a ferramenta diff para comparar os arquivos e validar se houveram alterações.
Porém, é possível facilmente alterar a ferramenta de comparação de código, por exemplo de diff para kdiff3.

 1. Adicione a extensão ao hgrc padrão ou ao .hgrc do local 

~~~ 
  Exemplo 1:
  shell$> vi /etc/mercurial/hgrc

  Exemplo 2:
  shell$> vi ~/.hgrc
~~~ 

Adicione as linhas:

~~~ 
  [extensions]
  extdiff =
~~~ 

Neste momento será possível especificar a ferramenta para diffs, exemplo passando o programa kdiff3 como parâmetro: 

  Exemplo:

~~~ 
  shell$> hg extdiff -p /usr/bin/kdiff3
~~~ 

Opções:

 -p = Ferramenta (program) por padrão é utilizado a ferramenta diff
 -o = parâmetros (options) por padrão é passado ao extdiff as opções -Npru


##### Alias

Ao invés de digitar hg extdiff -p /usr/bin/kdiff3 podemos criar um alias para digitar apenas: 

~~~ 
  shell$> hg kdiff3
~~~ 

Adicione no seu arquivo `~/.hgrc` local ou no hgrc geral `/etc/mercurial/hgrc`

~~~ 
 [extdiff]
 cmd.kdiff3 = kdiff3
 # Caso queria passar opções, remova o comentário e adicione abaixo
 # opts.kdiff3 = 
~~~ 

Agora já é possível utilizar o novo comando:

~~~ 
 shell$> hg kdiff3 
~~~ 



## Depuração


### SystemTap

O objetivo da ferramenta SystemTap é prover uma simples interface para coleta e visualização das informações sobre o Kernel Linux em execução. Através dela é possível identificar problemas de performance e funcionais.

SystemTap foi desenvolvido para eliminar a necessidade do desenvolvedor ter que recompilar, instalar e reiniciar o sistema toda vez que for coletar alguma informação sobre o Kernel. SystemTap prove uma simples interface em linha de comando.


#### Pré-requisitos


Para que o SystemTap funcione no seu sistema, será necessário recompilar o seu Kernel com símbolos de depuração OU instalar o pacote de depuração do kernel através da sua distribuição.

  1. Para habilitar os símbolos de depuração no kernel selecione:

~~~
      Kernel  hacking:
          [*] Kernel debugging 
          [*] Compile the kernel with debug info
~~~
 
  2. Instalando o pacote de debug do Kernel (Ubuntu)

~~~
       shell$> sudo apt-get install linux-image-debug-$(uname -r)
~~~
 
#### Instalação

##### Fedora e Red Hat 

~~~
shell$> sudo yum install systemtap
~~~

##### Debian e Ubuntu

~~~
    shell$> sudo apt-get install systemtap
~~~

##### Testando a Instalação

~~~
       shell$> stap -V
       SystemTap translator/driver (version 0.5.12 built 2007-02-02)
       (Using Red Hat elfutils 0.123 libraries.)
       Copyright (C) 2005-2006 Red Hat, Inc. and others
       This is free software; see the source for copying conditions.
~~~




#### Como utilizar ?

~~~
       shell$> cat syscall.open

       probe syscall.open
       {
           printf ("%s(%d) open (%s)\n", execname(), pid(), argstr)     
       } 

       shell$> sudo stap syscall.opn
~~~



#### Problemas comuns

##### Erro

~~~
       shell$> sudo stap strace-open.stp
       semantic error: libdwfl failure (dwfl_linux_kernel_report_offline): No such file or directory while
       resolving probe point kernel.function("sys_open")?
       semantic error: cannot find kernel debuginfo while resolving probe point kernel.function("sys32_open")?
       semantic error: no match for probe point while resolving probe point syscall.open
       Pass 2: analysis failed.  Try again with -v 
~~~

##### Indentificando o Problema

Execute o comando strace para localizar o problema:

~~~
       shell$> strace sudo stap strace-open.stp

      (Outputs do strace)
       .....

       Facilmente encontramos nosso problema, esta faltando em nosso sistema o arquivo vmlinux.debug

       open("/usr/lib/debug/lib/modules/2.6.X/vmlinux.debug", O_RDONLY|O_LARGEFILE) = -1 ENOENT 
       (No such file   or directory)
       write(2, "semantic error: libdwfl failure "..., 151semantic error: libdwfl failure (dwfl
~~~

##### Corrigindo o Problema

~~~
       shell$> cd /lib/modules/2.6.X
       shell$> ln -sf /boot/vmlinux-dbg-2.6.X vmlinux.debug
~~~

#### Referências

 - [SystemTap Tutorial por sourceware]()
 - [Instrumenting the Linux Kernel with SystemTap]()


### Kernel Log Time

O Kernel envia diversas mensagens ao seu arquivos de logs. Essas mensagens podem ser verificadas visualizando o arquivo de logs do sistema, normalmente /var/log/messages, ou executando o comando dmesg.

Em muitas ocasiões, será necessário visualizar com precisão em qual momento a mensagem foi enviado para o arquivo de log do sistema. Você pode configurar a opção Kernel Log Timestamps e cada mensagem terá um timestamp.

Para habilitar esta opção, no momento de configuração para uma nova compilação do seu Kernel, altere: 


~~~
       Kernel hacking:
          [*] Show timing information on printks
~~~

Após a compilação, efetue a validação de sua nova configuração: 

~~~
       shell> tail -f /var/log/messages
~~~

### Magic SysRq Keys

A opção Magic SysRq Keys é uma combinação de teclas, se a opção CONFIG_MAGIC_SYSRQ foi habilitada no momento de compilação, permite ao usuário executar diversos comandos de baixo nível utilizando a tecla SysRq.

Para habilitar esta opção, no momento de configuração para uma nova compilação do seu Kernel, altere: 

~~~
       Kernel  hacking:
           [*] Magic SysRq key
~~~

Sequência: Alt + SysRq + [OPCAO]

Tabela de comandos: [Magic SysRq Key]

Documentação disponível em: `/usr/src/linux/Documentation/sysrq.txt`


### Debug Filesystem

Um filesystem baseado em RAM e pode ser utilizado como output (saida) para várias mensagens de debug.

Para habilitar esta opção, no momento de configuração para uma nova compilação do seu Kernel, altere: 

~~~
      Kernel  hacking:
           [*] Debug filesystem
~~~

Após você habilitar esta opção e reiniciar sua máquina com o novo kernel, será criado o diretório `/sys/kernel/debug` com o intuito de ser um ponto de montagem para o filesystem debugfs

Manualmente: 

~~~
        $shell> mount -t debugfs none /sys/kernel/debug
~~~

Para montar automaticamente, adicione a seguinte linha no arquivo `/etc/fstab`: 

~~~
        debugfs /sys/kernel/debug debugfs 0 0
~~~

Quando o filesystem estiver montado, um grande número de diferentes diretórios e arquivos irão aparecer no diretório /sys/kernel/debug/. Todos os arquivos/diretórios são criados virtualmente/dinamicamente pelo kernel, assim como os arquivos/diretórios do procfs ou sysfs. Esses arquivos/diretórios podem ser usados para auxiliar na depuração de subsistemas ou simplemente verificar o que esta ocorrendo com o sistema.


### Outras opções para depuração


Existem diversas opções para depuração disponíveis no kernel, elas estão disponíveis na opção Kernel hacking.

Algumas opções:

~~~
      Kernel  hacking: 
           [*] Show timing information on printks
           [ ] Enable __must_check logic                                                    
           [*] Magic SysRq key                                                                  
           [*] Enable unused/obsolete exported symbols                                                 
           [*] Debug Filesystem                                             
           [ ] Run 'make headers_check' when building vmlinux
           [*] Kernel debugging                                                                
           [ ]   Debug shared IRQ handlers
           [*]   Detect Soft Lockups                                                         
           [*]   Collect scheduler debugging info
                  ......
~~~

Observação: Em caso de perda de performance, desative algumas opções de depuração. 


## UML - User Mode Linux

O User-Mode Linux é uma alternativa para uso de máquinas virtuais no Linux. 

### Download

  - [Kernel 2.6.22-rc2] (Arquitetura um)
  - [FedoraCore5-x86] RootFS


### Compilando Kernel

#### Makes

~~~
 maquina-real> mkdir /rootfs
 maquina-real> mkdir -p /tmp/modulesKernel

 maquina-real> mount -o loop FedoraCore5-x86-root_fs /rootfs

 maquina-real> cd /usr/src/linux
 maquina-real> export ARCH=um
 maquina-real> make menuconfig
 maquina-real> make modules
 maquina-real> make modules_install INSTALL_MOD_PATH=/tmp/modulosKernel
 maquina-real> cp -Rf /tmp/modulosKernel/lib/modules/2.6.X.X/ /rootfs/lib/modules/

 maquina-real> umount /rootfs
~~~


#### Opções

~~~
 Hostfs = UML-specific options -> Host filesystem
~~~


### Carregando

~~~
 ./vmlinux ubda=FedoraCore5-x86-root_fs mem=128M eth0=tuntap,,,192.168.1.254

 vmlinux = Imagem do Kernel (Arquitetura UM)
 ubda = Root FS
 mem = Memória a ser utilizada
 eth0 = Configuração de rede
~~~


### Configurando Rede

#### Fedora Core 5

  1. Adicione sua configuração de rede no arquivo ifcfg-eth0. Ex.:

~~~
uml-host> vi /etc/sysconfig/network-scripts/ifcfg-eth0

 DEVICE=eth0               # Dispositivo de Rede
 ONBOOT=yes                # Configuração no boot
 TYPE=Ethernet             # Tipo de dispositivo 
 IPADDR=192.168.1.253      # Adicione um IP válido da sua rede (que não esteja em uso)
 GATEWAY=192.168.1.1       # Gateway (máquina que irá prover a internet)
 NETMASK=255.255.255.0     # Máscara de rede

~~~
 
 2. Adicione seu nameserver no arquivo resolv.conf 

~~~
 maquina-REAL> cat /etc/resolv.conf

 search poa.virtua.com.br
 nameserver 201.21.192.103
 nameserver 201.21.192.105
~~~

  - Adicione sua configuração na máquina virtual UML

~~~
 UML-host> vi /etc/resolv.conf
~~~

  - Reinicie sua rede

~~~
 UML-host> /etc/init.d/network restart
~~~


### Instalando ferramentas básicas

~~~
 UML-host> yum install gcc make bzip2 ncurses-devel.i386 strace -y
~~~

## Autores

Douglas Landgraf <dougsland [*AT*] gmail [*DOT*] [*COM*]>
Ricardo José Maraschini <ricardomaraschini [*AT*] gmail [*DOT*] [*COM*]>


## Referências

  - Understanding Linux Kernel 3rd Edition Daniel P. Bovet & Marco Cesati
  - Understanding Linux Kernel 2nd Edition Daniel P. Bovet & Marco Cesati
  - Sistemas Operacionais Andrew S. Tanenbaum & Albert S. Woodhull





