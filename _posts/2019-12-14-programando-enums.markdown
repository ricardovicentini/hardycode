---
layout: post
title:  "Enums - Usando Enumeradores"
date:   2019-12-14 09:00:00 -0300
categories: Programacao
---
Nesse artigo além do básico de se trabalhar com Enums explico como deixar os enumeradores ainda mais declarativos utilizando DescriptionAttribute  

Um objeto Enum é um tipo de artefato de linguagens orientadas a objetos que permite padronizar valores com um determinado texto, como no exemplo abaixo:
```c#
public enum Perfil
{
    //Posto mais baixo do Exército brasileiro 
    Graduado = 1,
    //É o primeiro nível dos oficiais do exército 
    Subalterno = 2,
    //Nível acima dos subalternos
    Itermediario = 3,
    //São oficiais de nível superior em relação os demais níveis
    Superior = 4,
    //Classe mais alta do exército brasileiro
    General = 5
}
```

Sem enumeradores o código fica muito mais complexo e difícil de entender.  
Veja o exemplo abaixo, o perfil deve ser testado com seus respectivos códigos:
```c#
class Program
{
    private static int perfil;

    static void Main(string[] args)
    {
        //Perfil 1 = Graduado - Posto mais baixo do Exército brasileiro 
        //Perfil 2 = Subalterno - É o primeiro nível dos oficiais do exercíto 
        //Perfil 3 = Intermediario - São oficiais de nível superior em relação os demais níveis
        //Perfil 4 = Superior - São oficiais de nível superior em relação os demais níveis
        //Perfil 5 = General - Classe mais alta do exército brasileiro

        Console.WriteLine("Digite o código de sua patente");
        int.TryParse(Console.ReadLine(),out perfil);

        if (perfil == 1)
        {
            Console.WriteLine("Graduado - Posto mais baixo do Exército brasileiro");
            return;
        }
        if (perfil == 2)
        {
            Console.WriteLine("Subalterno - É o primeiro nível dos oficiais do exercíto ");
            return;
        }
        if (perfil == 3)
        {
            Console.WriteLine("Intermediario - São oficiais de nível superior em relação os demais níveis ");
            return;
        }
        if (perfil == 4)
        {
            Console.WriteLine("Superior - São oficiais de nível superior em relação os demais níveis ");
            return;
        }
        if (perfil == 5)
        {
            Console.WriteLine("General - Classe mais alta do exército brasileiro ");
            return;
        }

        Console.WriteLine("Patente não reconhecida");
        return;


    }

    
}
```
> Se não fosse pelos comentários o desenvolvedor não teria a mínima noção do que seriam esses códigos.  
> E pior, se o desenvolvedor precisar testar o perfil em outros arquivos e classes do programa ele terá que copiar os comentários para guiar futuros programadores que precisarem dar manutenção nesse código.

Agora veja o mesmo código utilizando o enum:

```c#
public enum Perfil
{
    //Posto mais baixo do Exército brasileiro 
    Graduado = 1,
    //É o primeiro nível dos oficiais do Exército 
    Subalterno = 2,
    //Nível acima dos subalternos
    Intermediario = 3,
    //Sáo oficiais de nível superior em relação os demais níveis
    Superior = 4,
    //Classe mais alta do Exército brasileiro
    General = 5
}

class Program
{
    private static Perfil perfil;

    static void Main(string[] args)
    {
        Console.WriteLine("Digite o código de sua patente");
        Enum.TryParse(Console.ReadLine(), out perfil);

        if (perfil == Perfil.Graduado)
        {
            Console.WriteLine("Graduado - Posto mais baixo do Exército brasileiro");
            return;
        }
        if (perfil == Perfil.Subalterno)
        {
            Console.WriteLine("Subalterno - É o primeiro nível dos oficiais do exercíto ");
            return;
        }
        if (perfil == Perfil.Intermediario)
        {
            Console.WriteLine("Intermediario - São oficiais de nível superior em relação os demais níveis ");
            return;
        }
        if (perfil == Perfil.Superior)
        {
            Console.WriteLine("Superior - São oficiais de nível superior em relação os demais níveis ");
            return;
        }
        if (perfil == Perfil.General)
        {
            Console.WriteLine("General - Classe mais alta do exército brasileiro ");
            return;
        }

        Console.WriteLine("Patente não reconhecida");
        return;

    }
```

> O código é praticamente o mesmo, apenas foi substituido nas linhas 17 e 22 a utilização do enum Perfil para fazer a leitura do código de patente digitado.  
> E as linhas que comparam o valor digitado com o perfil, linhas: 24, 29, 34, 39 e 44 que não mais usam números e sim a descrição de cada patente.  

Assim temos o benefício de tornar o código mais fácil de entender para o desenvolvedor que precisar dar manutenção no código.

## A classe Enum
No exemplo anterior utilizamos a classe Enum para converter um número digitado em um objeto do tipo Perfil

```c#
Enum.TryParse(Console.ReadLine(), out perfil);
```

Além dessa possibilidade a classe *Enum*  disponibliza diversas funcionalidades para tratarmos com maior facilidade os enumeradores. Por exemplo:
* GetNames() - Retorna um string array dos enumeradores 
* GetValues() - Retorna um Enum array dos enumeradores 

Esses são os principais, porém há outras funcionalidades que valem a pena serem conhecidas.

Mas com o GetNames() por exemplo é possível listar todos os Enumeradores do enum Perfil:
```c#
private static StringBuilder ListarEnums()
{
    var stringBuilder = new StringBuilder();
    foreach (var perfil in Enum.GetNames(typeof(PatenteV1.Perfil)))
    {
        stringBuilder.AppendLine(perfil);
    }

    return stringBuilder;
}
```

O código completo desse exemplo:  
```c#
public static class PatenteV1
{
    public enum Perfil
    {
        //Posto mais baixo do Exército brasileiro 
        Graduado = 1,
        //É o primeiro nível dos oficiais do Exército 
        Subalterno = 2,
        //Nível acima dos subalternos
        Intermediario = 3,
        //Sáo oficiais de nível superior em relação os demais níveis
        Superior = 4,
        //Classe mais alta do Exército brasileiro
        General = 5
    }
}

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Lista de perfis");
      
        Console.WriteLine(ListarEnums());
    }

    private static StringBuilder ListarEnums()
    {
        var stringBuilder = new StringBuilder();
        foreach (var perfil in Enum.GetNames(typeof(PatenteV1.Perfil)))
        {
            stringBuilder.AppendLine(perfil);
        }

        return stringBuilder;
    }
}
```
**E como resultado temos**

Lista de perfis  
Graduado  
Subalterno  
Intermediario  
Superior  
General  

# Enums com Atributo Description
---
Agora que você já sabe o que é um Enum vamos deixar os Enums ainda mais claros para o desenvolvedor e o código ainda mais simples utilizando DescriptionAttribute.  
No exemplo anterior criamos o enum assim:  
```c#
public enum Perfil
{
    //Posto mais baixo do Exército brasileiro 
    Graduado = 1,
    //É o primeiro nível dos oficiais do Exército 
    Subalterno = 2,
    //Nível acima dos subalternos
    Intermediario = 3,
    //Sáo oficiais de nível superior em relação os demais níveis
    Superior = 4,
    //Classe mais alta do Exército brasileiro
    General = 5
}
```
O problema no código acima é que apesar dos niveis de perfil estarem bem descritos, o significado de cada nível ainda depende de comentário no código, por exemplo:
> Graduado é o nível mais baixo do Exército  
O programdor só sabe disso porque está escrito no comentário  
Se solicitada uma tela para explicar o que é cada patente o programador teria que fazer exatamente como fizemos no exemplo anterior.  
Porém, podemos deixar essas explicações serem parte do código utilizando DescriptionAttribute do *namesapce* ComponentModel, conforme o exemplo abaixo:  


```c#
using System.ComponentModel;
public enum Perfil
{

    [Description("Posto mais baixo do Exército brasileiro ")]
    Graduado = 1,
    [Description("É o primeiro nível dos oficiais do Exército ")]
    Subalterno = 2,
    [Description("Nível acima dos subalternos")]
    Intermediario = 3,
    [Description("Sáo oficiais de nível superior em relação os demais níveis")]
    Superior = 4,
    [Description("Classe mais alta do Exército brasileiro")]
    General = 5
}
```

Note que as explicações deixaram de ser comentário e agora fazem parte do código, e através de uma função ExtensionMethod, podemos estender a classe Enum para receber um novo método chamado descrição, conforme abaixo:

```c#
public static class EnumExtensions
{
    public static string Descricao(this Enum enumRecebido)
    {
        var descriptionAttribute = enumRecebido.GetType()
            .GetMember(enumRecebido.ToString())[0]
            .GetCustomAttribute(typeof(DescriptionAttribute), false) as DescriptionAttribute;

        return descriptionAttribute is null ? enumRecebido.ToString() : descriptionAttribute.Description;
    }
}
```

Com essa nova extension então podemos chamar o método *Descricao* em cada um dos enums, conforme abaixo:

```c#
 static void Main(string[] args)
  {
      Console.WriteLine(ListarDescricoesPerfis());
  }

  private static string ListarDescricoesPerfis()
  {
      var stringBuilder = new StringBuilder();
      foreach (Perfil perfil in Enum.GetValues(typeof(Perfil)))
      {
          stringBuilder.AppendLine($"{perfil.ToString()} - {perfil.Descricao()}");
      }

      return stringBuilder.ToString();
  }
  
```

Na linha 9 obtenho cada um dos perfis do enum Perfil, e na linha 11 concateno a nome do Perfil  "perfil.ToString()" e a descrição "perfil.Descricao()"
e o resultado fica conforme abaixo:  

* Graduado - Posto mais baixo do Exército brasileiro
* Subalterno - É o primeiro nível dos oficiais do Exército
* Intermediario - Nível acima dos subalternos
* Superior - Sáo oficiais de nível superior em relação os demais níveis
* General - Classe mais alta do Exército brasileiro
* Master - Master  

#### Entendendo a Extenision Method **"Descricao()"**
---

```c#
public static class EnumExtensions
{
    public static string Descricao(this Enum enumRecebido)
    {
        var descriptionAttribute = enumRecebido.GetType()
            .GetMember(enumRecebido.ToString())[0]
            .GetCustomAttribute(typeof(DescriptionAttribute), false) as DescriptionAttribute;

        return descriptionAttribute is null ? enumRecebido.ToString() : descriptionAttribute.Description;
    }
}
```

Note que na linha 3, o método descrição será injetado na classe Enum, então quando chamamos no exemplo anterior *"perfil.Descricao()"* o perfil será passado para essa função, na linha 5 através do GetType e GetMember obtemos o DescriptionAttribute e na linha 9 o description atribute do perfil recebido será retornado, e caso no enum não tenha sido declardo o DescritpionAttribute, será retornado o "enumRecebido.ToString()"  

#### Conclusão
---

Trabalhar com Enumeradores, simplifica a manutenção do código, pois torna ele fácil de entendeder!
Utilizar DescriptionAttributes nos enumeradores torna o código ainda mais simples e tira a necessidade de comentar o siginificado de cada Enumerador, fazendo que o significado também seja parte do código fonte.

Espero que tenham gostado!

Baixe os exemplos do meu git em: [https://github.com/ricardovicentini/enums](https://github.com/ricardovicentini/enums)