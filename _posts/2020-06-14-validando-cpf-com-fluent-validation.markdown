---
layout: post
title:  "Validando CPF com FluentValidation."
date:   2020-05-16 12:32:00 -0300
categories: Frameworks
---
FluentValidation é um framework para Dotnet e Dotnetcore que facilita a centralização de validações de dados.  
Este framework possuí várias validações já prontas "Built-in Validators", tais como validação para valores nulos, para valores vazios, para ranges e vários outros.  
Além das validações "Built-in" também é possível construir validações customizadas e injetá-las no comportamento dos Validators através de "Extension Methods".  
Para ver a documentação do FluentValidation [clique aqui](https://docs.fluentvalidation.net/en/latest/index.html)

#### Mão na massa!
Vou criar um exemplo com uma console application, onde tenho a classe pessoa e nesta classe vou validar suas propriedades: Nome, Telefone, Email e Cpf utilizando o FluentValidation.  
A validação de Nome, Telefone e Email, será bem simples e vamos conseguir fazer com "Built-in Validators". Já para Cpf teremos que criar nossa validação customizada.  

Abaixo a definição da classe Pessoa
```c#
public class Pessoa
{
    public string Nome { get; private set; }
    public string Telefone { get; private set; }
    public string Email { get; private set; }
    public string Cpf { get; private set; }
}
```
Depois de adicionada a referência para FluentValidation através do Nuget package, a classe de validação para as propridades fica conforme abaixo:  
```c#
using FluentValidation;
public class PessoaValidator : AbstractValidator<Pessoa>
{
    public PessoaValidator()
    {
        RuleFor(x => x.Nome)
            .NotNull()
            .NotEmpty()
            .MinimumLength(2)
            .MaximumLength(20);

        RuleFor(x => x.Telefone)
            .NotNull()
            .NotEmpty()
            .Matches(@"^\d{8}|d{9}$");

        RuleFor(x => x.Email)
            .NotNull()
            .NotEmpty()
            .EmailAddress();
    }
}
```
É importante observar a herança para **AbstractValidator<<T>>** onde o **<<T>>** deve ser a classe que se deseja validar.  
Assim, no contrutor da classe **PessoaValidator** fica disponível o método **RuleFor**.  
No método **RuleFor** é mapeada a propriedade que se deseja validar linhas 6,12 e 17 e então temos acesso as validações "Built-in". Ex:
* NotNull()
* NotEmpty()
* MinimumLength()
* MaximumLength() e muitas outras...

Não pretendo entrar em detalhes de como elas funcionam, o próprio nome de cada uma já diz, porém recomendo testá-las e ler a documentação também vai ajudar bastante. :relaxed: 

Para o telefone estamos utilizando *Metches* no qual o regex informado será testado para validar este campo. Neste caso o telefone terá de ser 8 ou 9 dígitos numéricos.  

Por enquanto não vou validar o cpf, vou primeiro testar essas validações, para isso é preciso consumir a classe PessoaValidator dentro da  classe Pessoa, então ficou assim:  
```c#
public class Pessoa
{

    public string Nome { get; private set; }
    public string Telefone { get; private set; }
    public string Email { get; private set; }
    public string Cpf { get; private set; }

    readonly PessoaValidator validator = new PessoaValidator();

    public Pessoa(string nome, string telefone, string email, string cpf)
    {
        Nome = nome;
        Telefone = telefone;
        Email = email;
        Cpf = cpf;
        
    }

    public ValidationResult Validar()
    {
        return validator.Validate(this);
        
    }
}
```
Na linha nove crio a instância de PessoaValidator pois assim posso usar essa instância em qualquer parte da classe Pessoa, no exemplo acima está sendo utilizada no método Validar() linha 20.  
O método Validar retorna um **ValidationResult** que é uma estrutura do FluentValidation para armazenar o resultado das regras de validação que criamos na classe PessoaValidator.  
Observe também que adicionei um construtor para passar os valores de Nome, Telefone, Email e Cpf. (linha 11)  
Agora posso testar essas validações por meio da console application conforme abaixo:  
```c#
class Program
{
    static void Main(string[] args)
    {
        var p = new Pessoa(nome: null,telefone: null,email: null,cpf: null);
        var reultado = p.Validar();
        if (!reultado.IsValid)
        {
            foreach (var mensagem in reultado.Errors)
            {
                Console.WriteLine(mensagem);
            }
        }
        

    }
}
```
Estou criando a instância de Pessoa na linha 5, todos os valores nulos, com isso a validação .NotNull() irá gerar uma mensagem conforme imagem abaixo:  
![image-title-here](../../../../assets/FluentValidation1.jpg)  
Veja que temos mensagens para Nome, Telefone e Email.  
Veja também que além da mensagem para Nulo temos também a mensagem para vazio. Isso ocorre porque por padrão as validações são testadas de forma encadeada.
É possível configurar as validações de forma que apenas uma mensagem por vez será apresentada.  
Para isso adicione nas regras de validação a configuração: **.Cascade(CascadeMode.StopOnFirstFailure)**
```c#
public class PessoaValidator : AbstractValidator<Pessoa>
{
    public PessoaValidator()
    {
        RuleFor(x => x.Nome)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .MinimumLength(2)
            .MaximumLength(20);

        RuleFor(x => x.Telefone)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .Matches(@"^\d{8}|d{9}$");

        RuleFor(x => x.Email)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .EmailAddress();
    }
}
```
Nas linhas 6,13 e 19 adicionamos a configuração:
```c#
.Cascade(CascadeMode.StopOnFirstFailure)
```
com isso o resultado fica:  
![image-title-here](../../../../assets/FluentValidation2.jpg)  
Apenas uma validação por Propriedade.  
Agora fica a sugestão para que você altere os valores das propriedades de maneira que cause outras falhas de validação.  
### Validando o CPF
Agora vem a nossa cereja do bolo! :relaxed:  
Tenho que validar o cpf e quero validar da mesma forma fluída, ou seja:
```c#
RuleFor(x => x.Cpf)
    .CpfValido();
```
Como o **FluentValidation** não possuí um "Built-in Validation" para cpf vou criar um "Extension Method" para plugar essa validação, conforme abaixo:  
```c#
public static class CpfValidation
{

    public static IRuleBuilderInitial<T, string> CpfValido<T>(this IRuleBuilder<T, string> ruleBuilder)
    {
        return ruleBuilder.Custom((cpf, context) =>
        {
            //Inspirado na validação de CPF proposta por ElemarJr.
            //https://www.eximiaco.ms/pt/2020/01/10/no-c-8-ficou-mais-facil-alocar-arrays-na-stack-e-isso-pode-ter-um-impacto-positivo-tremendo-na-performance/

            if (string.IsNullOrWhiteSpace(cpf))
            {
                context.AddFailure($"'{context.DisplayName}' não pode ser nulo ou vazio.");
                return;
            }


            Span<int> cpfArray = stackalloc int[11];
            var count = 0;
            foreach (var c in cpf)
            {
                if (!char.IsDigit(c))
                {
                    context.AddFailure($"'{context.DisplayName}' tem que ser numérico.");
                    return;
                }


                if (char.IsDigit(c))
                {
                    if (count > 10)
                    {
                        context.AddFailure($"'{context.DisplayName}' deve possuir 11 caracteres. Foram informados " + cpf.Length);
                        return;
                    }


                    cpfArray[count] = c - '0';
                    count++;
                }
            }

            if (count != 11)
            {
                context.AddFailure($"'{context.DisplayName}' deve possuir 11 caracteres. Foram informados " + cpf.Length);
                return;
            }
            if (VerificarTodosValoresSaoIguais(ref cpfArray))
            {
                context.AddFailure($"'{context.DisplayName}' Não pode conter todos os dígitos iguais.");
                return;
            }

            var totalDigitoI = 0;
            var totalDigitoII = 0;
            int modI;
            int modII;

            for (var posicao = 0; posicao < cpfArray.Length - 2; posicao++)
            {
                totalDigitoI += cpfArray[posicao] * (10 - posicao);
                totalDigitoII += cpfArray[posicao] * (11 - posicao);
            }

            modI = totalDigitoI % 11;
            if (modI < 2) { modI = 0; }
            else { modI = 11 - modI; }

            if (cpfArray[9] != modI)
            {
                context.AddFailure($"'{context.DisplayName}' Inválido.");
                return;
            }

            totalDigitoII += modI * 2;

            modII = totalDigitoII % 11;
            if (modII < 2) { modII = 0; }
            else { modII = 11 - modII; }

            return;

        });
    }

    static bool VerificarTodosValoresSaoIguais(ref Span<int> input)
    {
        for (var i = 1; i < 11; i++)
        {
            if (input[i] != input[0])
            {
                return false;
            }
        }

        return true;
    }
}
```
Não vou entrar no detalhe da validação do CPF em si, para isso recomendo o excelente artigo do  [ElemarJr](https://www.youtube.com/watch?v=CgWTXh66cXc), inclusive o algoritimo de validação do cpf apresentado aqui é uma cópia do algoritimo proposto nesse artigo.  
Note que dizemos para o Fluent que há um problema quando usamos o método **context.AddFailure**.  
Minha estratégia foi: assim que achado um prolbema retorna para a validação, linhas (13,24,33,45 e 55)
```c#
public class PessoaValidator : AbstractValidator<Pessoa>
{
    public PessoaValidator()
    {
        RuleFor(x => x.Nome)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .MinimumLength(2)
            .MaximumLength(20);

        RuleFor(x => x.Telefone)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .Matches(@"^\d{8}|d{9}$");

        RuleFor(x => x.Email)
            .Cascade(CascadeMode.StopOnFirstFailure)
            .NotNull()
            .NotEmpty()
            .EmailAddress();

        RuleFor(x => x.Cpf)
            .CpfValido();
    }
}
```
O fato para mim aqui é que agora eu posso deixar a minha classe de validação fluída também para o Cpf (linhas 24 e 25)  
Veja, também que não é necessário testar NotNull() e NotEmpt() porque minha "Extension Method" já faz esse teste.  
Então se tentar criar uma instância de Pessoa com CPF inválido agora teremos a seguinte saida: 
![image-title-here](../../../../assets/FluentValidation3.jpg)  

### Conclusão
Com FluentValidation temos possiblidade de isolar regras complexas do código e posteriormente testá-las de forma fluída e elegante!
Obrigado por ler! :smiley:  
:floppy_disk: [Baixar código fonte](https://github.com/ricardovicentini/ExemploFluentValidation)