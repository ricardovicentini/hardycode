---
layout: post
title:  "ConfigureWait(false); Sabe o que é? Já usou alguma vez?"
date:   2021-04-25 14:20:00 -0300
categories: C#
---
Se as respostas para uma das perguntas do título desse artigo for "não", você deveria ler este artigo!

Resolvi escrever esse artigo depois de um questionamento no trabalho, se deveríamos usar ou não o <span style="color:red">ConfigureAwait(false)</span> em nossas aplicações.  
Os 2 principais questionamentos que sempre escuto são:
* mas afinal pra que server esse <span style="color:red">ConfigureAwait(false)</span>?
* ainda precisa disso no .NET Core?  

Bom, antes de tudo eu recomendo fortemente que vc leia meu [artigo anterior](/hardycode/c%23/2021/04/24/configure-await-faq-translation.html) que é a tradução de uma FAQ sobre ConfigureAwait que um desenvolvedor do time do .Net escreveu há um tempo atrás, mas que ainda vale a pena sua leitura integral.  

Ok, se você entrou no meu artigo anterior e levou um susto com o tamanho dele, eu não te julgo (rs). De fato tem muita coisa lá, por isso, nesse artigo eu vou tentar dar uma resumida nas questões mais fundamentais.

##### O que é o <span style="color:red">ConfigureAwait(false)</span>?  
Todo método awaitable, ou seja, que você pode chamá-lo de forma assíncrona usando o prefixo <span style="color:red">await</span> permite a possibilidade de ser chamado com <span style="color:red">ConfigureAwait(false/true)</span>, exemplo:  
```c#
using static System.Console;
using System.Threading.Tasks;
using MyProgram;

WriteLine("Antes da chamada do AguardarAsync");
await Program.AguardarAsync().ConfigureAwait(false);
WriteLine("Depois da chamada do AguardarAsync");

namespace MyProgram
{
  internal static class Program
    {
        internal static async Task AguardarAsync()
        {
            WriteLine("Antes do Task.Delay");
           
            await Task.Delay(1000).ConfigureAwait(false);
                
            WriteLine($"Depois do Task.Delay");
        }
    }    
}

```
##### O que faz <span style="color:red">ConfigureAwait(false)</span>? 
É uma forma simples de indicar que após a execução de um código assíncrono o Runtime do .Net não precisa se preocupar em rodar o resto do código na mesma thread em que o código assíncrono foi executado.  
Parece meio complicado mas veja no código anterior, na linha 6, estamos fazendo uma chamada para um método assíncrono, o <span style="color:red">AguardarAsync()</span> que está declarado na classe Program, na linha 13. Então o que o ConfigureAwait(false) da linha 6 está dizendo é que quando a chamada assíncrona terminar o Runtime não precisa se preocupar em executar o código da linha 7 na mesma thread em que o código, até esse ponto, foi executado.  

##### O que ganhamos usando <span style="color:red">ConfigureAwait(false)</span>? 
Ganhamos um pouco de performance, pois para garantir que o código rode na mesma thread, a thread é armazenada na heap do Runtime. 
* comparativo de performance: a esquerda com ConfigureAwait e a direita sem.  
![Comparativo da heap entre as duas abordagens](../../../../assets/Compartivo heap.jpg) 
É possível notar que a heap e a quantidade de alocação de objetos é ligeiramente menor usando o ConfigureAwait  

* Outro ganho é que conseguimos garantir que os consumidores dos métodos assíncronos não ficarão presos em "deadlock" devido a má utilização.  
No vídeo abaixo você pode ver mais detalhes sobre isso..  
[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/P8NxO0jHDzs/0.jpg)](https://www.youtube.com/watch?v=P8NxO0jHDzs)

##### Não precisa mais usar <span style="color:red">ConfigureAwait(false)</span> a partir do .Net Core?
Precisa sim, a utilização dele continua recomendada, muita gente critica sua necessidade e tentam criar soluções alternativas, que em geral não funcionam bem. Mas Sim!!! ConfigureAwait(false) continua sendo extremamente necessário, mesmo na versão ainda em preview do .Net 6  

#### Conclusão!

ConfigureAwait(false) é um dispositivo importantíssimo do .NET e merece nossa atenção, conhecer bem seu funcionamento pode ajudar muito a entender problemas de performance e deadlocks em aplicações de qualquer tipo!  

Dúvidas ou sugestões entre em contato, deixe seus comentários!