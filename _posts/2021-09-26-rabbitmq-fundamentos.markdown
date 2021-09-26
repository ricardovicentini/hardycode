---
layout: post
title:  "RabbitMq Fundamentos - parte 1 de 3"
date:   2021-09-26 08:00:00 -0300
categories: Microservicos
---
RabbitMq é um "message broker" baseado no protocolo AMQP totalmente *opensource*, ou seja gratuíto e aberto, o que permite aprender como foi feito e ainda podendo ser melhorado por você!:bowtie:

Protocolo AMQP
==============
**Advanced Message Queuing Protocol**, ou Protocolo de Enfileiramento de Mensagens Avançado. *tradução livre*.  :sweat_smile:  
Estabelece regras e comportamentos entre a comunicação de **Publishers**{: style="color: red"}, **Message Brokers**{: style="color: red"} e **Consumers**{: style="color: red"}.  

>Mas o que é um Message Broker? :sweat_smile:  

Eu defino assim: "É um agente de gerenciamento de mensagens, ele garante o recebimento e a entrega de mensagens".  
As mensagens são enviadas por **Pubilishers** e processadas por **Consumers**.  

Podemos ter uma visão geral do Protocolo AMQP através da imagem abaixo.  
![RabbitMq hello world](../../../../assets/2021-09-23-rabbitmq-fundamentos/hello-world-example-routing.png)  
A imagem acima foi extraída da [página oficial do RabbitMq](https://www.rabbitmq.com/tutorials/amqp-concepts.html) e demonstra o funcionamento básico do protocolo AMQP 0-9-1.  
*O* *publisher*{: style="color: red"} *publica uma mensagen no* *broker*{: style="color: red"} *que através de seu* *binding*{: style="color: red"} *envia uma cópia da mensagem para sua fila (* *queue*{: style="color: red"}) *e por fim é consumida no* *consumer*{: style="color: red"}.  

De forma mais detalhada, podemos observar nessa imagem que há uma aplicação do tipo **PUBLISHER**{: style="color: red"}, responsável por criar as mensagens e publicá-las em um **EXCHANGE**{: style="color: red"}. *Exchanges* são parte da composição do **Message Broker**{: style="color: red"} que na imagem acima aparece destacado em um retângulo sombreado.  
Quando um *Exchange* recebe uma mensagem ele pode distribuir cópias dessa mensagem para alguma, algumas ou até mesmo para nenhuma fila, usando regras chamadas **BINDINGS**{: style="color: red"}. E então o *Broker* entrega as mensagens para os consumidores, que podem assinar a fila através de uma *sbscription*{: style="color: red"}, nesse caso os consumidores são considerados **passivos**, pois recebem as mensagens do *Broker* conforme elas vão sendo entregues para a fila. Ou os consumidores podem buscar as mensagens na fila, quando bem entenderem e portanto neste caso são considerados **ativos**.  

O Agente de Gerenciamento de Mensagens *(Message Broker)* é composto dos seguintes módulos:  

**Exchange**
: é o agente *intercambiador* das mensagens recebidas, podemos dizer que é algo similar a uma agência dos correios, *só que bem mais eficiente*. :joy::joy::joy:  

**Bindings**
: Composição de regras que definem o destino da mensagem  

**Queues**
: repositório de mensagens que serão consumidas em ordem, ou seja, uma *fila*.  

> Todos esses elementos apresentados até o momento:
: * Publishers
* Exchanges
* Bindings
* Queues
* Consumers  
> São denomidados ***Entidades do protocolo AMQP***.

Resiliência
===========
A comunicação via rede de dados pode falhar, não é confiável! O consumidor da mensagem também pode falhar, pode sofrer um **crash** durante o processamento da mensagem ou até mesmo perder a conexão com a fila. É por isso que no protocolo AMQP existe a definição de *message acknowledgement*{: style="color : red"} **(confirmação de recebimento ou processamento de mensagens)**.  
Quando uma mensagem é entregue para o *consumer*, o *consumer* notifica o *broker* após o processamento da mensagem, ou assim que pega a mensagem **(se não houver necessidade de garantir que a mensagem foi processada)**.  
O *Message Acknowledgement*{: style="color : red"}, portanto, pode ser considerado um dispositivo de segurança importante, que garante a entrega e/ou o processamento das mensagens com sucesso.  

Pode acontecer também de o *Exchange* não ser capaz de endereçar a mensagem a uma fila e neste caso o *Broker* devolve essas mensagens para o *Publisher*, ou até mesmo enviar as mensagens para uma fila conhecida como **"dead letter queue"**{: style="color : red"}.  

Flexibilidade
=============
O protocolo AMQP é conhecido por ser um protocolo programável, isso significa que a criação e comportamento de suas *Entidades* devem ser primariamente definidas na própria aplicação: *publishers/consumers*.  
Ou seja, é a própria aplicação que declara as entidades do protocolo AMQP necessárias.  

Exchanges e seus tipos
======================

No protocolo AMQP podemos escolher entre 4 tipos diferentes de *Exchange* a escolha de cada tipo depende muito do que se deseja fazer com as mensagens. Por isso abaixo explicarei um pouco como funciona cada um deles:  

* Direct
* Fanout
* Topic
* Header
{: style="margin-left : 20px"}

Todos os *Exchanges* possuem atributos, os atributos definem como podemos configurar o comportamento do *Exchange*. Abaixo temos os principais atributos:  

* Name (*nome do exchange*)
* Durability (*define se o exchange permanecerá existindo no caso do broker ser reiniciado*)
* Auto-delete (*o exchange será excluído assim que sua ultima fila for desplugada de seu binding*)
{: style="margin-left : 20px"}

##### Direct Exchange
![Direct Exchange](../../../../assets/2021-09-23-rabbitmq-fundamentos/exchange-direct.png)  

Esse tipo de *Exchange* envia a mensagem para sua fila através do *routing key*, a fila terá o mesmo nome do *routing key*. Podemos ter multiplas instâncias do consumer de cada fila, assim as mensagens podem ser consumidas simultaneamente escalando o consumer.  

Na imagem acima temos 4 consumers do lado direito, 2 consumers tem binding para a routing key "images.archive", cada consumer recebe somente a mensagem com o routing key ao qual tem o binding, quando o consumer possuí mais de uma instância, o *Broker* é responsável por balancear o envio das mensagens para eles.

##### Funout Exchange  
![Fanout Exchange](../../../../assets/2021-09-23-rabbitmq-fundamentos/exchange-fanout.png)  

Todas as filas desse tipo de exchange recebe uma cópia da mensagem, então os consumers conectados às filas irão processar a mesma mensagem. Esse tipo de Exchange não utiliza o routing key.
Na imagem acima podemos ver esse comportamento.  
Resumidamente, podemos ter 200 filas ligadas nesse Exchange e todas as 200 filas irão receber a mesma mensagem, porém cada fila terá o seu consumer específico, cada um realizando tarefas distintas, mas sempre com a mesma mensagem.

##### Topic Exchange  
![Topic Exchange](../../../../assets/2021-09-23-rabbitmq-fundamentos/exchange-topic.jpg)  
Similar ao Fanout, porém, nesse tipo a routing key é utilizado para enviar a mensagem a uma fila de acordo com o padrão descrito no binding.  
Na imagem acima é possível observar que o *Exchange* envia a mensagem para uma fila de acordo com o padrão da routing key, o caractere '\*' funciona como um caractere coringa, então podemos concluir que a *queue1* aceita qualquer routing key que contenha a a palavra '[qualquer texto].payroll.[qualquer texto]', na *queue2* a routing key deve ter '[qualquer texto].hr.recuitment' e a *queue3* aceitará todas as mensagens que contenham na routing key 3 palavras separadas por pontos.  
 Sendo assim a *queue3* receberá todas as mensagens que vão para a *queue1* e *queue2*  

> Importante notar que a composição da routing key e feita sempre separada por *pontos*.

##### Header Exchange  
![Header Exchange](../../../../assets/2021-09-23-rabbitmq-fundamentos/exchange-headers.jpg)  
Esse tipo de Exchange usa uma espécie de cabeçalho na mensagem para realizar o envio da mensagem para a fila. Portanto, *Header Exchange* é uma espécie melhorada de *Direct Exchange* pois ao invés de utilizar a *routing key* esse Exchange utiliza o Header da mensagem que possui mais atributos do que uma simples *routing key*.  
Então na imagem acima, podemos ver que o *Exchange* envia mensagens com cabeçalho onde o material é *wood (madeira)* para a *queue1* e mensagens com cabeçalho onde o material for *metal* irão para a *queue2*. E assim cada consumer pode aplicar regras de negócios distintas para cada mensagem, sabendo que no *consumer1* só chegará *wood/madeira* e no *consumer2* só chegará metal. 

Conclusão da primeira parte
===========================

RabbitMq é um poderoso *Message Broker* baseado no protocolo AMQP 0-9-1.  
Com ele podemos fazer a comunicação entre aplicações de microserviços diferentes de maneira escalável e resiliente.  
Nessa primeira parte trouxe os conceitos fundamentais do protocolo AMQP, na próxima parte pretendo aprofundar um pouco mais nas filas e seus atributos e por fim em uma 3ª parte fazer algumas demos.  

Espero que gostem do material, e indiquem para que quiser aprender sobre RabbitMq, se tiverem duvidas ou sugestões, ficarei muito feliz em responder! 🙂  

- - -  

