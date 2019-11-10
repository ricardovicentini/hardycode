---
layout: post
title:  "Reactive Programming no dotnet core 3.0"
date:   2019-11-10 13:03:00 -0300
categories: Patterns
---
Já ouviu falar de Reactive Programming? ou Programação Reativa (tradução livre)  
Basicamente, Progração Reativa consiste em escrever codigos que respondem a eventos. Até ai nada novo, o pessoal do GOF (Gang of Four) inclusive já documentou um pattern para isso conhecido como  Observer e Observable.
E muitos frameworks e linguagens seguem esses padrões, por exemplo:
 * Aplicações desktop que respondem a eventos do sistema operacional, tais como: movimentação do mouse, pressionar teclas no teclado e etc

 Apesar do padrão ser bom, tem alguns problemas, principalmente quando precisamos tratar as informações com mais detalhes entre os eventos, por isso, foi criado no .net Fremework algumas extensões que melhoraram e facilitam a escrita de aplicações orientadas a eventos.  
 Conheça então o:
 **System.Reactive**

 > um pacote de extensões multiplataforma que tem uma abordagem de escrita funcional, com o objetivo de agregar o pattern Observable com LinQ e Schedulers.  
 Mais detalhes podem ser encontrados em http://reactivex.io/

Mas chega de história, preparei um exmplo bem legal...
##### Um jogo de Futebol!!!
> Antes que você se empolge d+, calma! Não tem nada gráfico!
Mas temos um juiz, dois times e algumas regras bem divertidas

Faça o fork no github: https://github.com/ricardovicentini/Jogo_fotebol_reativo

Funciona assim, para cada comando dado por um time ou pelo juiz, os participantes reagem de acordo com o comando (evento)
 
 Frontend do Código do jogo está em uma console aplication bem simples

    using Microsoft.Extensions.DependencyInjection;
    class Program
    {
        static void Main(string[] args)
        {
            

            var serviceProvider = new ServiceCollection()
                .AddScoped(r => new Regras(limiteFaltas: 3,limieteGols: 1, limiteTempo: 60))
                .AddSingleton<CentralEventos>()
                .AddScoped<Juiz>()
                .BuildServiceProvider();

            var central = serviceProvider.GetService<CentralEventos>();
            var juiz = serviceProvider.GetService<Juiz>();
            var time1 = new Time(central, "Palmeiras");
            var time2 = new Time(central, "Corinthians");

            try
            {
                juiz.IniciarPartida();  
                time1.MarcarGol();  
                time2.CometerFalta();  
                time1.CometerFalta();  
                time2.MarcarGol();  
                time1.CometerFalta();  
                time2.CometerFalta();  
                time2.CometerFalta();  
                time1.MarcarGol();  
            } 
            catch (System.ObjectDisposedException)
            {
                Console.WriteLine("O jogo já acabou");
            }
            

            Console.ReadKey();
        }
    }


Da linha 8 a linha 12 estou mapeando as instancias das classes que serão utilizadas via injeção de dependecia utilizando o container padrão do donet core, para utilizar esse container referencie no nuget:   
**Microsoft.Extensions.DependencyInjection;**

Veja na linha 9 que estou passando as regras do jogo: 
* limite de faltas: 3
* limite de gols: 1
* limite de tempo: 60 segundos  

Essas regras ditam o comportamento do juiz, pois casos um dos limites seja atendido o juiz encerra a partida

Sendo assim quando rodamos o código assim que executa a lina 21

    juiz.IniciarPartida();

No console tempos a saida

> Palmeiras: Vamos lá pessoal, pra cima deles!  
> Corinthians: Vamos lá pessoal, pra cima deles!

Pois assim que a partida inicia as classes Time recebem uma notificação de que  a partida começou e reagem a este evento  

E se deixar todo o código correr teremos então as respostas para cada evento de todos os participantes: times e juiz, conforme abaixo:

> Palmeiras: Vamos lá pessoal, pra cima deles!  
Corinthians: Vamos lá pessoal, pra cima deles!  
Juiz: Gol para o time Palmeiras  
Palmeiras: Muito bem pessoal!!!  
Corinthians: Ânimo pessoal!!! Vamos marcar mais forte  
Juiz: Fim de partida o time Palmeiras atingiu o nr. máximo de gols  
Palmeiras: Viva o Palmeiras!  
Corinthians: Na próxima Palmeiras vocês vão ver!  
O jogo já acabou  


Veja que quando o time marca gol o juiz reage indicando qual time marcou, o time adversário reage pedindo para o pessoal não desanimar e o time que marcou comemora o gol, perceba que os comandos de inicio e de gol partiram do Frontend mas é no Backend que todas as reações acontecem, inclusive, como a regra do jogo é terminar a partir no primeiro gol marcado na sequência o juiz termina a partida e novamente os times reagem ao término da partida, o vencedor comemora, o perdedor expressa ressentimento e os demais comandos do front passam a ser ignorados pois o jogo foi encerrado. 

Então agora vamos conhecer o que acontece no Backend dessa aplicação