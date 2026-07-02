# Explicação detalhada do `GlobalExceptionHandlingMiddleware`

## 1. Objetivo do código

Este arquivo cria um **middleware global de tratamento de exceções** para uma API ASP.NET Core.

Um middleware é um componente que participa do fluxo de todas as requisições HTTP. Ele pode:

1. receber a requisição;
2. executar alguma lógica antes dos demais componentes;
3. encaminhar a requisição para o próximo middleware;
4. executar alguma lógica depois que os componentes seguintes terminarem;
5. capturar erros lançados pelos componentes que estão abaixo dele no pipeline.

Neste caso, o middleware deixa a requisição continuar normalmente. Porém, se algum controller, serviço, caso de uso, repositório ou outro middleware lançar uma exceção não tratada, ele:

- captura a exceção;
- escolhe o status HTTP apropriado;
- cria um objeto `ProblemDetails`;
- registra a ocorrência nos logs;
- devolve um JSON padronizado ao cliente.

Sem esse middleware, cada controller poderia precisar de vários blocos `try/catch`, causando duplicação de código e respostas de erro inconsistentes.

---

## 2. Código completo numerado

```csharp
01 using System.Diagnostics;
02 using System.Text.Encodings.Web;
03 using System.Text.Json;
04 using Microsoft.AspNetCore.Mvc;
05 using MoodleProvisioner.Api.Modelos;
06
07 namespace MoodleProvisioner.Api.Middleware;
08
09 public sealed class GlobalExceptionHandlingMiddleware(
10     RequestDelegate next,
11     ILogger<GlobalExceptionHandlingMiddleware> logger)
12 {
13     private static readonly JsonSerializerOptions ProblemJsonOptions = new(JsonSerializerOptions.Web)
14     {
15         Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
16     };
17
18     // Este método é chamado pelo ASP.NET Core para cada requisição HTTP.
19     // O middleware tenta continuar o fluxo normal da API. Se qualquer camada abaixo
20     // dele lançar uma exceção, o catch transforma essa exceção em uma resposta HTTP padronizada.
21     public async Task InvokeAsync(HttpContext context)
22     {
23         try
24         {
25             await next(context);
26         }
27         catch (Exception error)
28         {
29             // Se a resposta já começou a ser enviada, não é mais seguro trocar o status
30             // ou escrever outro corpo JSON. Nessa situação, registramos o erro e deixamos
31             // o ASP.NET Core continuar o tratamento padrão.
32             if (context.Response.HasStarted)
33             {
34                 logger.LogError(error, "Uma exceção ocorreu depois que a resposta já havia começado.");
35                 throw;
36             }
37
38             await WriteProblemDetailsAsync(context, error);
39         }
40     }
41
42     // Converte a exceção capturada em ProblemDetails e escreve a resposta HTTP.
43     // ProblemDetails é um formato comum no ASP.NET Core para retornar erros de API
44     // com campos como status, title, detail e instance.
45     private async Task WriteProblemDetailsAsync(HttpContext context, Exception error)
46     {
47         var problem = CreateProblemDetails(context, error);
48
49         // Erros 5xx indicam falha interna e merecem log como erro.
50         // Erros 4xx são falhas esperadas de entrada/regra de negócio e ficam como aviso.
51         if (problem.Status >= StatusCodes.Status500InternalServerError)
52         {
53             logger.LogError(error, "Uma exceção não tratada ocorreu.");
54         }
55         else
56         {
57             logger.LogWarning(error, "Uma exceção de aplicação foi tratada.");
58         }
59
60         // Limpa qualquer conteúdo parcial, define o status HTTP e informa que o corpo
61         // segue o padrão application/problem+json.
62         context.Response.Clear();
63         context.Response.StatusCode = problem.Status ?? StatusCodes.Status500InternalServerError;
64         context.Response.ContentType = "application/problem+json";
65
66         // Serializa manualmente para manter o Content-Type definido acima.
67         // WriteAsJsonAsync poderia trocar o tipo de conteúdo para application/json.
68         await JsonSerializer.SerializeAsync(
69             context.Response.Body,
70             problem,
71             ProblemJsonOptions,
72             context.RequestAborted);
73     }
74
75     // Decide qual resposta será enviada para cada tipo de exceção.
76     // O switch abaixo deixa as regras em um único lugar: validação vira 400,
77     // duplicidade vira 409, recurso não encontrado vira 404 e falhas internas viram 500.
78     private static ProblemDetails CreateProblemDetails(HttpContext context, Exception error)
79     {
80         var problem = error switch
81         {
82             OperationCanceledException => CreateProblem(
83                 context,
84                 StatusCodes.Status499ClientClosedRequest,
85                 "Requisição cancelada.",
86                 "A requisição foi cancelada antes de ser concluída."),
87
88             FalhaValidacao validacao => CreateProblem(
89                 context,
90                 StatusCodes.Status400BadRequest,
91                 validacao.Message,
92                 "A requisição contém dados inválidos.",
93                 ("details", validacao.Erros)),
94
95             ArgumentException argument => CreateProblem(
96                 context,
97                 StatusCodes.Status400BadRequest,
98                 "Argumento inválido.",
99                 argument.Message),
100
101            UnauthorizedAccessException unauthorizedAccess => CreateProblem(
102                context,
103                StatusCodes.Status403Forbidden,
104                "Acesso negado.",
105                unauthorizedAccess.Message),
106
107            KeyNotFoundException keyNotFound => CreateProblem(
108                context,
109                StatusCodes.Status404NotFound,
110                "Recurso não encontrado.",
111                keyNotFound.Message),
112
113            FileNotFoundException fileNotFound => CreateProblem(
114                context,
115                StatusCodes.Status404NotFound,
116                "Arquivo não encontrado.",
117                fileNotFound.Message),
118
119            InstituicaoJaExiste instituicaoJaExiste => CreateProblem(
120                context,
121                StatusCodes.Status409Conflict,
122                instituicaoJaExiste.Message,
123                "Já existe uma instituição com os mesmos dados de provisionamento.",
124                ("matches", instituicaoJaExiste.Ocorrencias)),
125
126            InvalidOperationException invalidOperation => CreateProblem(
127                context,
128                StatusCodes.Status409Conflict,
129                "Operação inválida para o estado atual.",
130                invalidOperation.Message),
131
132            TimeoutException timeout => CreateProblem(
133                context,
134                StatusCodes.Status504GatewayTimeout,
135                "Tempo limite excedido.",
136                timeout.Message),
137
138            FalhaProvisionamento falhaProvisionamento => CreateProblem(
139                context,
140                StatusCodes.Status500InternalServerError,
141                "Falha ao provisionar a instituição.",
142                falhaProvisionamento.Message),
143
144            _ => CreateProblem(
145                context,
146                StatusCodes.Status500InternalServerError,
147                "Erro inesperado.",
148                "Ocorreu um erro inesperado ao processar a requisição.")
149        };
150
151        // traceId ajuda a relacionar a resposta recebida pelo cliente com os logs da API.
152        problem.Extensions["traceId"] = Activity.Current?.Id ?? context.TraceIdentifier;
153        return problem;
154    }
155
156    // Monta o objeto ProblemDetails usado como corpo da resposta.
157    // As extensões permitem adicionar campos extras, como details ou matches,
158    // sem criar uma classe diferente para cada tipo de erro.
159    private static ProblemDetails CreateProblem(
160        HttpContext context,
161        int statusCode,
162        string title,
163        string detail,
164        params (string Key, object Value)[] extensions)
165    {
166        var problem = new ProblemDetails
167        {
168            Status = statusCode,
169            Title = title,
170            Detail = detail,
171            Instance = context.Request.Path
172        };
173
174        // Copia cada extensão recebida para o JSON final.
175        foreach (var extension in extensions)
176        {
177            problem.Extensions[extension.Key] = extension.Value;
178        }
179
180        return problem;
181    }
182 }
```

> Os números foram adicionados somente para esta explicação. Eles não fazem parte do arquivo C# original.

---

# 3. Explicação linha a linha

## Linhas 1 a 5 — Importações com `using`

### Linha 1

```csharp
using System.Diagnostics;
```

A palavra-chave `using` importa um namespace, permitindo utilizar seus tipos sem escrever o nome completo deles.

Essa linha importa o namespace `System.Diagnostics`. Neste arquivo, ele é necessário por causa de:

```csharp
Activity.Current
```

`Activity` representa uma atividade de diagnóstico. Em aplicações web, ela pode representar o rastreamento de uma requisição. Seu identificador ajuda a relacionar:

- a resposta recebida pelo cliente;
- os logs produzidos pela aplicação;
- informações de tracing distribuído;
- chamadas entre diferentes serviços.

Sem o `using`, seria possível escrever:

```csharp
System.Diagnostics.Activity.Current
```

O `using` apenas evita ter de repetir o namespace completo.

---

### Linha 2

```csharp
using System.Text.Encodings.Web;
```

Importa tipos relacionados à codificação de texto para a web.

O tipo utilizado neste arquivo é:

```csharp
JavaScriptEncoder
```

Ele será configurado nas opções de serialização JSON.

---

### Linha 3

```csharp
using System.Text.Json;
```

Importa a biblioteca nativa de JSON do .NET.

Este arquivo usa dois tipos desse namespace:

```csharp
JsonSerializerOptions
JsonSerializer
```

- `JsonSerializerOptions` configura como os objetos serão convertidos para JSON.
- `JsonSerializer` realiza a conversão de objetos C# em JSON e de JSON em objetos C#.

---

### Linha 4

```csharp
using Microsoft.AspNetCore.Mvc;
```

Importa recursos do ASP.NET Core MVC.

O tipo utilizado é:

```csharp
ProblemDetails
```

`ProblemDetails` é uma classe destinada a representar erros HTTP de maneira padronizada.

Um objeto desse tipo geralmente produz um JSON semelhante a:

```json
{
  "status": 404,
  "title": "Recurso não encontrado.",
  "detail": "A instituição 15 não foi encontrada.",
  "instance": "/api/instituicoes/15",
  "traceId": "00-abc123..."
}
```

---

### Linha 5

```csharp
using MoodleProvisioner.Api.Modelos;
```

Importa um namespace pertencente ao próprio projeto.

Esse namespace provavelmente contém as exceções personalizadas:

```csharp
FalhaValidacao
InstituicaoJaExiste
FalhaProvisionamento
```

Essas classes não são exceções padrão do .NET. Elas foram criadas especificamente para representar problemas do domínio ou da aplicação.

Para que possam ser capturadas pelo `catch (Exception)`, normalmente elas devem herdar, direta ou indiretamente, de `Exception`.

Exemplo simplificado:

```csharp
public sealed class FalhaValidacao : Exception
{
    public IReadOnlyCollection<string> Erros { get; }

    public FalhaValidacao(
        string mensagem,
        IReadOnlyCollection<string> erros) : base(mensagem)
    {
        Erros = erros;
    }
}
```

---

## Linha 7 — Namespace do arquivo

```csharp
namespace MoodleProvisioner.Api.Middleware;
```

O namespace organiza as classes do projeto.

A classe ficará identificada pelo nome completo:

```csharp
MoodleProvisioner.Api.Middleware.GlobalExceptionHandlingMiddleware
```

Este projeto usa a sintaxe de **namespace com escopo de arquivo**, introduzida no C# 10.

Na sintaxe antiga, seria necessário envolver toda a classe com chaves:

```csharp
namespace MoodleProvisioner.Api.Middleware
{
    public class GlobalExceptionHandlingMiddleware
    {
    }
}
```

Na sintaxe moderna:

```csharp
namespace MoodleProvisioner.Api.Middleware;
```

Tudo o que aparece depois dessa linha no arquivo pertence ao namespace informado.

---

## Linhas 9 a 12 — Declaração da classe e injeção de dependências

### Linha 9

```csharp
public sealed class GlobalExceptionHandlingMiddleware(
```

Esta linha contém quatro conceitos.

### `public`

Define a acessibilidade da classe.

`public` significa que a classe pode ser acessada por outras partes da aplicação e por outros projetos que referenciem este projeto.

### `sealed`

Significa que a classe não pode ser herdada.

Portanto, isto não será permitido:

```csharp
public class MeuMiddleware : GlobalExceptionHandlingMiddleware
{
}
```

Usar `sealed` pode ser apropriado quando:

- a classe não foi projetada para extensão;
- não existe necessidade de sobrescrever seus comportamentos;
- deseja-se comunicar claramente que aquela implementação é final.

### `class`

Indica que está sendo declarada uma classe.

Uma classe é um tipo de referência que pode conter:

- campos;
- propriedades;
- métodos;
- construtores;
- eventos;
- outros tipos internos.

### Construtor primário

O trecho que começa com parênteses após o nome da classe:

```csharp
GlobalExceptionHandlingMiddleware(
    RequestDelegate next,
    ILogger<GlobalExceptionHandlingMiddleware> logger)
```

é um **construtor primário de classe**, recurso disponível a partir do C# 12.

Em uma forma mais tradicional, a classe poderia ser escrita assim:

```csharp
public sealed class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;

    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
}
```

Com o construtor primário, os parâmetros `next` e `logger` podem ser utilizados diretamente no corpo da classe.

---

### Linha 10

```csharp
RequestDelegate next,
```

`RequestDelegate` representa uma função capaz de processar uma requisição HTTP.

De forma conceitual, ele se parece com:

```csharp
delegate Task RequestDelegate(HttpContext context);
```

O parâmetro `next` representa o **próximo componente do pipeline**.

Imagine a sequência:

```text
Requisição
   ↓
Middleware de exceção
   ↓
Middleware de autenticação
   ↓
Middleware de autorização
   ↓
Controller
```

Dentro do middleware de exceção, chamar:

```csharp
await next(context);
```

significa:

> “Passe esta requisição para o próximo componente.”

O nome `next` é uma convenção. Poderia ser outro nome, mas `next` deixa clara sua finalidade.

---

### Linha 11

```csharp
ILogger<GlobalExceptionHandlingMiddleware> logger)
```

`ILogger<T>` é a abstração de logs fornecida pelo .NET.

O tipo genérico:

```csharp
ILogger<GlobalExceptionHandlingMiddleware>
```

informa que os logs pertencem à categoria dessa classe.

Isso ajuda ferramentas de logging a mostrar de onde o registro veio.

Exemplo conceitual de log:

```text
fail: MoodleProvisioner.Api.Middleware.GlobalExceptionHandlingMiddleware
      Uma exceção não tratada ocorreu.
```

O ASP.NET Core fornece `RequestDelegate` e `ILogger<T>` por meio de **injeção de dependência**.

Você não precisa criar essas dependências manualmente ao instanciar o middleware.

---

### Linha 12

```csharp
{
```

Abre o corpo da classe.

Tudo até a chave correspondente da linha 182 faz parte de `GlobalExceptionHandlingMiddleware`.

---

## Linhas 13 a 16 — Configuração do serializador JSON

### Linha 13

```csharp
private static readonly JsonSerializerOptions ProblemJsonOptions = new(JsonSerializerOptions.Web)
```

Essa linha declara e inicializa um campo.

Vamos dividir a declaração.

### `private`

O campo só pode ser acessado dentro da própria classe.

Outro componente não pode fazer:

```csharp
GlobalExceptionHandlingMiddleware.ProblemJsonOptions
```

### `static`

O campo pertence à classe, e não a uma instância específica.

Sem `static`, cada objeto do middleware teria sua própria cópia das opções.

Com `static`, existe uma única configuração compartilhada.

Isso é adequado porque:

- as opções são as mesmas para todas as requisições;
- não há necessidade de recriá-las;
- `JsonSerializerOptions` pode ser reutilizado depois de configurado.

### `readonly`

Impede que o campo receba outra instância depois da inicialização.

Isto não será permitido em outro método:

```csharp
ProblemJsonOptions = new JsonSerializerOptions();
```

`readonly` não significa que o objeto interno seja magicamente imutável. Significa que a referência armazenada no campo não pode ser substituída depois da inicialização permitida.

### `JsonSerializerOptions`

É o tipo do campo. Ele contém as regras de serialização.

### `ProblemJsonOptions`

É o nome do campo.

A convenção comum do .NET usa PascalCase para campos `static readonly`.

### `new(JsonSerializerOptions.Web)`

Cria uma nova instância utilizando outra configuração como base.

A forma completa seria:

```csharp
new JsonSerializerOptions(JsonSerializerOptions.Web)
```

A expressão usa **target-typed `new`**. Como o compilador já sabe que o tipo esperado é `JsonSerializerOptions`, não é necessário repeti-lo depois de `new`.

`JsonSerializerOptions.Web` fornece padrões adequados para APIs web, como propriedades JSON usando `camelCase`.

Exemplo:

```csharp
public string TraceIdentifier { get; set; }
```

poderia ser serializado como:

```json
{
  "traceIdentifier": "..."
}
```

---

### Linha 14

```csharp
{
```

Inicia um inicializador de objeto.

O objeto foi criado na linha 13 e suas propriedades serão configuradas entre as chaves.

---

### Linha 15

```csharp
Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
```

Configura o codificador utilizado na produção do JSON.

Por padrão, o serializador pode escapar determinados caracteres, produzindo sequências como:

```json
"\u00E7"
```

em vez de:

```json
"ç"
```

`UnsafeRelaxedJsonEscaping` permite que uma quantidade maior de caracteres seja escrita diretamente.

Isso torna mensagens em português mais legíveis no corpo bruto da resposta.

Por exemplo:

```json
{
  "title": "Requisição inválida"
}
```

em vez de uma versão com vários escapes Unicode.

O nome contém `Unsafe` porque essa configuração é menos restritiva. O JSON deve continuar sendo enviado com o tipo de conteúdo correto e não deve ser inserido diretamente, sem tratamento, dentro de HTML ou JavaScript.

Neste código, ele é usado para uma resposta JSON de API:

```text
application/problem+json
```

---

### Linha 16

```csharp
};
```

Fecha o inicializador do objeto e encerra a declaração do campo com `;`.

---

## Linhas 18 a 20 — Comentário sobre o método principal

```csharp
// Este método é chamado pelo ASP.NET Core para cada requisição HTTP.
// O middleware tenta continuar o fluxo normal da API. Se qualquer camada abaixo
// dele lançar uma exceção, o catch transforma essa exceção em uma resposta HTTP padronizada.
```

Linhas iniciadas por `//` são comentários.

O compilador ignora esses textos. Eles existem para explicar a intenção do código aos desenvolvedores.

Os comentários descrevem corretamente o papel do método `InvokeAsync`.

---

## Linha 21 — Assinatura de `InvokeAsync`

```csharp
public async Task InvokeAsync(HttpContext context)
```

Essa é a assinatura do método executado pelo ASP.NET Core.

### `public`

O método precisa ser acessível ao framework.

### `async`

Informa que o método contém operações assíncronas e pode usar `await`.

Operações web frequentemente são assíncronas porque podem aguardar:

- banco de dados;
- acesso a arquivos;
- chamadas HTTP;
- escrita da resposta;
- outros middlewares.

Enquanto aguarda uma operação de entrada e saída, a thread pode ser utilizada para atender outra requisição.

### `Task`

É o tipo de retorno do método assíncrono.

Como o método não devolve um valor de negócio, ele retorna `Task`, e não `Task<T>`.

Exemplo de método que devolve um valor:

```csharp
public async Task<int> ObterQuantidadeAsync()
```

### `InvokeAsync`

É o nome convencional que o ASP.NET Core procura em um middleware baseado em classe.

### `HttpContext context`

`HttpContext` representa o contexto completo da requisição HTTP atual.

Ele contém, entre outras informações:

```csharp
context.Request
context.Response
context.User
context.TraceIdentifier
context.RequestAborted
```

- `Request`: dados da requisição recebida.
- `Response`: resposta que será devolvida.
- `User`: usuário autenticado.
- `TraceIdentifier`: identificador da requisição.
- `RequestAborted`: token cancelado quando o cliente abandona a requisição.

---

## Linha 22

```csharp
{
```

Abre o corpo de `InvokeAsync`.

---

## Linhas 23 a 26 — Execução normal do pipeline

### Linha 23

```csharp
try
```

Inicia um bloco de código monitorado.

Se uma exceção ocorrer dentro do `try`, o fluxo pode ser direcionado ao `catch`.

---

### Linha 24

```csharp
{
```

Abre o bloco `try`.

---

### Linha 25

```csharp
await next(context);
```

Esta é a linha central do middleware.

Ela chama o próximo componente do pipeline, passando o mesmo `HttpContext`.

O `await` faz o método aguardar a conclusão de todo o pipeline abaixo dele.

Suponha este pipeline:

```text
GlobalExceptionHandlingMiddleware
    → AuthenticationMiddleware
        → AuthorizationMiddleware
            → Controller
                → Serviço
                    → Repositório
```

Quando o middleware executa:

```csharp
await next(context);
```

ele entra no restante do pipeline.

Se o repositório lançar uma exceção e ninguém abaixo tratá-la, ela sobe pela pilha:

```text
Repositório
    ↑ Serviço
        ↑ Controller
            ↑ outros middlewares
                ↑ GlobalExceptionHandlingMiddleware
```

Como a chamada está dentro de `try`, o middleware global pode capturá-la.

O código depois de `await next(context)` seria executado quando a resposta estivesse retornando pelo pipeline. Neste caso, não há nenhuma instrução depois dessa linha dentro do `try`.

---

### Linha 26

```csharp
}
```

Fecha o bloco `try`.

---

## Linhas 27 a 39 — Captura da exceção

### Linha 27

```csharp
catch (Exception error)
```

Captura qualquer exceção que herde de `Exception`.

- `Exception` é o tipo base da maioria das exceções do .NET.
- `error` é a variável que recebe o objeto da exceção capturada.

Esse objeto contém informações como:

```csharp
error.Message
error.StackTrace
error.InnerException
error.GetType()
```

Como o objetivo é um manipulador global, capturar `Exception` aqui é aceitável. Em código de negócio comum, um `catch (Exception)` vazio ou genérico pode esconder erros; neste middleware, a exceção será registrada e convertida em resposta.

---

### Linha 28

```csharp
{
```

Abre o bloco executado quando uma exceção é capturada.

---

### Linhas 29 a 31

Esses comentários explicam um ponto importante do protocolo HTTP.

Depois que os cabeçalhos da resposta são enviados ao cliente, não é seguro tentar alterar:

- o status HTTP;
- o `Content-Type`;
- outros cabeçalhos;
- todo o formato do corpo.

---

### Linha 32

```csharp
if (context.Response.HasStarted)
```

Verifica se a resposta HTTP já começou a ser enviada.

`HasStarted` é um valor booleano:

```csharp
true
false
```

Se for `true`, parte da resposta já pode estar no cliente.

Exemplo:

1. um endpoint começa a transmitir um arquivo;
2. os cabeçalhos e alguns bytes são enviados;
3. ocorre uma exceção;
4. o middleware tenta substituir tudo por um JSON.

Nesse momento, a substituição completa já não é confiável.

---

### Linha 33

```csharp
{
```

Abre o bloco do `if`.

---

### Linha 34

```csharp
logger.LogError(error, "Uma exceção ocorreu depois que a resposta já havia começado.");
```

Registra a exceção com nível `Error`.

O primeiro argumento é o objeto da exceção:

```csharp
error
```

Ao passá-lo, o provedor de logs pode registrar:

- tipo da exceção;
- mensagem;
- stack trace;
- exceções internas.

O segundo argumento é a mensagem descritiva do log.

Isso é melhor do que registrar somente:

```csharp
logger.LogError(error.Message);
```

porque somente a mensagem perderia o stack trace.

---

### Linha 35

```csharp
throw;
```

Relança a mesma exceção.

Usar apenas `throw;` preserva o stack trace original.

Evite substituir por:

```csharp
throw error;
```

porque isso pode alterar o ponto apresentado como origem do relançamento e prejudicar o diagnóstico.

A exceção é relançada porque o middleware já não consegue criar com segurança uma nova resposta completa. Assim, o servidor continua seu tratamento padrão para esse cenário.

---

### Linha 36

```csharp
}
```

Fecha o bloco do `if`.

---

### Linha 38

```csharp
await WriteProblemDetailsAsync(context, error);
```

Se a resposta ainda não começou, o middleware chama um método auxiliar para:

1. classificar a exceção;
2. criar o `ProblemDetails`;
3. registrar o log;
4. limpar a resposta;
5. definir o status;
6. serializar o JSON.

O método é assíncrono porque escrever no corpo da resposta é uma operação assíncrona.

---

### Linha 39

```csharp
}
```

Fecha o `catch`.

---

### Linha 40

```csharp
}
```

Fecha o método `InvokeAsync`.

---

## Linhas 42 a 45 — Método que grava a resposta de erro

Os comentários das linhas 42 a 44 explicam a responsabilidade do método.

### Linha 45

```csharp
private async Task WriteProblemDetailsAsync(HttpContext context, Exception error)
```

### `private`

O método só é usado internamente pela classe.

### `async Task`

É um método assíncrono sem valor de retorno.

### Parâmetros

```csharp
HttpContext context
Exception error
```

Ele recebe:

- o contexto da requisição;
- a exceção capturada.

---

### Linha 46

```csharp
{
```

Abre o corpo do método.

---

### Linha 47

```csharp
var problem = CreateProblemDetails(context, error);
```

Chama outro método para transformar a exceção em `ProblemDetails`.

### `var`

Pede ao compilador para inferir o tipo da variável.

Como `CreateProblemDetails` retorna `ProblemDetails`, isto:

```csharp
var problem = ...
```

é equivalente a:

```csharp
ProblemDetails problem = ...
```

`var` não torna a variável dinâmica. O tipo continua sendo definido em tempo de compilação.

A partir dessa linha, `problem` contém dados como:

```csharp
problem.Status
problem.Title
problem.Detail
problem.Instance
problem.Extensions
```

---

## Linhas 49 a 58 — Escolha do nível de log

### Linha 51

```csharp
if (problem.Status >= StatusCodes.Status500InternalServerError)
```

Verifica se o status é maior ou igual a 500.

`StatusCodes.Status500InternalServerError` é uma constante inteira cujo valor é `500`.

A condição equivale conceitualmente a:

```csharp
if (problem.Status >= 500)
```

Utilizar a constante deixa o código mais legível.

A propriedade `Status` de `ProblemDetails` é `int?`, ou seja, aceita:

- um número inteiro;
- `null`.

A comparação com um `int?` produz uma condição compatível com o `if`; neste código, `Status` sempre é preenchido por `CreateProblem`, mas o tipo da classe ainda é anulável.

---

### Linha 52

```csharp
{
```

Abre o bloco para erros 5xx.

---

### Linha 53

```csharp
logger.LogError(error, "Uma exceção não tratada ocorreu.");
```

Registra falhas internas com nível `Error`.

Erros 5xx normalmente indicam:

- erro inesperado no servidor;
- indisponibilidade interna;
- falha no provisionamento;
- timeout de dependência;
- defeito que pode exigir investigação.

---

### Linha 54

```csharp
}
```

Fecha o bloco do `if`.

---

### Linha 55

```csharp
else
```

Executa o bloco seguinte quando o status é menor que 500.

---

### Linha 56

```csharp
{
```

Abre o bloco do `else`.

---

### Linha 57

```csharp
logger.LogWarning(error, "Uma exceção de aplicação foi tratada.");
```

Registra erros 4xx como `Warning`.

Esses erros muitas vezes representam situações esperadas, por exemplo:

- dados inválidos;
- recurso não encontrado;
- conflito de estado;
- tentativa de cadastrar algo duplicado.

Eles ainda merecem observação, mas normalmente não indicam que a API está quebrada.

Uma equipe pode optar por usar `Information` em alguns erros 4xx muito comuns. A escolha depende da política de observabilidade do projeto.

---

### Linha 58

```csharp
}
```

Fecha o `else`.

---

## Linhas 60 a 64 — Preparação da resposta HTTP

### Linha 62

```csharp
context.Response.Clear();
```

Limpa dados parciais da resposta que ainda não foram enviados.

Isso pode remover:

- corpo ainda não enviado;
- cabeçalhos configurados;
- status definido anteriormente.

A chamada só é feita porque `InvokeAsync` já verificou que:

```csharp
context.Response.HasStarted == false
```

---

### Linha 63

```csharp
context.Response.StatusCode = problem.Status ?? StatusCodes.Status500InternalServerError;
```

Define o status HTTP da resposta.

O operador:

```csharp
??
```

é chamado de **operador de coalescência nula**.

Ele significa:

> Use o valor à esquerda, a menos que ele seja `null`; se for `null`, use o valor à direita.

Portanto:

```csharp
problem.Status ?? StatusCodes.Status500InternalServerError
```

significa:

- use `problem.Status`, se existir;
- caso contrário, use `500`.

Neste código, `CreateProblem` sempre preenche `Status`, mas o fallback torna o método mais defensivo.

---

### Linha 64

```csharp
context.Response.ContentType = "application/problem+json";
```

Define o tipo do conteúdo da resposta.

`application/problem+json` informa que:

- o corpo está em JSON;
- o JSON descreve um problema HTTP;
- a estrutura segue o padrão de `ProblemDetails`.

Isso é mais específico do que:

```text
application/json
```

Um cliente pode usar o `Content-Type` para decidir como interpretar a resposta.

---

## Linhas 66 a 72 — Serialização do `ProblemDetails`

### Linha 68

```csharp
await JsonSerializer.SerializeAsync(
```

Inicia uma chamada assíncrona ao serializador JSON.

O método converterá o objeto `problem` em bytes JSON e os escreverá diretamente no corpo da resposta.

---

### Linha 69

```csharp
context.Response.Body,
```

É o destino da serialização.

`Response.Body` é um `Stream`, isto é, um fluxo de bytes enviado ao cliente.

---

### Linha 70

```csharp
problem,
```

É o objeto que será serializado.

---

### Linha 71

```csharp
ProblemJsonOptions,
```

São as opções de serialização criadas nas linhas 13 a 16.

Elas controlam a forma do JSON e o codificador usado.

---

### Linha 72

```csharp
context.RequestAborted);
```

Passa um `CancellationToken`.

`context.RequestAborted` é cancelado quando, por exemplo:

- o usuário fecha a conexão;
- o cliente cancela a chamada;
- um proxy encerra a requisição.

Ao passar esse token, a escrita pode ser interrompida em vez de continuar consumindo recursos para uma conexão que já não existe.

O `);` fecha a chamada ao método.

---

### Linha 73

```csharp
}
```

Fecha `WriteProblemDetailsAsync`.

---

# 4. O `switch` que transforma exceções em respostas HTTP

## Linhas 75 a 78 — Responsabilidade do método

### Linha 78

```csharp
private static ProblemDetails CreateProblemDetails(HttpContext context, Exception error)
```

Esse método recebe uma exceção e devolve um `ProblemDetails`.

### `private`

Só pode ser chamado pela própria classe.

### `static`

Não depende de uma instância do middleware.

Ele utiliza somente:

- os parâmetros recebidos;
- outros métodos estáticos.

Como não usa `logger` nem `next`, pode ser estático.

### Tipo de retorno

```csharp
ProblemDetails
```

O método sempre devolve um objeto desse tipo.

---

### Linha 79

```csharp
{
```

Abre o corpo do método.

---

### Linha 80

```csharp
var problem = error switch
```

Inicia uma **switch expression** baseada no objeto `error`.

Diferentemente do `switch` tradicional:

```csharp
switch (error)
{
    case ArgumentException argument:
        ...
        break;
}
```

a switch expression produz diretamente um valor:

```csharp
var problem = error switch
{
    ...
};
```

Cada braço examina o tipo da exceção e retorna um `ProblemDetails`.

A ordem importa. O primeiro padrão compatível é escolhido.

---

### Linha 81

```csharp
{
```

Abre a switch expression.

---

## Linhas 82 a 86 — `OperationCanceledException`

```csharp
OperationCanceledException => CreateProblem(
    context,
    StatusCodes.Status499ClientClosedRequest,
    "Requisição cancelada.",
    "A requisição foi cancelada antes de ser concluída."),
```

### Padrão

```csharp
OperationCanceledException
```

Corresponde a uma exceção de cancelamento.

Ela pode acontecer quando:

- o cliente fecha a conexão;
- um `CancellationToken` é cancelado;
- uma operação é interrompida antes da conclusão.

### Status 499

```csharp
StatusCodes.Status499ClientClosedRequest
```

Representa o status `499`, usado como convenção para indicar que o cliente encerrou a requisição.

O código 499 não pertence ao conjunto original de códigos padronizados pelo HTTP da mesma maneira que 400, 404 ou 500. Ele é uma convenção bastante utilizada para “Client Closed Request”.

### `CreateProblem`

O método auxiliar recebe:

1. `context`;
2. status `499`;
3. título;
4. detalhe.

Nenhuma extensão adicional é enviada neste caso.

---

## Linhas 88 a 93 — `FalhaValidacao`

```csharp
FalhaValidacao validacao => CreateProblem(
    context,
    StatusCodes.Status400BadRequest,
    validacao.Message,
    "A requisição contém dados inválidos.",
    ("details", validacao.Erros)),
```

### Pattern matching com variável

```csharp
FalhaValidacao validacao
```

Além de verificar o tipo, o código cria a variável `validacao`.

Ela possui o tipo `FalhaValidacao`, permitindo acessar membros específicos:

```csharp
validacao.Message
validacao.Erros
```

### Status 400

`400 Bad Request` informa que a requisição possui dados inválidos.

### Título

```csharp
validacao.Message
```

A mensagem da própria exceção será usada como título.

Por exemplo:

```text
Falha na validação dos dados da instituição.
```

### Detalhe

```csharp
"A requisição contém dados inválidos."
```

É uma descrição geral do problema.

### Extensão `details`

```csharp
("details", validacao.Erros)
```

Essa sintaxe cria uma tupla com dois elementos:

```csharp
(string, object)
```

Os nomes esperados pelo método são:

```csharp
(string Key, object Value)
```

A chave será:

```text
details
```

O valor será:

```csharp
validacao.Erros
```

Exemplo de JSON:

```json
{
  "status": 400,
  "title": "Falha na validação.",
  "detail": "A requisição contém dados inválidos.",
  "instance": "/api/instituicoes",
  "details": [
    "O CNPJ é obrigatório.",
    "O domínio informado é inválido."
  ],
  "traceId": "00-abc..."
}
```

---

## Linhas 95 a 99 — `ArgumentException`

```csharp
ArgumentException argument => CreateProblem(
    context,
    StatusCodes.Status400BadRequest,
    "Argumento inválido.",
    argument.Message),
```

`ArgumentException` é uma exceção padrão do .NET utilizada quando um argumento recebido por um método é inválido.

Exemplo:

```csharp
if (string.IsNullOrWhiteSpace(nome))
{
    throw new ArgumentException("O nome não pode ser vazio.", nameof(nome));
}
```

O middleware responde com:

- status `400`;
- título fixo;
- mensagem da exceção como detalhe.

Possível JSON:

```json
{
  "status": 400,
  "title": "Argumento inválido.",
  "detail": "O nome não pode ser vazio. (Parameter 'nome')",
  "instance": "/api/instituicoes"
}
```

---

## Linhas 101 a 105 — `UnauthorizedAccessException`

```csharp
UnauthorizedAccessException unauthorizedAccess => CreateProblem(
    context,
    StatusCodes.Status403Forbidden,
    "Acesso negado.",
    unauthorizedAccess.Message),
```

Essa exceção é associada a uma operação para a qual o chamador não possui permissão.

A resposta usa:

```text
403 Forbidden
```

O status `403` normalmente significa:

> O servidor entendeu a requisição, mas o acesso não é permitido.

É importante distinguir:

- `401 Unauthorized`: normalmente indica ausência ou invalidade da autenticação;
- `403 Forbidden`: o usuário pode estar autenticado, mas não tem permissão.

Apesar do nome `UnauthorizedAccessException`, o código escolheu `403`, o que é coerente para “acesso proibido”.

Em APIs ASP.NET Core, autenticação e autorização normalmente são tratadas pelo próprio framework com `[Authorize]`, políticas e handlers. Essa exceção pode representar uma regra adicional da aplicação.

---

## Linhas 107 a 111 — `KeyNotFoundException`

```csharp
KeyNotFoundException keyNotFound => CreateProblem(
    context,
    StatusCodes.Status404NotFound,
    "Recurso não encontrado.",
    keyNotFound.Message),
```

`KeyNotFoundException` normalmente significa que uma chave não foi encontrada em uma coleção ou que o projeto decidiu usá-la para representar uma entidade inexistente.

O status escolhido é:

```text
404 Not Found
```

Exemplo de lançamento:

```csharp
throw new KeyNotFoundException(
    $"A instituição com ID {instituicaoId} não foi encontrada.");
```

---

## Linhas 113 a 117 — `FileNotFoundException`

```csharp
FileNotFoundException fileNotFound => CreateProblem(
    context,
    StatusCodes.Status404NotFound,
    "Arquivo não encontrado.",
    fileNotFound.Message),
```

Representa a ausência de um arquivo necessário.

Também é transformada em `404`.

Possíveis cenários:

- arquivo de configuração de um cliente não encontrado;
- template de provisionamento ausente;
- arquivo solicitado pelo endpoint inexistente.

É preciso avaliar semanticamente se o arquivo é um recurso solicitado pelo cliente ou uma dependência interna do servidor.

Se um arquivo interno obrigatório da aplicação desaparecer, um `500` pode ser mais apropriado, pois o cliente não tem como corrigir a situação. O código atual transforma toda `FileNotFoundException` em `404`.

---

## Linhas 119 a 124 — `InstituicaoJaExiste`

```csharp
InstituicaoJaExiste instituicaoJaExiste => CreateProblem(
    context,
    StatusCodes.Status409Conflict,
    instituicaoJaExiste.Message,
    "Já existe uma instituição com os mesmos dados de provisionamento.",
    ("matches", instituicaoJaExiste.Ocorrencias)),
```

É uma exceção personalizada do projeto.

Ela representa um conflito de duplicidade.

### Status 409

```text
409 Conflict
```

É apropriado quando a requisição é válida, mas entra em conflito com o estado atual do sistema.

Exemplo:

- tentar cadastrar uma instituição com domínio já utilizado;
- tentar provisionar novamente o mesmo identificador;
- tentar criar um recurso que precisa ser único.

### Extensão `matches`

```csharp
("matches", instituicaoJaExiste.Ocorrencias)
```

Adiciona ao JSON os registros que causaram o conflito.

Exemplo:

```json
{
  "status": 409,
  "title": "Instituição já cadastrada.",
  "detail": "Já existe uma instituição com os mesmos dados de provisionamento.",
  "instance": "/api/instituicoes",
  "matches": [
    {
      "campo": "dominio",
      "valor": "escola.exemplo.com"
    }
  ],
  "traceId": "00-def..."
}
```

---

## Linhas 126 a 130 — `InvalidOperationException`

```csharp
InvalidOperationException invalidOperation => CreateProblem(
    context,
    StatusCodes.Status409Conflict,
    "Operação inválida para o estado atual.",
    invalidOperation.Message),
```

`InvalidOperationException` indica que uma operação não pode ser executada no estado atual do objeto ou sistema.

Exemplos:

- tentar provisionar uma instituição já provisionada;
- tentar ativar um recurso ainda não configurado;
- tentar cancelar uma operação já finalizada.

O status `409 Conflict` comunica que existe conflito com o estado atual.

Uma observação arquitetural: `InvalidOperationException` é bastante genérica. Em projetos maiores, exceções de domínio específicas podem facilitar respostas mais precisas.

Exemplo:

```csharp
public sealed class InstituicaoJaProvisionadaException : Exception
{
}
```

---

## Linhas 132 a 136 — `TimeoutException`

```csharp
TimeoutException timeout => CreateProblem(
    context,
    StatusCodes.Status504GatewayTimeout,
    "Tempo limite excedido.",
    timeout.Message),
```

`TimeoutException` representa uma operação que não terminou dentro do prazo esperado.

O código retorna:

```text
504 Gateway Timeout
```

O status `504` é normalmente usado quando o servidor atua como gateway ou proxy e uma dependência externa não responde no tempo esperado.

Isso pode fazer sentido se a API estiver esperando:

- Moodle;
- banco remoto;
- outro serviço HTTP;
- serviço de infraestrutura.

Se o timeout ocorrer inteiramente dentro da própria aplicação, sem uma dependência upstream, algumas equipes podem preferir outro status 5xx. A escolha depende do cenário.

---

## Linhas 138 a 142 — `FalhaProvisionamento`

```csharp
FalhaProvisionamento falhaProvisionamento => CreateProblem(
    context,
    StatusCodes.Status500InternalServerError,
    "Falha ao provisionar a instituição.",
    falhaProvisionamento.Message),
```

É uma exceção personalizada que representa falha no processo de provisionamento.

A resposta usa:

```text
500 Internal Server Error
```

Isso informa que o servidor não conseguiu concluir a operação.

O título é genérico e o detalhe usa a mensagem da exceção.

Atenção: mensagens de exceção 5xx não devem revelar informações sensíveis, como:

- senha;
- string de conexão;
- caminho interno do servidor;
- comando de shell;
- stack trace;
- endereço interno;
- token;
- resposta completa de outro sistema.

Se `falhaProvisionamento.Message` puder conter dados internos, seria mais seguro enviar ao cliente uma mensagem genérica e manter a mensagem completa apenas no log.

---

## Linhas 144 a 148 — Caso padrão `_`

```csharp
_ => CreateProblem(
    context,
    StatusCodes.Status500InternalServerError,
    "Erro inesperado.",
    "Ocorreu um erro inesperado ao processar a requisição.")
```

O símbolo `_` funciona como um padrão de descarte.

Ele significa:

> Qualquer exceção que não tenha correspondido aos casos anteriores.

É equivalente ao `default` de um `switch` tradicional.

O middleware evita expor a mensagem real de exceções desconhecidas.

Isso é uma boa prática de segurança.

Por exemplo, uma exceção interna poderia conter:

```text
Connection failed for Server=10.0.0.5;Database=...
```

O cliente recebe apenas:

```json
{
  "status": 500,
  "title": "Erro inesperado.",
  "detail": "Ocorreu um erro inesperado ao processar a requisição."
}
```

A exceção completa continua disponível no log por causa de:

```csharp
logger.LogError(error, ...)
```

---

### Linha 149

```csharp
};
```

Fecha a switch expression e encerra a atribuição à variável `problem`.

---

## Linhas 151 e 152 — Identificador de rastreamento

### Linha 152

```csharp
problem.Extensions["traceId"] = Activity.Current?.Id ?? context.TraceIdentifier;
```

Essa linha adiciona um campo extra ao `ProblemDetails`.

### `problem.Extensions`

É um dicionário usado para incluir propriedades adicionais no JSON.

### `["traceId"]`

Define a chave da extensão.

### `Activity.Current?.Id`

Tenta obter o identificador da atividade de diagnóstico atual.

O operador:

```csharp
?.
```

é o operador condicional nulo.

A expressão:

```csharp
Activity.Current?.Id
```

significa:

- se `Activity.Current` não for `null`, acesse `Id`;
- se for `null`, o resultado será `null`.

Sem esse operador, isto poderia gerar `NullReferenceException`:

```csharp
Activity.Current.Id
```

### `?? context.TraceIdentifier`

Se não houver `Activity.Current?.Id`, usa o identificador do próprio `HttpContext`.

Portanto, a lógica é:

```text
Use o ID da Activity atual.
Se ele não existir, use o TraceIdentifier da requisição.
```

O resultado aparece no JSON:

```json
{
  "traceId": "00-f5d3..."
}
```

Quando um usuário informa esse ID ao suporte, a equipe pode procurar o mesmo valor nos logs e encontrar a exceção correspondente.

Para que isso funcione plenamente, a configuração de logs também deve incluir o trace ID no escopo ou nas propriedades registradas.

---

### Linha 153

```csharp
return problem;
```

Devolve o `ProblemDetails` criado.

O fluxo retorna para:

```csharp
var problem = CreateProblemDetails(context, error);
```

em `WriteProblemDetailsAsync`.

---

### Linha 154

```csharp
}
```

Fecha o método `CreateProblemDetails`.

---

# 5. Método auxiliar `CreateProblem`

## Linhas 156 a 159 — Finalidade do método

Esse método evita repetir a criação de `ProblemDetails` em cada braço do `switch`.

Sem ele, cada caso teria de escrever algo parecido com:

```csharp
new ProblemDetails
{
    Status = 400,
    Title = "...",
    Detail = "...",
    Instance = context.Request.Path
};
```

Centralizar essa construção reduz duplicação.

---

## Linhas 159 a 164 — Assinatura e parâmetros

```csharp
private static ProblemDetails CreateProblem(
    HttpContext context,
    int statusCode,
    string title,
    string detail,
    params (string Key, object Value)[] extensions)
```

### `HttpContext context`

Usado para obter o caminho da requisição.

### `int statusCode`

Código HTTP, como:

```text
400
404
409
500
504
```

### `string title`

Título curto do problema.

### `string detail`

Descrição mais detalhada.

### `params`

A palavra-chave `params` permite passar zero, um ou vários argumentos sem criar manualmente um array.

A assinatura:

```csharp
params (string Key, object Value)[] extensions
```

significa:

- `extensions` é um array;
- cada elemento é uma tupla;
- cada tupla contém uma chave `string`;
- cada tupla contém um valor `object`.

Estas chamadas são válidas:

```csharp
CreateProblem(context, 404, "Não encontrado", "Detalhes");
```

Nenhuma extensão foi passada.

```csharp
CreateProblem(
    context,
    400,
    "Dados inválidos",
    "Revise os dados",
    ("details", erros));
```

Uma extensão foi passada.

```csharp
CreateProblem(
    context,
    409,
    "Conflito",
    "Existem conflitos",
    ("matches", ocorrencias),
    ("field", "domain"));
```

Duas extensões foram passadas.

### `object Value`

Usar `object` permite receber diversos tipos:

```csharp
string
int
bool
List<string>
IReadOnlyCollection<Erro>
objeto personalizado
```

O serializador determinará como converter cada valor para JSON.

---

### Linha 165

```csharp
{
```

Abre o corpo de `CreateProblem`.

---

## Linhas 166 a 172 — Criação de `ProblemDetails`

### Linha 166

```csharp
var problem = new ProblemDetails
```

Cria uma nova instância de `ProblemDetails`.

---

### Linha 167

```csharp
{
```

Inicia um inicializador de objeto.

---

### Linha 168

```csharp
Status = statusCode,
```

Preenche o status com o parâmetro recebido.

---

### Linha 169

```csharp
Title = title,
```

Preenche o título.

---

### Linha 170

```csharp
Detail = detail,
```

Preenche a descrição detalhada.

---

### Linha 171

```csharp
Instance = context.Request.Path
```

Informa em qual recurso ou caminho o erro aconteceu.

Se a requisição for:

```http
POST /api/instituicoes
```

o JSON poderá conter:

```json
{
  "instance": "/api/instituicoes"
}
```

`context.Request.Path` é um `PathString`. O serializador consegue representá-lo como texto.

O campo `instance` não é uma instância do objeto C#. No contexto de `ProblemDetails`, ele identifica a ocorrência ou o recurso relacionado ao problema.

---

### Linha 172

```csharp
};
```

Fecha o inicializador e encerra a declaração da variável.

---

## Linhas 174 a 178 — Inclusão das extensões

### Linha 175

```csharp
foreach (var extension in extensions)
```

Percorre todas as tuplas recebidas no parâmetro `extensions`.

Se a chamada foi:

```csharp
("details", validacao.Erros)
```

a variável `extension` terá:

```csharp
extension.Key
extension.Value
```

### `foreach`

É usado para percorrer uma sequência.

Conceitualmente:

```text
Para cada extensão existente no array, execute o bloco.
```

Se nenhuma extensão foi fornecida, o array estará vazio e o bloco não será executado.

---

### Linha 176

```csharp
{
```

Abre o bloco do `foreach`.

---

### Linha 177

```csharp
problem.Extensions[extension.Key] = extension.Value;
```

Adiciona ou substitui uma entrada no dicionário de extensões.

Exemplo:

```csharp
extension.Key   // "details"
extension.Value // coleção de erros
```

Resultado conceitual:

```csharp
problem.Extensions["details"] = validacao.Erros;
```

Na serialização, a extensão aparece no nível principal do JSON.

Ela não fica necessariamente dentro de um objeto chamado `extensions`.

---

### Linha 178

```csharp
}
```

Fecha o `foreach`.

---

### Linha 180

```csharp
return problem;
```

Devolve o objeto construído para o método chamador.

---

### Linha 181

```csharp
}
```

Fecha `CreateProblem`.

---

### Linha 182

```csharp
}
```

Fecha a classe `GlobalExceptionHandlingMiddleware`.

---

# 6. Fluxo completo de execução

Considere uma requisição:

```http
POST /api/instituicoes
Content-Type: application/json
```

O fluxo pode ser representado assim:

```text
1. O ASP.NET Core recebe a requisição.
                         ↓
2. GlobalExceptionHandlingMiddleware.InvokeAsync é chamado.
                         ↓
3. O middleware entra no bloco try.
                         ↓
4. await next(context) chama o próximo middleware.
                         ↓
5. A requisição chega ao endpoint/controller.
                         ↓
6. O controller chama a camada de aplicação.
                         ↓
7. A camada de aplicação encontra dados inválidos.
                         ↓
8. Ela lança FalhaValidacao.
                         ↓
9. A exceção sobe até o middleware global.
                         ↓
10. catch (Exception error) captura a exceção.
                         ↓
11. Como a resposta ainda não começou, o middleware chama
    WriteProblemDetailsAsync.
                         ↓
12. CreateProblemDetails identifica FalhaValidacao.
                         ↓
13. Cria ProblemDetails com status 400 e extensão details.
                         ↓
14. O middleware registra um LogWarning.
                         ↓
15. Limpa a resposta e define:
    StatusCode = 400
    ContentType = application/problem+json
                         ↓
16. Serializa o ProblemDetails no corpo da resposta.
                         ↓
17. O cliente recebe o JSON padronizado.
```

---

# 7. Mapeamento das exceções

| Exceção | Status | Significado |
|---|---:|---|
| `OperationCanceledException` | 499 | O cliente ou um token cancelou a requisição |
| `FalhaValidacao` | 400 | Os dados enviados são inválidos |
| `ArgumentException` | 400 | Um argumento recebido é inválido |
| `UnauthorizedAccessException` | 403 | A operação não é permitida |
| `KeyNotFoundException` | 404 | O recurso procurado não existe |
| `FileNotFoundException` | 404 | O arquivo procurado não existe |
| `InstituicaoJaExiste` | 409 | Há conflito por duplicidade |
| `InvalidOperationException` | 409 | A operação não é válida no estado atual |
| `TimeoutException` | 504 | Uma operação excedeu o tempo limite |
| `FalhaProvisionamento` | 500 | O provisionamento falhou internamente |
| Qualquer outra exceção | 500 | Erro inesperado |

---

# 8. Exemplos de respostas

## Falha de validação

```json
{
  "status": 400,
  "title": "Não foi possível validar a instituição.",
  "detail": "A requisição contém dados inválidos.",
  "instance": "/api/instituicoes",
  "details": [
    "O nome da instituição é obrigatório.",
    "O domínio informado já está em uso."
  ],
  "traceId": "00-2d2835dc0eb2407f8464e9c2c92003c0-9fb2db957abfce38-00"
}
```

## Instituição duplicada

```json
{
  "status": 409,
  "title": "Instituição já existente.",
  "detail": "Já existe uma instituição com os mesmos dados de provisionamento.",
  "instance": "/api/instituicoes",
  "matches": [
    {
      "tipo": "dominio",
      "valor": "colegio-exemplo"
    }
  ],
  "traceId": "00-a82e2f..."
}
```

## Erro inesperado

```json
{
  "status": 500,
  "title": "Erro inesperado.",
  "detail": "Ocorreu um erro inesperado ao processar a requisição.",
  "instance": "/api/instituicoes",
  "traceId": "00-e6d13c..."
}
```

---

# 9. Como o middleware é registrado

A classe só participa do pipeline se for registrada no `Program.cs`.

Exemplo:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseMiddleware<GlobalExceptionHandlingMiddleware>();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

A ordem dos middlewares importa.

Para capturar exceções dos componentes seguintes, o middleware de exceção deve aparecer antes deles:

```csharp
app.UseMiddleware<GlobalExceptionHandlingMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

Ele só consegue capturar exceções lançadas por elementos executados depois de sua chamada no pipeline.

Exemplo inadequado para capturar erros dos controllers:

```csharp
app.MapControllers();
app.UseMiddleware<GlobalExceptionHandlingMiddleware>();
```

Na prática, o pipeline e o roteamento precisam ser organizados de acordo com a versão e a estrutura do projeto, mas o princípio permanece: o middleware de exceção deve envolver os componentes cujas exceções ele precisa tratar.

---

# 10. Por que usar um middleware em vez de `try/catch` nos controllers?

Sem o middleware:

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Obter(int id)
{
    try
    {
        var instituicao = await service.ObterAsync(id);
        return Ok(instituicao);
    }
    catch (KeyNotFoundException error)
    {
        return NotFound(new
        {
            mensagem = error.Message
        });
    }
    catch (Exception)
    {
        return StatusCode(500);
    }
}
```

Esse padrão teria de ser repetido em vários endpoints.

Com o middleware:

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Obter(int id)
{
    var instituicao = await service.ObterAsync(id);
    return Ok(instituicao);
}
```

A camada de aplicação pode lançar:

```csharp
throw new KeyNotFoundException(
    $"A instituição {id} não foi encontrada.");
```

O middleware converte a exceção automaticamente em `404`.

Benefícios:

- controllers menores;
- respostas consistentes;
- logs centralizados;
- menor duplicação;
- regras de erro concentradas em um único lugar.

---

# 11. Conceitos de C# presentes no arquivo

## 11.1 Construtor primário

```csharp
public sealed class MinhaClasse(Dependencia dependencia)
```

Permite declarar dependências na própria assinatura da classe.

---

## 11.2 Programação assíncrona

```csharp
public async Task InvokeAsync(...)
{
    await next(context);
}
```

Evita bloquear threads durante operações de entrada e saída.

---

## 11.3 Injeção de dependência

```csharp
RequestDelegate next
ILogger<GlobalExceptionHandlingMiddleware> logger
```

As dependências são fornecidas pelo ASP.NET Core.

---

## 11.4 Pattern matching

```csharp
FalhaValidacao validacao => ...
```

Verifica o tipo e cria uma variável já convertida para esse tipo.

---

## 11.5 Switch expression

```csharp
var problem = error switch
{
    ...
};
```

Seleciona e produz um valor de maneira declarativa.

---

## 11.6 Tuplas

```csharp
("details", validacao.Erros)
```

Agrupam temporariamente mais de um valor.

---

## 11.7 Parâmetro `params`

```csharp
params (string Key, object Value)[] extensions
```

Permite receber uma quantidade variável de extensões.

---

## 11.8 Operador condicional nulo

```csharp
Activity.Current?.Id
```

Acessa `Id` somente se `Activity.Current` não for `null`.

---

## 11.9 Operador de coalescência nula

```csharp
problem.Status ?? 500
```

Usa o valor da direita quando o da esquerda é `null`.

---

## 11.10 Inicializador de objeto

```csharp
new ProblemDetails
{
    Status = statusCode,
    Title = title
}
```

Cria o objeto e configura suas propriedades.

---

## 11.11 `static readonly`

```csharp
private static readonly JsonSerializerOptions ProblemJsonOptions
```

Cria uma referência compartilhada pela classe e impede que ela seja substituída depois da inicialização.

---

# 12. Pontos positivos da implementação

1. **Centraliza o tratamento de exceções.**
2. **Evita `try/catch` repetitivo nos controllers.**
3. **Usa `ProblemDetails` para padronizar respostas.**
4. **Diferencia erros 4xx e 5xx nos logs.**
5. **Não expõe a mensagem de exceções inesperadas.**
6. **Inclui `traceId` para investigação.**
7. **Verifica `Response.HasStarted`.**
8. **Preserva o stack trace ao usar `throw;`.**
9. **Respeita o cancelamento da requisição ao serializar.**
10. **Permite campos extras com `ProblemDetails.Extensions`.**

---

# 13. Pontos que merecem atenção

## 13.1 Exposição de mensagens internas

Alguns casos devolvem diretamente:

```csharp
falhaProvisionamento.Message
timeout.Message
fileNotFound.Message
invalidOperation.Message
```

Dependendo de como essas exceções são criadas, as mensagens podem revelar detalhes internos.

Uma abordagem mais segura para erros 5xx seria:

```csharp
FalhaProvisionamento => CreateProblem(
    context,
    StatusCodes.Status500InternalServerError,
    "Falha ao provisionar a instituição.",
    "Não foi possível concluir o provisionamento. Utilize o traceId ao contatar o suporte.")
```

A mensagem completa continuaria no log.

---

## 13.2 Uso genérico de exceções padrão

Exceções como:

```csharp
KeyNotFoundException
InvalidOperationException
ArgumentException
```

podem ser lançadas por bibliotecas internas por motivos não relacionados à resposta HTTP pretendida.

Exemplo: uma biblioteca pode lançar `KeyNotFoundException` por um erro de programação, e o middleware responderá `404`, embora o problema real seja um `500`.

Exceções personalizadas deixam a intenção mais explícita:

```csharp
InstituicaoNaoEncontradaException
OperacaoDeProvisionamentoInvalidaException
```

---

## 13.3 `FileNotFoundException` nem sempre significa 404

Se o usuário solicitou um arquivo que não existe, `404` é adequado.

Se a API perdeu um arquivo interno obrigatório para funcionar, o problema é interno e pode ser mais apropriado devolver `500`.

---

## 13.4 `OperationCanceledException`

Nem todo cancelamento é necessariamente causado pelo cliente.

Uma operação pode usar um token interno cancelado pelo servidor.

Uma verificação mais específica pode considerar:

```csharp
context.RequestAborted.IsCancellationRequested
```

Assim, seria possível diferenciar:

- cliente desconectado;
- cancelamento interno;
- timeout controlado pela aplicação.

---

## 13.5 Ordem no pipeline

Se o middleware for registrado depois de um componente, ele não capturará exceções desse componente.

A posição no `Program.cs` deve ser escolhida com cuidado.

---

# 14. Versão conceitualmente equivalente sem construtor primário

Para quem está começando em C#, esta versão pode ser mais fácil de visualizar:

```csharp
public sealed class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;

    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception error)
        {
            if (context.Response.HasStarted)
            {
                _logger.LogError(
                    error,
                    "Uma exceção ocorreu depois que a resposta já havia começado.");

                throw;
            }

            await WriteProblemDetailsAsync(context, error);
        }
    }

    // Os demais métodos continuariam iguais.
}
```

Na versão original:

```csharp
RequestDelegate next
ILogger<GlobalExceptionHandlingMiddleware> logger
```

substituem, de forma mais compacta:

```csharp
private readonly RequestDelegate _next;
private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;
```

e o construtor tradicional.

---

# 15. Exemplo prático de lançamento e tratamento

## Camada de aplicação

```csharp
public async Task<InstituicaoDto> ObterInstituicaoAsync(
    int instituicaoId,
    CancellationToken cancellationToken)
{
    var instituicao = await repositorio.ObterPorIdAsync(
        instituicaoId,
        cancellationToken);

    if (instituicao is null)
    {
        throw new KeyNotFoundException(
            $"A instituição de ID {instituicaoId} não foi encontrada.");
    }

    return new InstituicaoDto(
        instituicao.Id,
        instituicao.Nome);
}
```

## Controller

```csharp
[ApiController]
[Route("api/instituicoes")]
public sealed class InstituicoesController : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<ActionResult<InstituicaoDto>> Obter(
        int id,
        [FromServices] InstituicaoService service,
        CancellationToken cancellationToken)
    {
        var instituicao = await service.ObterInstituicaoAsync(
            id,
            cancellationToken);

        return Ok(instituicao);
    }
}
```

O controller não contém `try/catch`.

Se a instituição não existir:

1. o serviço lança `KeyNotFoundException`;
2. a exceção atravessa o controller;
3. o middleware captura;
4. o `switch` seleciona o caso de `KeyNotFoundException`;
5. a API devolve `404`.

---

# 16. Resumo mental do código

Uma forma simples de memorizar o funcionamento é:

```text
InvokeAsync
    tenta executar o próximo componente
    se ocorrer erro:
        se a resposta já começou:
            registra e relança
        senão:
            cria ProblemDetails
            registra o erro
            limpa a resposta
            define status e Content-Type
            grava o JSON
```

E o método de classificação funciona assim:

```text
Exceção recebida
    ↓
switch pelo tipo
    ↓
status + título + detalhe + extensões
    ↓
traceId
    ↓
ProblemDetails
```

---

# 17. Conclusão

O `GlobalExceptionHandlingMiddleware` funciona como uma camada de proteção ao redor do restante da API.

A linha mais importante para compreender o fluxo é:

```csharp
await next(context);
```

Ela chama todos os componentes seguintes. Como está dentro de um `try`, qualquer exceção não tratada que subir desses componentes poderá ser capturada pelo `catch`.

Depois, o código usa o tipo concreto da exceção para decidir qual resposta HTTP produzir. O resultado é uma API com erros mais consistentes, logs centralizados e controllers mais limpos.

Os principais conceitos de C# e ASP.NET Core demonstrados são:

- middleware;
- pipeline HTTP;
- injeção de dependência;
- construtor primário;
- `async` e `await`;
- `try/catch`;
- pattern matching;
- switch expression;
- tuplas;
- `params`;
- serialização JSON;
- `ProblemDetails`;
- logging;
- cancelamento;
- tracing.
