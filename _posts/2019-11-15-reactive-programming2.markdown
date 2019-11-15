---
layout: post
title:  "Reactive Programming no dotnet core 3.0 (parte2)"
date:   2019-11-10 13:03:00 -0300
categories: Patterns
---
No artigo anterior demonstrei uma visão geral de como consumir Observables com System.Reactive.  
Agora vamos dar uma olhada como são essas classes:  
> Primeiro defini os eventos:  

```c#
public interface IEvento { } 
public class ApitoInicio : IEvento { }
public class ApitoFinal : IEvento { }  
```
A interface IEvento serve para tipar todas as demais classes que implementam essa interface, ou seja, toda classe que implementa IEvento  é um evento. 
Os eventos ApitoInico e ApitoFinal são simples não precisam trafegar nenhum tipo de informação, podem ser denominados como evento "oco". 

Abaixo temos eventos mais complexos
```c#
public class ApitoGol : IEvento 
{
    public ApitoGol(Time timeEmCampo)
    {
        TimeEmCampo = timeEmCampo;
    }

    public Time TimeEmCampo { get; set; }
}

public class ApitoFalta : IEvento
{
    public ApitoFalta(Time timeEmCampo)
    {
        TimeEmCampo = timeEmCampo;
    }

    public Time TimeEmCampo { get; set; }
    
}
public class GolMarcado : IEvento
{
    public GolMarcado(Time timeEmCampo, int golsMarcados)
    {
        TimeEmCampo = timeEmCampo;
        GolsMarcados = golsMarcados;
    }

    public Time TimeEmCampo { get; set; }
    public int GolsMarcados { get; set; }
}

public class FaltaCometida : IEvento
{
    public FaltaCometida(Time timeEmCampo, int faltas)
    {
        TimeEmCampo = timeEmCampo;
        Faltas = faltas;
    }

    public Time TimeEmCampo { get; set; }
    public int Faltas { get; set; }
}

public class Derrota : IEvento 
{
    public Derrota(Time timeEmCampo)
    {
        TimeEmCampo = timeEmCampo;
    }

    public Time TimeEmCampo { get; set; }
}
public class Vitoria : IEvento 
{
    public Vitoria(Time timeEmCampo)
    {
        TimeEmCampo = timeEmCampo;
    }

    public Time TimeEmCampo { get; set; }
}

```
Todas essas classes são eventos pois implementam IEvento, e todos recebem em seu construtor um time dessa forma o evento "é de um time, ou é para um time", semanticamente podemos dizer que:
> ApitoGol é para um time  
> ApitoFalta é para um time  
> GolMarcado é de um time  
> FaltaCometida é de um time  
> Derrota é de um time  
> Vitória é de um time  

___Essa semântica é permitida graças ao construtor desses eventos que recebem o time como parametro de entrada___  

Note também que as classes GolMarcado e FaltaCometida recebem o total de faltas o que permite aos observadores desse evento receberem os totais.


#### Regras do jogo
                
----
```c#
public class Regras
{
    public Regras(int limiteFaltas, int limieteGols, int limiteTempo)
    {
        LimiteFaltas = limiteFaltas;
        LimieteGols = limieteGols;
        LimiteTempo = limiteTempo;
    }

    public int LimiteFaltas { get; set; }
    public int LimieteGols { get; set; }
    public int LimiteTempo { get; set; }
}
```

No artigo anterior vimos que no frontend as regras são adicionadas ao container de injeção de dependências e ditam o comportamento do jogo, conforme lembrado abaixo:  

```c#
var serviceProvider = new ServiceCollection()
                .AddScoped(r => new Regras(limiteFaltas: 3,limieteGols: 1, limiteTempo: 60))
                .AddSingleton<CentralEventos>()
                .AddScoped<Juiz>()
                .BuildServiceProvider();
```
Veja que adicionamos ao container uma instância de Regras, CentralEventos e Juiz, vamos dar uma olhada na classe Juiz  
```c#
public class Juiz : Ator
{
    private Regras _regras;
    public Juiz(CentralEventos central,Regras regras) : base(central)
    {
        _regras = regras;

        central.OfType<GolMarcado>()
            .Timeout(TimeSpan.FromSeconds(regras.LimiteTempo))
            .Subscribe(
                e   => ApitarGol(e.TimeEmCampo,e.GolsMarcados),
                ex  => FinalizarPorTempo(ex)
            );

        central.OfType<FaltaCometida>()
            .Timeout(TimeSpan.FromSeconds(regras.LimiteTempo))
            .Subscribe(
                e  => ApitarFalta(e.TimeEmCampo, e.Faltas),
                ex => FinalizarPorTempo(ex)
            );

    }
    private void FinalizarPorTempo(Exception ex)
    {
        Console.WriteLine($"Juiz: Fim de jogo, o tempo acabou! {ex.Message}");
        FinalizarPartida();
    }

    public void IniciarPartida()
    {
        central.Notificar(new ApitoInicio());
    }

    private void ApitarFalta(Time time, int faltas)
    {
        Console.WriteLine($"Juiz: Infração cometida pelo time {time.Nome}");
        central.Notificar(new ApitoFalta(time));
        if (faltas == _regras.LimiteFaltas)
        {
            Console.WriteLine($"Juiz: Fim de partida o time {time.Nome} atingiu o nr. máximo de faltas");
            central.Notificar(new Derrota(time));
            FinalizarPartida();
        }
        
    }

    private void ApitarGol(Time time,int golsMarcados)
    {
        Console.WriteLine($"Juiz: Gol para o time {time.Nome}");
        central.Notificar(new ApitoGol(time));
        if (golsMarcados == _regras.LimieteGols)
        {
            Console.WriteLine($"Juiz: Fim de partida o time {time.Nome} atingiu o nr. máximo de gols");
            central.Notificar(new Vitoria(time));
            FinalizarPartida();
        }
    }

    private void FinalizarPartida()
    {
        central.Notificar(new ApitoFinal());
        central.Parar();
    }
}
```

Podemos dizer que o Juiz é um Ator, pois implementa a classe Ator (linha 1) e o juiz recebe a CentralEventos e as Regras através do construtor da classe Juiz (linhas 4-22).  
O próprio container de injeção de dependência garante que a instância de Regras recebida seja a instância adicionada no container anteriormente, caso o container não encontre essa instância haveria uma **Exception por null reference**   

Agora vamos dar uma olhada na classe Ator  

```c#
public class Ator
{
    protected CentralEventos central;

    public Ator(CentralEventos central)
    {
        this.central = central ?? throw new ArgumentNullException(paramName: nameof(central));
    }
}
```

Todo ator recebe uma instância de CentralEventos, essa instância também é resolvida pelo container de injeção de dependência, caso a instância seja nula teremos uma **Exception do tipo ArgumentNullException** e voltando a classe Juiz veja que a instância de CentralEventos é repassada para a classe Ator através da palavra chave **:base(central)**

```c#
public class Juiz : Ator
{
    private Regras _regras;
    public Juiz(CentralEventos central,Regras regras) : base(central)
    {
...
```

#### CentralEventos  

----
```c#
public class CentralEventos : IObservable<IEvento>
{
    private Subject<IEvento> assinatura = new Subject<IEvento>();
    public IDisposable Subscribe(IObserver<IEvento> observer)
    {
        return assinatura.Subscribe(observer);
    }

    public void Notificar(IEvento evento)
    {
        assinatura.OnNext(evento);
    }

    public void Parar()
    {
        assinatura.Dispose();
    }
}
```

Está é a classe responsável por **Notificar** todos os **Observadores** dos eventos que ocorrem no campo!  
Então veja que esta classe implementa a interface ***IObservable*** onde Observable seja do tipo IEvento.  
Lembra do IEvento??  
Nossa interface que apenas tipa todos os eventos que foram criados.  
Então basicamente nessa linha 1 da classe CentralEventos estamos construindo a seguinte semântica:
> Toda classe que implementa IEvento é uma classe **Observável** (OBSERVABLE) e a CentralEventos é responsável em notificar todos os **Observadores** (OBSERVERS).  



Para facilitar o tratamento dos Observers utilizo a classe Subject (linha 3), indicando que Subject tratará o tipo IEvento e por implementar IObservable, a CentralEventos é obrigada a implementar o método Subscribe (linha 4) 
Sendo assim, a instância de Subject (assinatura) é associada ao Observer. E quando a central de eventos é notificada de algum evento através do método Notificar (linha 9) a **reação** ao evento no Observador é disparada através do método OnNext(), bem similar a um delegate! Não? 

Voltando para a classe Juiz podemos ler nas linhas abaixo que estão no nosso construtor:

```c#
central.OfType<GolMarcado>()
        .Timeout(TimeSpan.FromSeconds(regras.LimiteTempo))
        .Subscribe(
            e   => ApitarGol(e.TimeEmCampo,e.GolsMarcados),
            ex  => FinalizarPorTempo(ex)
        );

central.OfType<FaltaCometida>()
    .Timeout(TimeSpan.FromSeconds(regras.LimiteTempo))
    .Subscribe(
        e  => ApitarFalta(e.TimeEmCampo, e.Faltas),
        ex => FinalizarPorTempo(ex)
    );
```

a seguinte semântica:  
___Quando a central notificar um GolMarcado (linha 1) dentro da regra de limite de tempo (linha 2) o juiz está inscrito (linha 3) para **reagir** apitando Gol para o time do GolMarcado e seu total de gols marcados (linha 4) e caso o limite de tempo seja atingido o juiz está inscrito para finalizar a partida (linha 5)___  

> Isso, meus amigos, é a magia da programação funcional, dizer o que se quer e não como se quer!  (Obs. R.I.P. if) :pray: 


Que tal agora você tentar identificar a semântica nas linhas 8-13?  

Simples, não é?  

Agora vamos dar uma olhada na classe Time:  
```c#
public class Time : Ator
{
    public string Nome { get; private set; }
    private int GolsMarcados = 0;
    private int FaltasCometidas = 0;
    public Time(CentralEventos central, string nome) : base(central)
    {
        Nome = nome;
        central.OfType<ApitoInicio>()
            .Subscribe(
                ini => Console.WriteLine($"{Nome}: Vamos lá pessoal, pra cima deles!")
            );
        central.OfType<ApitoGol>() // Gol contra o time
            .Where(gol => !gol.TimeEmCampo.Equals(this))
            .Subscribe(
                gol => Console.WriteLine($"{Nome}: Ânimo pessoal!!! Vamos marcar mais forte")
            );

        central.OfType<ApitoGol>() // Gol do Time
            .Where(gol => gol.TimeEmCampo.Equals(this))
            .Subscribe(
                gol => Console.WriteLine($"{Nome}: Muito bem pessoal!!!")
            );

        central.OfType<ApitoFalta>() //Falta do outro time
            .Where(falta => !falta.TimeEmCampo.Equals(this))
            .Subscribe(
                falta => Console.WriteLine($"{Nome}: pô seu juiz, cadê o cartão?")
            );

        central.OfType<ApitoFalta>() //Falta do time
            .Where(falta => falta.TimeEmCampo.Equals(this))
            .Subscribe(
                falta => Console.WriteLine($"{Nome}: foi na bola seu juiz, que injusto!")
            );

        central.OfType<Derrota>() // derrota por faltas do time
            .Where(falta => falta.TimeEmCampo.Equals(this))
            .Subscribe(
                falta => Console.WriteLine($"{Nome}: Esse nr de faltas bem que poderia ser maior :(")
            );

        central.OfType<Derrota>() // derrota por faltas do outro time
            .Where(falta => !falta.TimeEmCampo.Equals(this))
            .Subscribe(
                falta => Console.WriteLine($"{Nome}: Jogar com violência da nisso!")
            );

        central.OfType<Vitoria>() // vitoria do time
            .Where(v => v.TimeEmCampo.Equals(this))
            .Subscribe(
                vit => Console.WriteLine($"{Nome}: Viva o {Nome}!")
            );

        central.OfType<Vitoria>() // vitoria do outro time
            .Where(v => !v.TimeEmCampo.Equals(this))
            .Subscribe(
                vit => Console.WriteLine($"{Nome}: Na próxima {vit.TimeEmCampo.Nome} vocês vão ver!")
            );

        central.OfType<ApitoFinal>()
            .Subscribe(
                fim => central.Parar()
            );
    }

    public void MarcarGol()
    {
        GolsMarcados++;
        central.Notificar(new GolMarcado(this,GolsMarcados));
    }

    public void CometerFalta()
    {
        FaltasCometidas++;
        central.Notificar(new FaltaCometida(this,FaltasCometidas));
    }
}
```

Note que time também é um Ator, que utiliza variáveis para controlar o nr. de gols marcados e o nr. de faltas marcadas.  
As **reações** aos eventos, é só mais do mesmo!  
Uma atenção especial pode ser dada aos métodos **MarcarGol()** e **CometerFalta()**  
Veja que os gols ou faltas são computados e a instância de CentralEventos é notificada dos respectivos eventos e assim tanto o Juiz quanto o time adversário recebem e **reagem** aos eventos conforme apresentado anteriormente.  

### Conclusão

---

A utilização das extensões de System.Reactive facilitam o trabalho de construir aplicações REATIVAS valendo-se da utilização de um paradigma mais funcional com sintaxe semelhantes ao LinQ e um conjunto de APIs para facilitar o uso de Threads síncronas e assíncronas com Schedulers.  

> Nota: O objetivo deste artigo é demonstrar o poder das extensões do System.Reactive mas não aborda todos os assuntos e complexidades que podem estar envolvidos no desenvolvimento de aplicações Reativas!

### Desafio

---
Agora que tal você implementar esse jogo com o narrador, treinador e torcida? 


> Espero que todos gostem e aproveitem este conhecimento! Aguardo o seu pull request :laughing: