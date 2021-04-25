---
layout: post
title:  "ConfigureWait FAQ - Traduzido"
date:   2021-04-24 14:20:00 -0300
categories: C#
---

O texto a seguir é uma tradução livre do artigo [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/) de [Stephen Toub](https://devblogs.microsoft.com/dotnet/author/toub/) membro do time de desenvolvimento do .NET

.Net adicionou o async/await  as linguagens e libs há aproximadamente 7 anos atrás. Naquela época isso se alastrou rapidamente, como uma queimada, não apenas no ecossistema .NET, mas também em uma grande quantidade de outras linguagens e frameworks. Isso proporcionou muitas melhorias no .Net, em termos de construções adicionais na linguagem que utilizam assincronismo, APIs oferecendo suporte ao async, e melhorias fundamentais na infraestrutura que fizeram async/await pulsar (particularmente em performance e melhorias no .Net Core em relação a capacidade de habilitar diagnósticos).  

Embora, um aspecto do async/await continua a trazer perguntas, o <span style="color:red">ConfigureAwait</span>. Neste post eu espero responder a maioria deles. Eu pretendo que este post seja tanto, lido do início até o fim, além de também poder ser usado como referência futura tipo Perguntas Frequentes (FAQ).  

Para realmente entender o <span style="color:red">ConfigureAwait</span>, nós precisamos voltar um pouco...

#### O que é SynchornizationContext?
Na documentação do  [System.Threading.SynchronizationContext](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext?view=net-5.0), diz que: "Prove a funcionalidade básica propagar o contexto de sincronização em várias Models de sincronização (synchronization models)".

Para 99,9% dos casos de uso, o SynchronizationContext é apenas um tipo que prove um método Post virtual, que transporta um delegate para ser executado assincronamente (existem uma variedade de outros membros virtuais no SynchronizationContext, mas eles são muito menos usados, são irrelevantes nesta discussão). O Post do tipo base literalmente apenas chama <span style="color:red">ThreadPool.QueueUserWorkItem</span> para assincronamente apenas invocar o delegate transportado. Entretanto, os tipos derivados sobrescrevem o Post para permitir que o delegate transportado seja executado no lugar e no tempo mais apropriado.  

Por exemplo, o Windows Forms tem um tipo derivado do SynchronizationContext que sobrescreve o Post para fazer o equivalente do <span style="color:red">Control.BeginInvoke</span>; que significa que qualquer chamada para este método *Post* irá invocar o delegate transportado mais tarde em algum ponto da thread associada com o Controle de tela em questão. Mais conhecido como "UI thread". Aplicações Windows Forms se apoiam na manipulação de mensagem do Win32 e tem um "message loop" rodando na UI thread, o qual simplesmente fica esperando novas mensagens chegarem para serem processadas. Essas mensagens podem ser cliques e movimentos do mouse, digitação no teclado, eventos de sistema, delegates que precisem ser invocados, etc. Então dada uma instância de SynchronizationContext da UI Thread de uma aplicação Windows Forms, executar um delegate nessa IU Thread, é necessário apenas passar esse delegate para o <span style="color:red">Post</span>.

O mesmo ocorre com o WPF. Ele tem seu próprio SynchronizationContext com o *Post* sobrescrito que similarmente "transporta" um delegate para a UI thread (via Dspatch.BeginInvoke), neste caso gerenciado por um WPF Dispatcher em vez de um Controle do Windowns Form.  

E para Windows RunTime (WinRT). Também tem seu próprio SynchronizationContext com o *Post* sobrescrito que também enfileira o delegate para a IU thread através do seu CoreDispatcher.  

Isso vai além de simplesmente "executar o delegate na UI thread". Qualquer um pode implementar um SynchronizationContext com um *Post* que faz alguma coisa. Por exemplo, eu posso não me importar em que thread um delegate irá ser executado. Mas eu quero me assegurar que qualquer delegate enviado no *Post* para o meu SynchronizationContext será executado com algum grau de limitação de concorrência. E eu posso conseguir fazer isso com uma implementação personalizada do SynchronizationContext como esta:
```c#
internal sealed class MaxConcurrencySynchronizationContext : SynchronizationContext
{
    private readonly SemaphoreSlim _semaphore;

    public MaxConcurrencySynchronizationContext(int maxConcurrencyLevel) =>
        _semaphore = new SemaphoreSlim(maxConcurrencyLevel);

    public override void Post(SendOrPostCallback d, object state) =>
        _semaphore.WaitAsync().ContinueWith(delegate
        {
            try { d(state); } finally { _semaphore.Release(); }
        }, default, TaskContinuationOptions.None, TaskScheduler.Default);

    public override void Send(SendOrPostCallback d, object state)
    {
        _semaphore.Wait();
        try { d(state); } finally { _semaphore.Release(); }
    }
}
```
De fato, o framework do unit testing prove um SynchronizationContext muito parecido com esse, o qual é usado para limitar a quantidade de código associado com  testes que podem rodar concorrentemente.  

O benefício disso tudo é o mesmo de qualquer abstração: Ele prove uma única API que pode ser usada para enfileirar um delegate para manipular qualquer coisa que o seu criador desejar, sem a necessidade de saber o detalhe de sua implementação. Então, se estou escrevendo um Lib e eu quero ir externamente e executar algo, assim enfileirar o delegate de volta a localização do contexto inicial, eu apenas preciso pegar o SynchronizationContext dele, colocá-lo em espera, e então quando eu finalizar meu trabalho, chamo o *Post* naquele contexto para liberar o delegate que eu queria invocar, ou para WPF eu deveria pegar o Dispatcher e usar o seu BeginInvoke, ou para xunit eu deveria de algum modo adquirir seu contexto e colocá-lo na fila. Eu apenas preciso pegar o SynchronizationContext e usá-lo depois. Para conseguir isso o SynchronizationContext fornece uma propriedade chamada Current, Para que eu consiga atingir o objetivo mencionado anteriormente, eu posso escrever um código como este:
```c#
public void DoWork(Action worker, Action completion)
{
    SynchronizationContext sc = SynchronizationContext.Current;
    ThreadPool.QueueUserWorkItem(_ =>
    {
        try { worker(); }
        finally { sc.Post(_ => completion(), null); }
    });
}
```

Um framework que quer expor um contexto personalizado através da propriedade Current usa o método SynchronizationContext.SetSynchronizationContext.  

#### O que é um TaskScheduler?

SynchronizationContext é uma abstração de um "scheduler". As vezes os frameworks têm sua própria abstração de um scheduler, e o  <span style="color:red">System.Threading</span>. Tasks não é diferente. Quando as tasks são devolvidas por um delegate como tal, elas podem ser empilhadas e executadas, Elas são associadas com um  <span style="color:red">System.Threading.Tasks.TaskScheduler</span>. Assim como o  <span style="color:red">SynchronizationContext</span> fornece um método virtual  <span style="color:red">Post</span> para empilhar uma invocação do delegate (com a implementação invocando depois o delegate via mecanismos típicos de invocação de delegate), <span style="color:red">TaskScheduler</span> fornece uma abstração do método <span style="color:red">QueueTask</span> (com uma implementação que invoca esta <span style="color:red">Task</span> através do método <span style="color:red">ExecuteTask</span>).  

O scheduler padrão retornado pelo <span style="color:red">TaskScheduler.Default</span> é o thread pool, más é possível derivar da classe <span style="color:red">TaskScheduler</span> e sobrescrever os métodos relevantes para alterar o comportamento de quando e onde uma <span style="color:red">Task</span> é acionada. Por exemplo, as Libs do Core incluem o tipo <span style="color:red">System.Threading.Tasks.ConcurrentExclusiveSchedulerPair</span>. Uma instância dessa classe expõe duas propriedades do tipo <span style="color:red">TaskScheduler</span>, uma chamada <span style="color:red">ExclusiveScheduller</span> e outra <span style="color:red">ConcurrentScheduler</span>. Tasks agendadas no <span style="color:red">ConcurrentScheduler</span> podem ser executadas concorrentemente, mas subjetivamente no limite fornecido para o <span style="color:red">ConcurrentExclusiveSchedulerPair</span> no momento de sua construção (parecido com o <span style="color:red">MaxConcurrencySynchronizationContext</span> demonstrado anteriormente), e nenhuma task no <span style="color:red">ConcurrentScheduler</span> será executada enquanto uma Task agendada no <span style="color:red">ExclusiveScheduler</span> estiver rodando, com apenas uma Task exclusiva de cada vez sendo permitida rodar... dessa forma, o comportamento é muito parecido com uma trava leitor/escritor. <span style="color:red">(reader/writer-lock)</span>  

Assim como <span style="color:red">SynchronizationContext, TaskScheduler</span> também tem uma propriedade Current, o qual retorna o "current" (atual) <span style="color:red">TaskScheduler</span>. Diferente do <span style="color:red">SynchronizationContext</span>, entretanto, não existe um método para alterar o shceduler atual. Ao invés disso, o scheduler atual é aquele que está associado a task atualmente em execução, e o scheduler é fornecido ao sistema como parte da Task original. Então por exemplo, este programa irá exibir "True", pois a lambda usada com <span style="color:red">StartNew</span> é executada no <span style="color:red">ExclusiveScheduler</span> do <span style="color:red">ConcurrentExclusiveSchedulerPair</span> e veremos <span style="color:red">TaskScheduler.Current</span> recebendo esse scheduler:
```c#
using System;
using System.Threading.Tasks;

class Program
{
    static void Main()
    {
        var cesp = new ConcurrentExclusiveSchedulerPair();
        Task.Factory.StartNew(() =>
        {
            Console.WriteLine(TaskScheduler.Current == cesp.ExclusiveScheduler);
        }, default, TaskCreationOptions.None, cesp.ExclusiveScheduler).Wait();
    }
}
```
Curiosamente, o <span style="color:red">TaskScheduler</span> fornece um método estático<span style="color:red"> FromCurrentSynchronizationContext</span>, que cria um novo <span style="color:red">TaskScheduler</span> que enfileira <span style="color:red">Task</span> para serem executadas no que quer que o <span style="color:red">SynchronizationContext.Current</span> retornar, usando seu método <span style="color:red">Post</span> para enfileirar tasks.  

#### Como SynchronizationContext e TaskScheduler se relacionam com o await?  

Considere escrever uma aplicação UI com um botão. Ao clicar no botão, queremos fazer o download de algum texto de um site da web e colocar esse texto como conteúdo do botão. O botão deve apenas ser acessado pela UI thread que o criou, então quando nós finalizarmos o download do texto de data e hora com sucesso e quisermos armazenar esse resultado no botão, nós teremos de fazer isso através da thread que criou o botão. Se não fizermos isso, recebermos um erro do tipo:
```c#
 System.InvalidOperationException: 'The calling thread cannot access this object because a different thread owns it.'
```

Se formos escrever este código, poderíamos usar  <span style="color:red">SynchronizationContext</span> conforme exibido anteriormente, para enviar a alteração de conteúdo de volta para o contexto da thread original, via TaskScheduler:

```c#
private static readonly HttpClient s_httpClient = new HttpClient();

private void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    s_httpClient.GetStringAsync("http://example.com/currenttime").ContinueWith(downloadTask =>
    {
        downloadBtn.Content = downloadTask.Result;
    }, TaskScheduler.FromCurrentSynchronizationContext());
}
```
ou usando o SynchronizationContext diretamente:
```c#
private static readonly HttpClient s_httpClient = new HttpClient();

private void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    SynchronizationContext sc = SynchronizationContext.Current;
    s_httpClient.GetStringAsync("http://example.com/currenttime").ContinueWith(downloadTask =>
    {
        sc.Post(delegate
        {
            downloadBtn.Content = downloadTask.Result;
        }, null);
    });
}
```
Ambas as soluções, no entanto, usam callbacks explicitamente. Em vez disso, gostaríamos de escrever o código naturalmente com async / await::

```c#
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime");
    downloadBtn.Content = text;
}
```
Isto "simplesmente funciona",alterando o conteúdo na UI thread com sucesso, porque exatamente como na nossa implementação acima, usando <span style="color:red">await</span> em uma <span style="color:red">Task</span> mantém a atenção no <span style="color:red">SynchronizationContext.Current</span> padrão, assim como também <span style="color:red">TaskScheduler.Current</span>. Quando você usa <span style="color:red">wait</span> no c#, o compilador transforma o código em uma Task (chamando <span style="color:red">GetAwaiter</span>) o "aguardado" (neste caso, a <span style="color:red">Task</span>) para um "awaiter" (newste caso, um <span style="color:red">TaskAwaiter</span>). Esse "awaiter" é responsável por captiurar o callback (geralmente descrito como a "continuação") Isso será retornado para a "máquina de estado" quando o objeto marcado com "await" completar, e isso é feito com qualquer que seja o contexto/Scheduler que foi capturado no momento que o callback foi registrado. Embora não seja exatamente o código usado (há otimizações e ajustes adicionais empregados), sendo algo assim: 
```c#
object scheduler = SynchronizationContext.Current;
if (scheduler is null && TaskScheduler.Current != TaskScheduler.Default)
{
    scheduler = TaskScheduler.Current;
}
```
Em outras palavras, primeiro é verificado se existe um SynchronizationContext configurado, e se não existir, se há um TaksScheduler não padrão em execução. Se encontrar um, quando o callback estiver pronto para ser acionado, será utilizado o scheduler capiturado; Caso contrário, geralmente executará o callback aguardado como parte da operação em execução.

#### O que o ConfigureAwait(false) faz?  

O método <span style="color:red">ConfigureAwait</span> não tem nada de especial: ele não é reconhecido em nenhuma forma especial pelo compilador ou pela runtime. Ele é um simples método que retorna uma estrutura (uma <span style="color:red">ConfiguredTaskAwaitable</span>) que embala a task original da chamada, assim como o parâmetro especificado de valor booleano. Lembre-se que o <span style="color:red">await</span> pode ser usado com qualquer tipo que expõe o padrão correto. Retornando um tipo diferente, quer dizer que quando o compilador acessar as instâncias do método <span style="color:red">GetAwaiter</span> (que é parte do padrão) ele está executando então fora do tipo retornado vindo do <span style="color:red">ConfigureAwait</span> em vez de fora da task diretamente, e então fornece um gancho para alterar o comportamento de como o <span style="color:red">await</span> se comporta através deste awaiter customizado.

Especificamente, aguardar o tipo retornado de <span style="color:red">ConfigureAwait(*ContinueOnCapturedContext:* false)</span> em vez de aguardar a Task diretamente acaba impactando na lógica demonstrada anteriormente em como o context/scheduler alvo é capturado. Isso faz efetivamente a lógica demostrada anteriormente mais parecida com isso:  
```c#
object scheduler = null;
if (continueOnCapturedContext)
{
    scheduler = SynchronizationContext.Current;
    if (scheduler is null && TaskScheduler.Current != TaskScheduler.Default)
    {
        scheduler = TaskScheduler.Current;
    }
}
```

Em outras palavras, passando <span style="color:red">false</span>, mesmo que haja um contexto em execução ou um scheduler para chamar de volta, isso irá fingir que não existe.

#### Porque eu iria querer usar o ConfigureAwait(false)?

<span style="color:red">ConfigureAwait(*ContinueOnCapturedContext:* false)</span> é usado para evitar o retorno obrigatório do callback no contexto ou scheduler original. Isso tem alguns benefícios.  

*Melhoria de performance.* Há um custo no enfileiramento do callback em vez de somente executá-lo, ambos porque há trabalho extra (e tipicamente alocação extra) envolvida, mas também porque significa que certas otimizações que gostaríamos de empregar em tempo de execução não podem ser usadas (nós podemos fazer mais otimizações quando sabemos exatamente como o callback será enviado, mas se ele for manipulado de fora em uma implementação arbitrária de uma abstração, as vezes podemos estar limitados). Para pontos muito utilizados, até mesmo os custos extras em verificar o <span style="color:red">SynchronizationContext</span> atual e o <span style="color:red">TaskScheduler</span> atual (ambos envolvem acesso a threads státicas) podem adicionar uma sobrecarga considerável. Se o código depois de um <span style="color:red">await</span> pode evitar todo esse custo: não precisará enfileiras desnecessariamente, pode utilizar toda otimização que é possível, e pode evitar acesso desnecessário a thread stática.  

*Evitar deadlocks (congelamento da aplicação).* Considere um método de uma lib que usa <span style="color:red">await</span> para obter o resultado de algum download na rede. Você chama esse método e sincronamente bloqueia para aguardar que ele esteja completo, usando <span style="color:red">.Wait()</span> ou <span style="color:red">.Result</span> ou <span style="color:red">.GetAwaiter().GetResult()</span> fora do objeto Task retornado. Agora considere o que ocorre se sua chamada acontece quando o <span style="color:red">SynchronizationContext</span> atual é um dos que limita o número de operações que podem ser rodadas em 1, seja explicitamente via algo como <span style="color:red">MaxConcurrencySynchronizationContext</span> demonstrado anteriormente, ou implicitamente por ser um contexto que tem apenas uma thread que pode ser usada, por exemplo, a IU Thread. Então voce chama o método nessa thread única e então bloquei aguardando que a operação complete. A operação abre a conexão com  a rede de download e aguarda. Já que por padrão aguardar uma Task vai capturar o <span style="color:red">SynchronizationContext</span> atual, e o fará, e quando o download na rede for completo, o processo será enfileirado de volta para o SynchronizationContext o callback que irá chamar o lembrete da operação. Mas a única thread que pode processar o callback enfileirado está no momento bloqueado pelo seu código aguardando a operação completar. E essa operação não vai ser concluída até o callback ser processado. *Deadlock!* Isso pode ocorrer mesmo quando o contexto não está limitado a uma única thread, mas quando os recursos são limitados de alguma maneira. Imagine a mesma situação, exceto que usaremos MaxConcurrencySynchronizationContext com o limite de 4. E em vez de fazer apenas uma chamada da operação, não enfileiramos neste contexto 4 chamadas, cada uma delas faz a chamada e bloqueia aguardando que seja completada. Nós agora continuamos bloqueados todos os recursos aguardando pelo método assíncrono completar, e a única coisa que permitirá que estes métodos assíncronos completem será se o callback deles puder ser processado pelo contexto que já está totalmente consumido. Novamente, deadlock! Se em vez disso o método da lib tivesse usado <span style="color:red">ConfigureAwait(false)</span>, O callback não seria enfileirado para retornar no contexto original, evitando cenários de deadlocks (congelamento da aplicação).

#### Por que eu iria querer usar ConfigureAwait(true)?

Você não irá querer isso, a menos que você queira indicar que explicitamente que você não está usando <span style="color:red">ConfigureAwait(false)</span> (por exemplo para silenciar algum tipo de alerta de análise estática). <span style="color:red">ConfigureAwait(true)</span> não faz nada significativo. Quando comparando <span style="color:red">await task</span> com <span style="color:red">task.ConfigureAwait(true)</span> eles são idênticos funcionalmente. Se você ver <span style="color:red">ConfigureAwait(true)</span> nos códigos de produção, você pode deletá-los sem nenhum efeito nocivo.  

The método <span style="color:red">ConfigureAwait</span> aceita o parâmetro booleano porque existem algumas situações específicas na qual você quer passar em uma variavel o controle da situação. Mas em 99% dos casos de uso é com o valor fixo "falso" no valor do argumento, <span style="color:red">ConfigureAwait(false)</span>.

#### Quando eu deveria usar <span style="color:red">ConfigureAwait(false)</span>?  

Depende: Você está implementando algo em nível de aplicação ou uma biblioteca de uso comum?  

Quando escrever aplicações, você normalmente quer o comportamento padrão (por isso este é o comportamento padrão). Se uma aplicação modelo/ambiente (exemplo: Windows Forms, WPF, ASP.NET Core, etc.) publicar um <span style="color:red">SynchronizationContext</span> customizado, quase que certamente há um bom motivo para isso: isso está provendo um modo do código que se importa como synchronization context interage com o modelo/ambiente da aplicação de forma apropriada. Portanto, se você escrever um manipulador de eventos em uma aplicação Windows Forms, escrever um teste unitário no xunit, escrever um ASP.NET MVC controller, se o modelo de aplicativo publica ou não um SynchronizationContext, você quer usara esse <span style="color:red">SynchronizationContext</span> se ele existir. E isso significa o padrão <span style="color:red">ConfigureAwait(true)</span>. Você faz apenas o uso do <span style="color:red">await</span>, e a coisa certa acontece em relação aos callbacks/continuations em questão, sendo enviado de volta para o contexto original se algum existiu. Isso nos leva a orientação geral de: *Se você está escrevendo código no nível da aplicação, não use <span style="color:red">ConfigureAwait(false)</span>*. Se você relembrar do manipulador de evento de Clique exemplificado anteriormente nesse post:
```c#
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime");
    downloadBtn.Content = text;
}
```
A atribuição de downloadBtn.Contet = text precisa ser feita de volta no contexto original. Se o código tivesse violado essa diretriz e em vez disso, usou <span style="color:red">ConfigureAwait(false)</span> quando não deveria:
```c#
private static readonly HttpClient s_httpClient = new HttpClient();

private async void downloadBtn_Click(object sender, RoutedEventArgs e)
{
    string text = await s_httpClient.GetStringAsync("http://example.com/currenttime").ConfigureAwait(false); // bug
    downloadBtn.Content = text;
}
```
Mal funcionamento será o resultado. O mesmo aconteceria em uma aplicação ASP.NET clássica baseado no HttpContext.Current; usando ConfigureAwait(false) e assim tentando usar <span style="color:red">HttpContext.Current</span> provavelmente resultará em problemas.  

Em contrates, bibliotecas de "propósito geral" em parte porque elas não se preocupam com o ambiente que serão usadas. Você pode utilizá-la apartir de uma aplicação web ou aplicação cliente, ou apartir de um teste, não importa, já que o código da biblioteca é agnóstico ao modelo da aplicação, poderá ser usado nela. Sendo agnóstico também significa que não irá fazer nada que necessita interagir com o modelo da aplicação de um modo restrito. Exemplo, não irá acessar controles da UI, porque uma biblioteca de uso geral não tem conhecimento dos controles de UI. Desde que não precisemos rodar o código em um ambiente em particular, podemos evitar coninuations/callback de volta para o contexto original, e fazemos isso usando o  <span style="color:red">ConfigureAwait(false)</span> e ganhamos ambos o benefício da performance e leitura que esse comando trás. Isso nos leva a uma orientação geral de: * se estamos escrevendo uma biblioteca de uso comum, <span style="color:red">USE ConfigureAwait(false)</span>* em todos os <span style="color:red">await;</span> com apenas algumas exceções, nos casos em que não está sendo usado, muito provávelmente isso é um bug a ser corrigido. por exemplo, [este PR](https://github.com/dotnet/corefx/pull/38610) corrige um <span style="color:red">ConfigureAwait(false)</span> que estava faltando na chamada <span style="color:red">HttpClient</span>.  

Assim como toda orientação, é claro, pode haver exceções, locais onde isso possa não fazer sentido. Por exemplo, uma das maiores isenções (ou pelo menos categorias que requerem reflexão) em bibliotecas de uso gera é quando essas bibliotecas tem APIs que carregam delegates a serem invocados. Nestes casos, o chamador da lib está passando potencialmente código em nível da aplicação para ser invocado dentro da lib, o que então efetivamente torna discutível as suposições de "propósito geral" da biblioteca. Considere por exemplo uma versão asynchrona do LINQ onde os métodos, exemplo:
```c#
public static async IAsyncEnumerable<T> WhereAsync(this IAsyncEnumerable<T> source, Func<T, bool> predicate)
```
Este predicado precisa ser invocado de volta no  <span style="color:red">SynchronizationContext</span> do chamador? Isso depende da decisão de implementação do <span style="color:red">WhereAsync</span>, e essa é a razão que pode ter feito ele não usar o <span style="color:red">ConfigureAwait(false)</span>.

Mesmo com esses casos especiais, a orientação geral é válida e é um bom ponto de partida: usar <span style="color:red">ConfigureAwait(false)</span> se você estiver escrevendo um código para biblioteca de uso geral / agnóstico de modelo de aplicação, e senão,  não use.

#### O ConfigureAwait(false) garante que o callback não será executado no contexto original?

Não. Isso garante que não será enfileirado de volta para o contexto original. Mas não significa que o código depois de um <span style="color:red">await task.ConfigureAwait(false)</span> não irá ser executado no contexto original.  Isso porque aguardar uma task "aguardável" já completa, apenas mantém a execução do <span style="color:red">await</span> de forma síncrona em vez de forçar que algo seja enfileirado de volta. Então, se você aguarda um task que já foi completa no momento que ela está sendo aguardada, independentemente de você ter usado ou não o <span style="color:red">ConfigureAwait(false)</span> o código imediatamente depois disso irá continuar a ser executado na thread atual em qualquer que seja este contexto atual.

### Todo bem usar ConfigureAwait(false) apenas no primeiro await e não no resto?

Em geral, não. o tópico anterior. Se o <span style="color:red">await task.ConfigureAwait(false)</span> envolve uma task que já está completa no momento que é aguardada (o que é naverdae incrivelmente comum), então o <span style="color:red">ConfigureAwait(false)</span> será sem sentido, conforme a thread continua a executar o código do método depois dele e continua no mesmo contexto que estava anteriormente.

Uma exceção notável a isso é se você sabe que o primeiro <span style="color:red">await</span> será sempre completado de forma síncrona e a coisa aguardada irá invocar seu callback em um ambiente livre de um SynchronizationContext customizado ou um TaskScheduler. Por exemplo, <span style="color:red">CriptoStream</span> na Lib .NET quer garantir que seu código potencialmente intensivo de computação não seja executa como parte da invocação síncrona do chamador, então ele usa [um awaiter customizado](https://github.com/dotnet/runtime/blob/4f9ae42d861fcb4be2fcd5d3d55d5f227d30e723/src/libraries/System.Security.Cryptography.Primitives/src/System/Security/Cryptography/CryptoStream.cs#L205) para garantir que tudo depois do primeiro <span style="color:red">await</span> execute em uma thread do thread pool. Embora, mesmo neste caso você notará que o próximo <span style="color:red">await</span> continua a usar o <span style="color:red">ConfigureAwait(false)</span> tecnicamente isso não é necessário, mas isso torna a revisão do código muito mais fácil, caso contrário, toda vez que este código é examinado, ele não requer uma análise para entender por que ConfigureAwait(false) foi deixado de fora.  

#### Posso usar Task.Run para evitar usar ConfigureAwait(false)?

Sim, se você escrever:

```c#
Task.Run(async delegate
{
    await SomethingAsync(); // won't see the original context
});
```

Assim um <span style="color:red">ConfigureAwait(false)</span> nesta chamada de <span style="color:red">SomethingAsync()</span> será uma NOP (no operation/sem execução), porque o delegate passou para Task.Run será executado em uma thread do thread pool, sem nenhum código do usuário no alto da pilha, de modo que <span style="color:red">SynchronizationContext.Current</span> retornará nulo. Além disso, <span style="color:red">Task.Run</span> usa implicitamente <span style="color:red">TaskScheduler.Default</span>, o significa que <span style="color:red">TaskScheduler.Current</span> dentro de um delegate também irá retornar <span style="color:red">Default</span>. E isso significa que <span style="color:red">await</span> irá apresentar o mesmo comportamento, independente do uso ou não do  <span style="color:red">ConfigureAwait(false)</span>. Isso também não oferece nenhuma garantia do que o código dentro dessa lambda pode fazer. Se você tiver o código:
```c#
Task.Run(async delegate
{
    SynchronizationContext.SetSynchronizationContext(new SomeCoolSyncCtx());
    await SomethingAsync(); // will target SomeCoolSyncCtx
});
```

Assim o código dentro de <span style="color:red">SomethingAsync(false)</span> irá de fato ver <span style="color:red">SynchronizationContext.Current</span> como uma instância de <span style="color:red">SomeCoolSyncCtx</span> e tanto esse <span style="color:red">await</span> e outros <span style="color:red">awaits</span> não configurados dentro de <span style="color:red">SomethingAsync()</span> vão voltar para esse contexto. Portanto, para usar essa abordagem, você precisa entender o que todo o código que você está enfileirando pode ou não pode fazer e se essas ações poderiam ir contra a sua intenção.  

Essa abordagem também demonstra o custo de criar/enfileirar um objeto task adicional. Que pode ou não pode importar para sua aplicação ou lib dependendo da sua necessidade de desempenho.  

Também tenha em mente que estes truques podem causar mais problemas do que eles valem e ter outras consequências indesejadas. Por exemplo, ferramentas de análise estática (exp. analisadores de Roslyn) foram escritos para marcar awaits que não usam <span style="color:red">ConfigureAwait(false)</span>. tais como [CA2007](https://docs.microsoft.com/en-us/visualstudio/code-quality/ca2007?view=vs-2019). Se você ativar este analisador e então aplicar um truque como este apenas para evitar a chamada do <span style="color:red">ConfigureAwait(false)</span>, há uma boa chance do analisador marcá-la, e na verdade criar mais trabalho para você. Então talvez você desabilite o analisador por causa da barulheira dele, aí você acaba deixando passar outros lugares no código onde na verdade deveriam estar usando <span style="color:red">ConfigureAwait(false)</span>.

#### Posso usar SynchronizationContext.SetSynchronizationContext para evitar usar ConfigureAwait(false)?

Não. Bom, talvez. Isso vai depender do código em questão.

Alguns desenvolvedores escrevem códigos como este:
```c#
Task t;
SynchronizationContext old = SynchronizationContext.Current;
SynchronizationContext.SetSynchronizationContext(null);
try
{
    t = CallCodeThatUsesAwaitAsync(); // aguaradar aqui dentro não verá o contexto original
}
finally { SynchronizationContext.SetSynchronizationContext(old); }
await t; // irá ainda apontar para o contexto original
```

Na esperança de que isso irá fazer o código dentro de <span style="color:red">CallCodeThatUsesAwaitAsync</span> veja o contexto atual como nulo.
E isso acontecerá. Entretanto, acima não irá fazer nada que afete o <span style="color:red">await</span> de enxergar o <span style="color:red">TaskScheduler.Current</span>, então se este código estiver rodando em algum <span style="color:red">TaskScheduler, await</span> customizado dentro de  <span style="color:red">CallCodeThatUsesAwaitAsync</span> (E este não usa  <span style="color:red">ConfigureAwait(false)</span>) ele ainda o enxergará e será enfileirado de volta para aquele TaskScheduler.  

Todas as mesmas advertências se aplicam no Task.Run FAQ anterior: Existem implicações de desempenho nesse tipo de solução alternativa, e código dentro do try poderia também contrariar essas tentativas configurando um contexto diferente (ou invocando código com um outro <span style="color:red">TaskScheduler</span> diferente do convencional).  

Com esse padrão, você tembém precisa ser cuidadoso sobre varições sutis:

```c#
SynchronizationContext old = SynchronizationContext.Current;
SynchronizationContext.SetSynchronizationContext(null);
try
{
    await t;
}
finally { SynchronizationContext.SetSynchronizationContext(old); }
```
Ve o problema? É um pouco difícil de ver, mas é também potencialmente muito impactante. Não há garantia que o <span style="color:red">await</span> acabará invocando o callback/continuation na thread original, talvez de fato não aconteça na thread original, o que poderia levar atividades subsequentes nesta thread a ver o contexto errado (para contra atacar isso, modelos de aplicações bem escritos que configuram um contexto customizado geralmente adicionam código para resetar manualmente isso, antes de invocar qualquer outro código). E mesmo que isso aconteça de ser executado na mesma thread, isso pode acontecer um pouco antes do início, neste caso o contexto não será restaurado corretamente por um tempo. E isso executa em uma thread diferente, isso poderia acabar alocando o contexto errado nessa thread. E assim por diante. Muito longe do ideal.  

#### Eu estou usando <span style="color:red">GetAwaiter().GetResult()</span>. Eu preciso usar o <span style="color:red">ConfigureAwait(false)?</span>  

No. <span style="color:red">ConfigureAwait</span> apenas afeta o callback. Especificamente, o Padrão "awaiter" demanda awaiters para expor uma propriedade <span style="color:red">IsCompleted</span>, um método <span style="color:red">GetResult</span> e um método <span style="color:red">OnComplete</span> (opcionalmente com um método <span style="color:red">UnsafeOnComplete</span>)). <span style="color:red">ConfigureAwait</span> apenas afeta o comportamento do <span style="color:red">{Unsafe}OnCompleted</span>, então se você está apenas chamando diretamente o método <span style="color:red">GetResult</span> para o awaiter, tanto se você estiver fazendo iso no <span style="color:red">TaskAwaiter</span> ou no <span style="color:red">ConfiguredTaskAwaitable.ConfiguredTaskAwaiter</span> fará zero mudanças diferentes de comportamento. Então se você ver <span style="color:red">Task.ConfigureAwait(false).GetAwaiter().GetResult()</span> no código, você pode substituir iso com <span style="color:red">task.GetAwaiter().GetResult()</span> (e também considere se você realmente quer bloquear dessa forma).  

#### Eu sei, eu estou executando em um ambiente que nunca terá um SynchronizationContext ou um TaskScheduler customizado. Posso deixar de usar o ConfigureAwait(false) ?

Talvez. Isso depende o quando você tem certeza da parte "nunca". Conforme mencionei anteriormente na FAQs, simplesmente porque  o modelo de aplicação que você está trabalhando não configura um <span style="color:red">SynchronizationContext</span> customizado e não invoca seu código em um <span style="color:red">TaskScheduler</span> customizado, não significa que algum outro usuário ou código de biblioteca não o fará. Então você precisa ter certeza de que isso não ocorrerá, ou pelo menos reconhecer o risco se isso ocorrer.  

#### Ouvi dizer que ConfigureAwait(false) não é mais necessário no .Net Core. Verdadeiro?  

Falso. É necessário quando executamos no .NET Core exatamente pelas mesmas razões que isso é necessário quando executamos no .Net Framework. Nada mudou a este respeito.  

O que mudou, entretanto, é se certos ambientes publicam o se próprio <span style="color:red">SynchronizationContext</span>. Em particular, enquanto o ASP.NET clássico no .NET Framework tem [o seu próprio SynchronizationContext](https://github.com/microsoft/referencesource/blob/3b1eaf5203992df69de44c783a3eda37d3d4cd10/System.Web/AspNetSynchronizationContextBase.cs), em contraste o ASP.NET Core não tem. Isso significa que o código que roda em ASP.NET Core por padrão não enxerga um <span style="color:red">SynchronizationContext</span> customizado, o que diminui a necessidade de usar <span style="color:red">ConfigureAwait(false)</span> nesse ambiente.  

Isso não significa, entretanto, que nunca haverá um <span style="color:red">SynchronizationContext</span> ou <span style="color:red">TaskScheduler</span> customizado presente. Se algum código de usuário (ou outro código de biblioteca que seu aplicativo está usando) define um contexto personalizado e chama seu código, ou invoca se codigo em uma Task agendado em um <span style="color:red">TaskScheduler</span> personalizado, então mesmo em um ASP.NET Core seus awaits podem ver um contexto ou shceduler fora do padrão que pode levar você a querer usar o <span style="color:red">ConfigureAwait(false)</span>. Claro que nesse tipo de situação se você evitar bloqueio de sincronismo (o que você deveria evitar fazer em aplicações web, independentemente) e se você nao se importar com pequenas sobrecargas de desempenho nessas poucas ocorrências, você provavelmente pode se livrar de usar <span style="color:red">ConfigureAwait(false)</span>.  

#### Posso usar o <span style="color:red">ConfigureAwait</span> quando estiver aguardando em um foreach um IAsyncEnumerable?

Sim. veja este [artigo da MSDN Magazine](https://docs.microsoft.com/en-us/archive/msdn-magazine/2019/november/csharp-iterating-with-async-enumerables-in-csharp-8) por exemplo.  

<span style="color:red">await foreach</span> vincula a um padrão, e assim, enquanto isso pode ser usado para enumerar um <span style="color:red">IAsyncEnumerable<T></span>, isso também pode ser usado para enumerar algo que exponha a superfície certa da API. As bibliotecas de tempo de execução do .NET incluem um [extension method](https://github.com/dotnet/runtime/blob/91a717450bf5faa44d9295c01f4204dc5010e95c/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/TaskAsyncEnumerableExtensions.cs#L25-L26) <span style="color:red">ConfigureAwait</span> no <span style="color:red">IAsyncEnumerable<T></span> que retorna um tipo personalizado que envelopa o <span style="color:red">IAsyncEnumerable<T></span> e um booleano e expõem o padrão certo. Quando o compilador gera chamadas ao método <span style="color:red">MoveNextAsync</span> e <span style="color:red">DisposeAsync</span> do enumerador, essas chamadas são para o enumerador retornado e configurado como "tipo struct". E isso em turnos executa os awaits configurados do modo desejado.  

#### Posso usar o <span style="color:red">ConfigureAwait</span> quando o 'await' usa um IAsyncDisposable?  

Sim, embora com uma pequena complicação.  

Como com <span style="color:red">IAsyncEnumerable<T></span> descrita na FAQ anterior, as bibliotecas de tempo de execução do .NET expõem um método de extensão <span style="color:red">ConfigureAwait</span> no IAsyncDisposable, e <span style="color:red">await using</span> irá funcionar felizmente com isso, já que isso implementa o padrão de forma apropriada (expecíficamente expondo um método DisposeAsync):
```c#
await using (var c = new MyAsyncDisposableClass().ConfigureAwait(false))
{
    ...
}
```
O problema aqui é q o tipo do <span style="color:red">c</span> é novamente o <span style="color:red">MyAsyncDisposableClass</span> desejado. Isso também tem o efeito de aumentar o escopo do <span style="color:red">c</span>; se isso é impactante, você pode envolver tudo entre chaves. 

#### Eu uso ConfigureAwait(false), mas meu AsyncLocal ainda fluiu para o código após o await. Isso é um bug?  

Não. Isso é esperado. Dados do <span style="color:red">AsyncLocal<T></span> fluem como parte do <span style="color:red">ExecutionContext</span>, o que é separado do <span style="color:red">SynchronizationContext</span>. A não ser que você explicitamente desabilitou o fluxo <span style="color:red">ExecutionContext</span> com <span style="color:red">ExecutionContext.SuppressFlow()</span>, <span style="color:red">ExecutionContext</span> (e assim os dados <span style="color:red">AsynchLocal<T></span>) sempre vão fluir através de awaits, independente de ConfigureAwait ser usado para evitar a captura do SynchronizationContext original. Para obter mais informações consulte esta postagem do [blog](https://devblogs.microsoft.com/pfxteam/executioncontext-vs-synchronizationcontext/).  

#### A linguagem poderia me ajudar a evitar a necessidade de usar o <span style="color:red">ConfigureAwait(false)</span> explicito em minhas bibliotecas?  

Os desenvolvedores de Libs, as vezes, expressam a frustração que sentem com a necessidade de usar o <span style="color:red">ConfigureAwait(false)</span> e pedem por alternativas menos invasivas.  

Atualmente não temos nenhuma, pelo menos nada desenvolvido dentro da linguagem / compilador / runtime. Existem entretanto muitas propostas de como poderia ser essa solução, por exemplo:
https://github.com/dotnet/csharplang/issues/645, 
https://github.com/dotnet/csharplang/issues/2542, 
https://github.com/dotnet/csharplang/issues/2649, and 
https://github.com/dotnet/csharplang/issues/2746.  

Se isso é importante para você, ou se você sente que tem idéias novas e interessantes aqui, eu encorajo você a contribuir com suas ideias para essas ou para novas discussões.
