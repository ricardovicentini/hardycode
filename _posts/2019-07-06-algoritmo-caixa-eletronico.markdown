---
layout: post
title:  "Algoritmo de saque em caixa eletrônico"
date:   2019-07-06 09:00:00 -0300
categories: Algoritmos
---

Precisamos construir um algoritmo para saque de dinheiro em caixas eletrônicos, então temos o seguinte desafio:  

> O caixa precisa ser inicializado com um número específico de notas de 100, 50 e 20  
> Deve se ser permitido saques apenas com valores que possibilitem entregrar a quantidade de notas corretamente 
> Só pode sacar se o caixa tiver dinheiro suficiente

##### Então vamos codar!?

<div class="row">
    <div class="col xl12">
      <div class="card blue-grey darken-1">
        <div class="card-content white-text">
          <span class="card-title">Não!!!</span>
          <p>Um erro muito comum que nós desenvolvedores cometemos é comumente conhecido como VLSF "Vontade Louca de Sair Fazendo".</p>
        </div>
      </div>
    </div>
  </div>
  

> _Tentar organizar primeiro o que queremos resolver e definir para quem iremos resolver, ajuda a organizar o código e evita retrabalho._

##### Vamos primeiro fazer o seguinte exercíco:  
> Para quem é essa solução?  
> como essa solução será consumida?  
> o que a solução deve fazer?  
> como deve fazer? (*finalmente código!* :heart_eyes_cat::heart_eyes_cat::heart_eyes_cat:)  

----


**Para quem?**
> Usúarios de caixa eletrônico

**Como será consumida?**
> O usuário irá fazer um saque de sua conta

**O que preciso fazer?**
> verificar se o caixa tem saldo para o saque solicitado  
> verificar se o valor solicitado pode ser retirado com as notas disponíveis no caixa  
> efetivar o saque separando as notas  
> atualizar o saldo do caixa  

**Então, orientado pelo que temos que fazer, podemos codar!** 

\#SOQUENAO :grinning:  

##### Antes, organize suas idéias

Utilizando o diagrama de sequência da UML (Unified Model Language), podemos ter uma visão mais clara das camadas, classes e métodos que pretendemos construir, conforme imagem abaixo:  
  

![image-title-here](../../../../assets/diagram de sequencia - sacar.jpg)
