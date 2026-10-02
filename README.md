# Diario.de.bordo

# Encontro 6 – Desafio Prático: O Dilema do Servidor em Nuvem

## Introdução

A startup CloudData utiliza um servidor de núcleo único para executar dois tipos de tarefas: processos interativos, relacionados às requisições dos usuários na interface web, e processos batch, responsáveis pela geração de relatórios financeiros.

O problema apresentado ocorre porque o servidor está utilizando o escalonamento FCFS (First-Come, First-Served). Nesse modelo, os processos são atendidos de acordo com a ordem em que chegam. Como algumas tarefas de geração de relatórios podem ser mais demoradas, os processos rápidos da interface podem precisar esperar, prejudicando o tempo de resposta do sistema.

A análise a seguir aborda o funcionamento das chamadas de sistema, os problemas relacionados ao FCFS e uma possível solução utilizando outro algoritmo de escalonamento.

---

## Parte A – Entendendo a Barreira do Sistema

Quando o processo de geração de relatório precisa acessar informações armazenadas no disco, ele não pode acessar diretamente o hardware. Para realizar essa operação, o processo solicita o serviço ao Sistema Operacional por meio de uma chamada de sistema, conhecida como System Call.

As chamadas de sistema funcionam como uma interface entre os programas executados pelo usuário e o núcleo do Sistema Operacional. No caso de uma leitura de arquivo, por exemplo, o processo pode solicitar uma operação de leitura e o Sistema Operacional fica responsável por realizar o acesso necessário ao dispositivo de armazenamento.

Durante esse processo ocorre uma mudança no nível de acesso da CPU. Inicialmente, o programa está executando em Modo Usuário, que possui permissões limitadas e não permite acesso direto a recursos críticos do computador.

Ao realizar a System Call, a execução passa temporariamente para o Modo Kernel. Nesse modo, o Sistema Operacional possui os privilégios necessários para acessar o hardware e executar a operação solicitada. Depois que o serviço é realizado, a execução retorna ao Modo Usuário e o programa continua normalmente.

Em uma operação de leitura do disco, o processo também pode ficar bloqueado enquanto espera o término da operação de entrada e saída. Nesse período, a CPU pode ser utilizada por outro processo que esteja pronto para executar.

De forma simplificada, o fluxo é:

**Processo em Modo Usuário → System Call → Modo Kernel → acesso ao recurso → retorno ao Modo Usuário.**

A documentação do Linux explica que as chamadas de sistema funcionam como serviços disponibilizados pelo kernel para aplicações e provocam a mudança da execução do espaço de usuário para o espaço do kernel. :chatgpt-content-reference{index="0"}

---

## Parte B – Diagnosticando o Escalonador

### Por que o FCFS está causando o congelamento da interface?

O FCFS trabalha seguindo a ordem de chegada dos processos. Isso significa que o primeiro processo que entra na fila de prontos é executado antes dos demais.

O problema ocorre quando um processo mais demorado é colocado antes de processos rápidos. Por exemplo, se a geração de um relatório começar antes de uma requisição da interface web, essa requisição pode precisar esperar até que o processo que está utilizando a CPU termine sua execução ou fique bloqueado.

Como os usuários da interface precisam de respostas rápidas, esse tempo de espera pode ser percebido como um congelamento do sistema.

Esse comportamento é especialmente ruim em um servidor de núcleo único, porque somente um processo pode utilizar a CPU em determinado instante.

Material acadêmico da Universidade Stanford apresenta o FCFS, também chamado de FIFO, como um modelo em que o primeiro processo da fila é executado até terminar ou bloquear. :chatgpt-content-reference{index="1"}

### O que significa dizer que o FCFS é não preemptivo?

Dizer que um algoritmo é não preemptivo significa que o Sistema Operacional não retira a CPU de um processo simplesmente porque outro processo mais importante ou mais rápido chegou.

Depois que o processo recebe a CPU, ele continua executando até terminar sua tarefa ou realizar alguma operação que provoque seu bloqueio, como uma espera por entrada e saída.

No cenário da CloudData, isso prejudica principalmente os processos interativos. Caso um processo mais pesado esteja utilizando a CPU, uma requisição rápida da interface não poderá interrompê-lo apenas por precisar de uma resposta mais rápida.

Isso aumenta o tempo de resposta da aplicação e explica a sensação de travamento relatada pelos usuários.

---

## Parte C – Propondo a Solução

### Algoritmo escolhido: Round-Robin

Entre SJF, SRTN e Round-Robin, uma alternativa adequada para melhorar a responsividade da interface é o Round-Robin.

O Round-Robin é um algoritmo preemptivo. Nesse método, cada processo recebe a CPU por um pequeno intervalo de tempo chamado de quantum ou time slice.

Quando o quantum termina e o processo ainda não concluiu sua execução, ele é interrompido e colocado novamente no final da fila. Em seguida, outro processo recebe a CPU.

O funcionamento pode ser representado da seguinte maneira:

**Processo A → Processo B → Processo C → Processo A → Processo B...**

Dessa maneira, um processo longo não permanece utilizando a CPU por um período muito grande enquanto outros processos aguardam.

No caso da CloudData, as requisições da interface web teriam oportunidades frequentes de execução, mesmo que existissem relatórios pesados sendo processados ao mesmo tempo. Isso ajudaria a reduzir o tempo de resposta percebido pelos usuários.

O Round-Robin também é adequado para ambientes interativos porque busca dividir o tempo de CPU entre os processos de maneira mais equilibrada. A principal decisão necessária é definir corretamente o tamanho do quantum. Um quantum muito grande pode fazer o algoritmo se comportar de forma semelhante ao FCFS, enquanto um quantum muito pequeno pode causar muitas trocas de contexto.

Fontes acadêmicas descrevem o Round-Robin como um algoritmo em que os processos utilizam a CPU durante um quantum e, ao final desse período, retornam para a fila caso ainda não tenham terminado. Esse modelo pode melhorar o tempo de resposta de aplicações interativas. :chatgpt-content-reference{index="2"}

O vídeo da Neso Academy também apresenta o Round-Robin como um algoritmo de escalonamento baseado em turnos de execução entre os processos. :chatgpt-content-reference{index="3"}

---

## Configuração do escalonamento em um servidor

Em um servidor real, a política de escalonamento faz parte do Sistema Operacional e é controlada pelo escalonador do kernel.

No Linux, por exemplo, existem diferentes políticas de escalonamento para processos e threads, incluindo políticas normais, processos batch e opções de tempo real.

Algumas dessas políticas podem ser definidas utilizando mecanismos do próprio Sistema Operacional. O Linux possui, por exemplo, as políticas `SCHED_FIFO`, `SCHED_RR` e `SCHED_BATCH`. Também existem comandos e chamadas de sistema capazes de alterar a prioridade e a política associada a determinados processos.

Por isso, o exemplo da atividade com FCFS deve ser entendido principalmente como um cenário didático para demonstrar os efeitos dos diferentes algoritmos. Em um servidor Linux atual, normalmente não se configura todo o sistema simplesmente como “FCFS”; diferentes políticas e prioridades podem ser utilizadas dependendo do tipo de processo. :chatgpt-content-reference{index="4"}

---

## Starvation e Prioridades

Caso a empresa escolha utilizar um algoritmo baseado em prioridades e defina os processos da interface web sempre com prioridade máxima, pode ocorrer um problema chamado Starvation, também conhecido como inanição.

Starvation acontece quando um processo permanece esperando por muito tempo porque outros processos com prioridade maior continuam sendo escolhidos para execução.

No caso apresentado, os relatórios poderiam ficar constantemente esperando pela CPU caso novas requisições da interface chegassem o tempo todo com prioridade superior.

Mesmo que o sistema continue funcionando, os processos batch poderiam demorar muito para executar ou, em uma situação extrema, permanecer aguardando indefinidamente. :chatgpt-content-reference{index="5"}

### Aging

Um mecanismo utilizado para evitar esse problema é chamado de Aging.

No Aging, a prioridade de um processo aumenta gradualmente conforme o tempo que ele permanece esperando.

Assim, mesmo que o relatório comece com uma prioridade mais baixa, sua prioridade será elevada aos poucos. Depois de determinado tempo, ele conseguirá disputar a CPU com os demais processos.

Dessa forma, o Sistema Operacional consegue manter boa resposta para as tarefas importantes sem impedir completamente a execução das tarefas de menor prioridade. :chatgpt-content-reference{index="6"}

---

## Pesquisa Multimídia

Para complementar o conteúdo estudado em aula, foram consultadas fontes em diferentes formatos.

### Vídeo

O vídeo **Scheduling Algorithms – Round Robin Scheduling**, produzido pela Neso Academy, explica como o Round-Robin organiza os processos em uma fila e distribui períodos de execução entre eles.

A principal contribuição da fonte para esta atividade foi compreender como o quantum permite interromper temporariamente um processo e liberar a CPU para os demais.

### Áudio

O episódio **Unlocking Efficiency: Essential Scheduling Criteria for Smarter Planning**, do podcast *Operating Systems Crashcasts*, aborda critérios utilizados para avaliar algoritmos de escalonamento, como utilização da CPU e tempo de resposta.

O conteúdo ajuda a relacionar o problema da CloudData com a necessidade de escolher um algoritmo que não considere somente a execução dos processos, mas também a experiência do usuário e o tempo necessário para que uma tarefa comece a responder. :chatgpt-content-reference{index="7"}

### Texto

Foram utilizadas documentações técnicas e materiais acadêmicos sobre chamadas de sistema e escalonamento de processos.

A documentação Linux Kernel Labs descreve a System Call como um mecanismo que permite que aplicações solicitem serviços ao kernel e explica a mudança de Modo Usuário para Modo Kernel. :chatgpt-content-reference{index="8"}

Também foram consultados materiais sobre FCFS, Round-Robin e escalonamento por prioridades, permitindo comparar o comportamento dos algoritmos no cenário apresentado.

---

## Instrumento Visual de Síntese
