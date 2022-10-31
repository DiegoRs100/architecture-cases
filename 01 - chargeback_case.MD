# Case - Chargeback

### Objetivo do negócio

Um banco tem a necessidade de processar as requisições de chargeback de seus clientes e para isso ele precisa diponibilizar uma interface para receber essas solicitações e enviá-las para a MASTERCARD, que irá então validar e processar os chargebacks.

**OBS:** Para esse case não iremos considerar front-end e bff.

### Restrições

* O envio de dados para a MASTERCARD deve ser realizado em arquivos batch através de um servidor FTP, onde cada linha representa uma solicitação de chargeback;
* A MASTERCARD aceita apenas 4 envios por dia;
* Arquivos que por algum motivo foram interrompidos no meio da transferência contam como um envio válido para MASTERCARD;
* O servidor FTP disponibilizado pelas MASTERCARD pode ser acessado apenas por um servidor FTP on-premise dentro do banco;
* A MASTERCARD processa os chargebacks de forma assincrona e disponibiliza os resultados dessa operação gerando um arquivo batch em um diretório específico no seu servidor FTP.

### Principais requisitos funcionais

* A mesma solicitação de chargeback não deve ser enviada mais de uma vez para MASTERCARD;
* A solicitação de chargeback deve ser enviada o mais rápido possível;
* A solução deve ser concebida inicialmente considerando um número médio de 4 milhões de chargebacks por dia.

### Principais atributos de qualidade

CONFORMIDADE > CONFIABILIDADE > RASTREABILIDADE > VELOCIDADE

# Solução técnica

### Diagrama técnico

![Diagram](https://raw.githubusercontent.com/DiegoRs100/architecture-cases/main/src/01%20-%20chargeback_case_diagram.bmp)

### Aplicações

**chargeback-api**

Uma api REST responsável por receber individualmente as solicitações de chargeback, verificar se são válidas e garantir que não estão duplicadas. Feito isso, a aplicação deverá gravar o registro em um banco de domínio.

**chargeback-scheduler**

O obejtivo desse worker é calcular por algoritimos qual o melhor momento do dia para realizar o envio de uma batch e qual o intervalo para o próximo envio, de forma a garantir sempre 4 envios por dia.
Uma vez que ele entenda que um batch deve ser enviado, é reponsável por buscar no banco de leitura todos os registros disponíveis para envio e gerar um arquivo batch no servidor FTP do banco contendo esses registros.
Por fim, ao criar um batch de chargeback, deve dispara eventos informando que os dados estão aguardando processamento.

**chrageback-batch-sender**

Esse worker fica instanciado dentro do servidor FTP e tem por objetivo monitorar uma pasta de rede onde os arquivos batch serão gerados.
Quando um novo arquivo é gerado, o worker realiza o envio do mesmo para o servidor FTP da MASTERCARD. Logo em seguida, dispara eventos informando o envio para a MASTERCARD;

**chrageback-batch-monitor**

Esse worker fica instanciado dentro do servidor FTP e é responsável por monitorar o diretório de rede onde os resultados das operações de chargeback são gerados.
Quando um batch com os resultados é gerado, o worker captura o resultado desse processamento e dispara eventos informando o resultado.

**chargeback-update-worker**

Esse worker é responsável por escutar os eventos de atualização do processamento em batch e atualizar o banco de domínio de chargeback.

## Conceitos utilizados 

* CQRS
* Micro-serviços
* Comunicação orientada a eventos
* Comunicação REST
* Cloud computing
* On-premisse computing

## ADRS

### Databases de leitura e escrita persistindo os dados de chargeback

**Contexto:**

Devido a quantidade de leitura e escrita do processo de chargeback o banco pode apresentar problemas de lock ou lentidão no momento de gerar os arquivos batch.

**Decisão:**

Foi definido que os dados de chargeback serão duplicados em bancos de leitura e escrita, considerando que o banco de leitura será um NOSQL para gerar melhor desempenho.
Além disso, a sincronização entre os bancos será feita via Kafka CDC.

**Consequências:**

Maior complexidade em ter de manter os dados em dois locais diferentes, além disso o CDC representa um novo ponto de falha, portanto sua configuração deve ser resiliente.

### Banco de dados de domínio sendo acessado por mais de uma aplicação

**Contexto:**

A aplicação 'chargeback-scheduler' necessita gravar dados no banco para realizar o controle de envios diários e logs de execução.

**Decisão:**

Decidiu-se por utilizar o mesmo banco de domínio de chargeback utilizado pela 'chargeback-api', isso se dá pois ambos os serviços estão fortemente acoplados e bancos separados não representam um ganho significativo de interoperabilidade frente aos custos de implementação.

**Consequências:**

Uma vez que ambas as aplicações estão acessando o mesmo banco de dados, caso o mesmo pare, o impacto é maior e os dois sistemas irão cair. Dessa forma, uma falha no banco gera sensivelmente mais impacto na operação como um todo.

### Gerar eventos de atualização de chargeback individualmente

**Contexto:**

Uma das óticas avaliada foi o envio de enventos informando mudança de status no processamento de chargeback, as duas opções consideradas foram enviar o evento com registros em lote ou enviar eventos indivisualmente, um evento por chargeback.

**Decisão:**

Optou-se pelo envio individual, pois isso garante maior rastreabilidade do processo registro a registro, além de trafegar mensagens menores na rede.
Avaliando bem a questão, concluiu-se também que o Kafka tem pode suficiente para dar conta desse processamento.

**Consequências:**

Essa decisão implica em um número maior de mensagens trafegando pela rede, aumentando o risco de pacotes não serem processados. Por conta disso o controle de deadletters tende a ser um tanto mais complexo de manter.

### Manter aplicações rodando on-premise no servidor FTP do banco

**Contexto:**

É necessário que os batchs gerados na pasta de rede do servidor sejam enviados para o FTP da MASTERCARD.

**Decisão:**

Optou-se por instanciar a aplicação 'chrageback-batch-sender' direto no servidor FTP do banco. Alguns itens foram considerados para essa tomada de decisão:
* Não é possivel acessar o FTP da MASTERCARD diretamente da AWS;
* A aplicação em questão não tem necessidade de ser escalada, pois executa apenas uma ação por vez, que não pode ser paralelizada.

**Consequências:**

Por estar dentro de um servidor on-premise toda a infra é mais complicada de se dar manutenção, além disso para possíveis reboots de aplicação é necessário ter acesso ao servidor.
Será necessário que a aplicação seja muito bem documentada para que o time que a sustentar não perca visibilidade da mesma.
