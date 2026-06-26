# Visão geral

O papel dele é semelhante ao de um `Program.cs` ou de um serviço de inicialização em .NET que:

1. lê configurações de variáveis de ambiente;
2. conecta-se ao Moodle;
3. atualiza dados da instituição;
4. cria um usuário técnico;
5. cria um papel com permissões específicas;
6. habilita a API REST;
7. gera um token;
8. salva esse token em um arquivo.

Ele roda depois que o Moodle já foi instalado ou atualizado pelo `docker-entrypoint.sh`.

## Tradução rápida de PHP para C#

| PHP | Aproximação em C# |
|---|---|
| `$nome` | `var nome` |
| `$objeto->campo` | `objeto.Campo` |
| `$objeto->metodo()` | `objeto.Metodo()` |
| `Classe::metodo()` | `Classe.Metodo()` estático |
| `function executar(): void` | `void Executar()` |
| `stdClass` | `dynamic`, `ExpandoObject` ou DTO sem tipagem forte |
| `['campo' => $valor]` | `Dictionary<string, object>` ou objeto anônimo |
| `(object)[...]` | conversão de array para objeto dinâmico |
| `require_once` | referência/carregamento obrigatório de código |
| `global $DB` | acesso a uma variável global |
| `===` | igualdade estrita de valor e tipo |
| `!==` | diferença estrita de valor ou tipo |
| `.` | concatenação de strings |
| `??` | operador de coalescência nula, igual ao C# |
| `fn($x) => ...` | lambda `x => ...` |

Para manter a explicação legível, agruparei linhas em branco e comentários, mas explicarei individualmente todas as instruções executáveis.

---

# 1. Inicialização do script

## Linha 1

```php id="d7y2gp"
<?php
```

Indica o início de um código PHP.

É semelhante a o compilador identificar que o arquivo contém código C#, embora em C# isso seja determinado pela extensão `.cs`.

## Linhas 3–6

```php id="4or9ol"
// Informa ao Moodle que este arquivo esta rodando via linha de comando.
// Muitas rotinas internas do Moodle verificam essa constante para permitir
// execucao fora do navegador.
```

São comentários explicando o motivo da próxima instrução.

## Linha 7

```php id="p5ms54"
define('CLI_SCRIPT', true);
```

Cria uma constante global chamada `CLI_SCRIPT` com valor `true`.

Em C#, uma aproximação seria:

```csharp id="nxr94s"
public const bool CliScript = true;
```

A diferença é que, em PHP, `define()` cria essa constante durante a execução.

O Moodle verifica essa constante para saber que o código está sendo executado pelo terminal, e não por uma requisição HTTP. Algumas proteções internas do Moodle impedem determinados arquivos de serem executados diretamente fora de um contexto válido. Essa constante comunica que o contexto de linha de comando é intencional.

---

## Linhas 9–12

São comentários explicando o carregamento do ambiente do Moodle.

## Linha 13

```php id="52jkgn"
require_once(__DIR__ . '/../config.php');
```

Essa linha carrega o `config.php` do Moodle.

Separando a expressão:

```php id="ledexe"
__DIR__
```

É uma constante especial do PHP que representa o diretório do arquivo atual.

Se o arquivo estiver em:

```text id="xrf696"
/var/www/html/bootstrap/provision.php
```

então:

```php id="q8eosv"
__DIR__
```

será:

```text id="1to7mg"
/var/www/html/bootstrap
```

A expressão:

```php id="72jm6g"
__DIR__ . '/../config.php'
```

concatena o diretório atual com `../config.php`, chegando a:

```text id="6nec3q"
/var/www/html/config.php
```

O operador `.` concatena strings em PHP. Em C# seria `+`.

O `require_once`:

- carrega e executa o arquivo;
- interrompe o programa se ele não for encontrado;
- garante que seja carregado apenas uma vez.

Uma comparação conceitual com C# seria a inicialização do host e dos serviços:

```csharp id="zjtwdn"
var builder = WebApplication.CreateBuilder(args);
```

Depois do carregamento do `config.php`, ficam disponíveis estruturas importantes do Moodle, principalmente:

```php id="vqs6on"
$CFG
$DB
```

- `$CFG` contém configurações globais;
- `$DB` é a abstração de acesso ao banco do Moodle.

## Linha 14

```php id="5xial8"
require_once($CFG->dirroot . '/user/lib.php');
```

Carrega a biblioteca de gerenciamento de usuários do Moodle.

`$CFG->dirroot` contém o diretório raiz do código do Moodle, normalmente:

```text id="e48bxn"
/var/www/html
```

Portanto, o arquivo carregado será:

```text id="k8akl1"
/var/www/html/user/lib.php
```

Essa biblioteca disponibiliza funções como:

```php id="bb9hc0"
user_create_user()
user_update_user()
```

Em C#, seria conceitualmente semelhante a importar e disponibilizar um serviço de usuários:

```csharp id="7ng4b5"
using Moodle.Users;
```

Mas `require_once` não é exatamente igual a `using`. O `using` apenas permite referenciar tipos já compilados; o `require_once` carrega e executa outro arquivo PHP durante a execução.

## Linha 15

```php id="jms4zu"
require_once($CFG->dirroot . '/webservice/lib.php');
```

Carrega a biblioteca do Moodle que manipula serviços externos e webservices.

É nessa biblioteca que está a classe:

```php id="hgq81k"
webservice
```

Ela será utilizada para criar e atualizar o serviço REST da W3Soft. fileciteturn21file0L3-L16

---

# 2. Função de log

## Linhas 17–18

Comentários explicando que os logs serão enviados para a saída padrão do processo.

## Linha 19

```php id="qhon2x"
function bootstrap_log(string $message): void {
```

Declara uma função chamada `bootstrap_log`.

Ela:

- recebe um parâmetro `$message`;
- exige que ele seja `string`;
- não retorna valor, por isso `: void`.

Equivalente aproximado em C#:

```csharp id="1q4kcm"
static void BootstrapLog(string message)
{
}
```

## Linha 20

```php id="98ers5"
fwrite(STDOUT, "[moodle-bootstrap] {$message}" . PHP_EOL);
```

Escreve uma mensagem na saída padrão.

### `STDOUT`

Representa o fluxo de saída padrão do processo.

No container, essa saída é capturada pelo Docker e pode ser consultada com:

```bash id="77u29p"
docker logs nome-do-container
```

Em C# seria semelhante a:

```csharp id="1a921m"
Console.WriteLine($"[moodle-bootstrap] {message}");
```

### `"{$message}"`

É interpolação de string em PHP.

Equivale a:

```csharp id="5uqo9z"
$"{message}"
```

### `PHP_EOL`

Representa a quebra de linha adequada ao sistema operacional.

Em C#, o equivalente seria:

```csharp id="56gwpn"
Environment.NewLine
```

## Linha 21

```php id="xcfzox"
}
```

Encerra a função.

---

# 3. Função de erro fatal

## Linhas 23–24

Comentários explicando que essa função registra o erro e encerra o programa.

## Linha 25

```php id="ppt7ik"
function bootstrap_fail(string $message): never {
```

Declara uma função que recebe uma mensagem e tem retorno `never`.

`never` significa que a função **jamais retorna ao chamador**.

Isso ocorre porque ela sempre encerra o processo.

Em C#, não há um equivalente direto amplamente usado na assinatura. Conceitualmente seria:

```csharp id="0e9vjc"
static void BootstrapFail(string message)
{
    Console.Error.WriteLine(message);
    Environment.Exit(1);
}
```

## Linha 26

```php id="0dpn7f"
fwrite(STDERR, "[moodle-bootstrap] ERROR: {$message}" . PHP_EOL);
```

Escreve a mensagem no fluxo de erros, `STDERR`.

Em C#:

```csharp id="dvec0z"
Console.Error.WriteLine($"[moodle-bootstrap] ERROR: {message}");
```

Separar `STDOUT` e `STDERR` permite que ferramentas de infraestrutura distingam logs normais de erros.

## Linha 27

```php id="8ak8tc"
exit(1);
```

Encerra imediatamente o processo com código de saída `1`.

Por convenção:

- `0` significa sucesso;
- valores diferentes de `0` significam erro.

Em C#:

```csharp id="j9cwoc"
Environment.Exit(1);
```

Como esse script é executado durante a inicialização do container, um erro aqui pode impedir que o container seja considerado iniciado corretamente. fileciteturn21file0L18-L29

---

# 4. Leitura obrigatória de variável de ambiente

## Linha 32

```php id="a5uwrh"
function env_required(string $name): string {
```

Cria uma função que:

- recebe o nome de uma variável de ambiente;
- retorna o valor como string;
- falha se a variável não existir.

Em C#:

```csharp id="t2d58y"
static string GetRequiredEnvironmentVariable(string name)
```

## Linha 33

```php id="b6ne1v"
$value = getenv($name);
```

Lê a variável de ambiente.

Em C#:

```csharp id="maqxjm"
var value = Environment.GetEnvironmentVariable(name);
```

No PHP, `getenv()` pode retornar:

- uma string, quando a variável existe;
- `false`, quando não existe.

## Linha 35

```php id="1coz1t"
if ($value === false || $value === '') {
```

Verifica duas condições:

```php id="05mlgb"
$value === false
```

A variável não existe.

```php id="8deexk"
$value === ''
```

A variável existe, mas está vazia.

O operador `===` é uma igualdade estrita. Ele compara o valor e o tipo.

Em C#:

```csharp id="zz2lby"
if (value is null || value == string.Empty)
```

Poderia também ser:

```csharp id="qk2lbf"
if (string.IsNullOrEmpty(value))
```

## Linha 36

```php id="todckx"
bootstrap_fail("Missing required environment variable: {$name}");
```

Interrompe o programa informando qual variável obrigatória está ausente.

Isso é uma estratégia de **fail fast**: falhar imediatamente ao detectar uma configuração inválida, em vez de continuar e apresentar um erro menos claro posteriormente.

## Linha 39

```php id="7fyqmk"
return $value;
```

Se a variável existir e não estiver vazia, retorna seu valor.

Em C#:

```csharp id="mec04v"
return value;
```

Depois dessa validação, o restante do código pode tratar o retorno como uma string válida. fileciteturn21file0L31-L41

---

# 5. Variável de ambiente opcional

## Linha 45

```php id="g8lhpa"
function env_default(string $name, string $default): string {
```

Cria uma função que lê uma variável de ambiente, mas permite fornecer um valor padrão.

Em C#:

```csharp id="gjtr5o"
static string GetEnvironmentVariableOrDefault(
    string name,
    string defaultValue)
```

## Linha 46

```php id="d7eixz"
$value = getenv($name);
```

Tenta ler a variável.

## Linha 48

```php id="ndhnbv"
if ($value === false || $value === '') {
```

Verifica se ela não existe ou está vazia.

## Linha 49

```php id="7fz4ba"
return $default;
```

Nesse caso, retorna o valor padrão.

## Linha 52

```php id="jxf70k"
return $value;
```

Se a variável estiver configurada, retorna o valor informado pelo Docker ou sistema operacional.

Em C#, toda a função poderia ser escrita assim:

```csharp id="o063g6"
static string GetEnvironmentVariableOrDefault(
    string name,
    string defaultValue)
{
    var value = Environment.GetEnvironmentVariable(name);

    return string.IsNullOrEmpty(value)
        ? defaultValue
        : value;
}
```

Essa função permite que uma configuração tenha comportamento padrão, mas possa ser sobrescrita por cada instituição. fileciteturn21file0L43-L54

---

# 6. Conversão de variável para booleano

## Linha 58

```php id="ewoukx"
function env_bool(string $name, bool $default): bool {
```

Cria uma função para converter uma variável de ambiente em booleano.

Variáveis de ambiente são sempre strings. Portanto, valores como:

```text id="b03h7n"
true
false
1
0
yes
no
```

precisam ser interpretados.

## Linha 59

```php id="n01hym"
$value = getenv($name);
```

Lê a variável.

## Linhas 61–63

```php id="y0j3lh"
if ($value === false || $value === '') {
    return $default;
}
```

Se ela não existir ou estiver vazia, retorna o valor booleano padrão recebido no parâmetro.

## Linha 65

```php id="wbksa5"
return in_array(
    strtolower($value),
    ['1', 'true', 'yes', 'on'],
    true
);
```

Essa expressão contém três operações.

### `strtolower($value)`

Transforma o texto em letras minúsculas.

Assim:

```text id="rj41ku"
TRUE
True
true
```

tornam-se:

```text id="7juel0"
true
```

### `in_array(...)`

Verifica se o valor está dentro do array:

```php id="278c4h"
['1', 'true', 'yes', 'on']
```

Se estiver, retorna `true`.

Qualquer outro valor retorna `false`, inclusive:

```text id="arvtww"
0
false
no
off
qualquer-coisa
```

### Terceiro argumento `true`

Solicita uma comparação estrita de tipo e valor.

Uma versão C# seria:

```csharp id="n862jv"
static bool GetBooleanEnvironmentVariable(
    string name,
    bool defaultValue)
{
    var value = Environment.GetEnvironmentVariable(name);

    if (string.IsNullOrEmpty(value))
        return defaultValue;

    return new[] { "1", "true", "yes", "on" }
        .Contains(value.ToLowerInvariant());
}
```

Um detalhe importante é que uma string inválida não gera erro. Ela simplesmente é interpretada como `false`. fileciteturn21file0L56-L67

---

# 7. Transformação de CSV em array

## Linha 70

```php id="fdnkc5"
function split_csv(string $value): array {
```

Cria uma função que recebe uma string separada por vírgulas e retorna um array.

Exemplo de entrada:

```text id="r046e8"
funcao1, funcao2,,funcao3
```

Saída:

```php id="yu7o06"
[
    'funcao1',
    'funcao2',
    'funcao3'
]
```

## Linha 71

```php id="duta5i"
$items = array_map('trim', explode(',', $value));
```

Essa linha executa duas operações.

### `explode(',', $value)`

Divide a string pela vírgula.

Em C#:

```csharp id="xkuscj"
value.Split(',')
```

### `array_map('trim', ...)`

Executa `trim` em cada elemento.

`trim` remove espaços no início e no fim.

Em C#:

```csharp id="n3hpjm"
value.Split(',')
     .Select(item => item.Trim())
```

## Linha 72

```php id="3fgaam"
return array_values(
    array_filter(
        $items,
        static fn(string $item): bool => $item !== ''
    )
);
```

### `array_filter`

Filtra os itens.

A lambda:

```php id="nx06e9"
static fn(string $item): bool => $item !== ''
```

mantém apenas strings não vazias.

Em C#:

```csharp id="53vjyk"
.Where(item => item != "")
```

### `array_values`

Reorganiza os índices do array.

Depois de um filtro, o PHP pode preservar os índices antigos. `array_values()` cria novamente índices sequenciais:

```text id="quz8zy"
0, 1, 2...
```

A função completa em C# seria:

```csharp id="zfl294"
static string[] SplitCsv(string value)
{
    return value
        .Split(',')
        .Select(item => item.Trim())
        .Where(item => item != string.Empty)
        .ToArray();
}
```

fileciteturn21file0L69-L74

---

# 8. Atualização da identidade do site

## Linha 77

```php id="245sn0"
function update_site_identity(): void {
```

Declara a função responsável por atualizar os dados visíveis da instituição.

## Linha 78

```php id="dv6u7t"
global $DB;
```

Importa a variável global `$DB` para o escopo da função.

Em PHP, variáveis globais não ficam automaticamente disponíveis dentro de funções. É necessário declarar:

```php id="r2r5hp"
global $DB;
```

Em uma aplicação C# moderna, o normal seria receber uma dependência pelo construtor:

```csharp id="iy1dyi"
public class SiteProvisioner
{
    private readonly IMoodleDatabase _database;

    public SiteProvisioner(IMoodleDatabase database)
    {
        _database = database;
    }
}
```

O uso de `global` torna a dependência implícita. Porém, esse padrão é comum em scripts internos do Moodle.

---

## Linhas 80–84

```php id="0uzx6f"
$fullname = env_required('MOODLE_SITE_FULLNAME');
$shortname = env_required('MOODLE_SITE_SHORTNAME');
$summary = env_default('MOODLE_SITE_SUMMARY', '');
$supportemail = env_required('MOODLE_SUPPORT_EMAIL');
$timezone = env_required('MOODLE_ADMIN_TIMEZONE');
```

Lê as configurações da instituição.

- `$fullname`: nome completo da plataforma;
- `$shortname`: nome abreviado;
- `$summary`: descrição, opcional;
- `$supportemail`: e-mail de suporte;
- `$timezone`: fuso horário.

Exemplo:

```env id="mirha8"
MOODLE_SITE_FULLNAME=Escola Municipal A
MOODLE_SITE_SHORTNAME=Escola A
MOODLE_SITE_SUMMARY=Ambiente virtual da Escola A
MOODLE_SUPPORT_EMAIL=suporte@escolaa.com.br
MOODLE_ADMIN_TIMEZONE=America/Maceio
```

---

## Linha 86

```php id="wks0dw"
$site = $DB->get_record(
    'course',
    ['id' => SITEID],
    '*',
    MUST_EXIST
);
```

Busca no banco o registro que representa a página principal do Moodle.

### `'course'`

Nome lógico da tabela.

O Moodle acrescenta automaticamente o prefixo configurado. Portanto, pode consultar fisicamente:

```text id="67w7yw"
mdl_course
```

### `['id' => SITEID]`

É o filtro:

```sql id="f4plog"
WHERE id = SITEID
```

`SITEID` é uma constante do Moodle que identifica o curso especial que representa o site principal.

### `'*'`

Solicita todos os campos.

### `MUST_EXIST`

Determina que o registro deve existir.

Se ele não existir, o Moodle lança uma exceção.

Em C# com EF Core, seria aproximadamente:

```csharp id="slcmv5"
var site = await dbContext.Courses
    .SingleAsync(course => course.Id == SiteId);
```

## Linha 87

```php id="igo060"
$changed = false;
```

Cria uma flag para controlar se algum campo foi alterado.

Isso evita executar um `UPDATE` desnecessário.

---

## Linha 92

```php id="iiex1q"
foreach (
    [
        'fullname' => $fullname,
        'shortname' => $shortname,
        'summary' => $summary
    ] as $field => $value
) {
```

Cria um array associativo:

```php id="ul90i0"
[
    'fullname' => $fullname,
    'shortname' => $shortname,
    'summary' => $summary
]
```

Ele é parecido com um dicionário:

```csharp id="qnaqxr"
var fields = new Dictionary<string, string>
{
    ["fullname"] = fullName,
    ["shortname"] = shortName,
    ["summary"] = summary
};
```

O `foreach` recebe:

- `$field`: nome do campo;
- `$value`: valor desejado.

Exemplo da primeira iteração:

```text id="xrhrx5"
$field = "fullname"
$value = "Escola Municipal A"
```

## Linha 93

```php id="1c6qpa"
if ((string)$site->{$field} !== $value) {
```

Essa é uma das linhas mais específicas do PHP.

### `$site->{$field}`

Acessa dinamicamente uma propriedade cujo nome está dentro de `$field`.

Se:

```php id="o4etci"
$field = 'fullname';
```

então:

```php id="rgxl02"
$site->{$field}
```

equivale a:

```php id="nbd7of"
$site->fullname
```

Em C# seria necessário usar reflexão, `dynamic` ou um mapeamento manual.

### `(string)`

Converte o valor atual para string.

### `!==`

Compara estritamente.

Portanto, a linha pergunta:

> O valor atual desse campo é diferente do valor configurado na variável de ambiente?

## Linha 94

```php id="98sng8"
$site->{$field} = $value;
```

Atualiza dinamicamente a propriedade.

## Linha 95

```php id="6x4nge"
$changed = true;
```

Registra que houve pelo menos uma alteração.

---

## Linha 99

```php id="gxv9tv"
if ($changed) {
```

Verifica se algum campo foi modificado.

## Linha 100

```php id="ms5eh1"
$site->timemodified = time();
```

Atualiza a data da última alteração.

`time()` retorna o timestamp Unix atual em segundos.

Em C#:

```csharp id="ugen68"
site.TimeModified = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
```

## Linha 101

```php id="d2pvud"
$DB->update_record('course', $site);
```

Atualiza o registro na tabela `course`.

É semelhante a:

```csharp id="zzayam"
dbContext.Courses.Update(site);
await dbContext.SaveChangesAsync();
```

A API do Moodle identifica o registro pelo campo `id` presente no objeto.

## Linha 102

```php id="dxmskh"
bootstrap_log("Updated site identity.");
```

Registra que a identidade foi atualizada.

## Linhas 103–105

```php id="dqyeo5"
} else {
    bootstrap_log("Site identity already up to date.");
}
```

Se nada mudou, apenas informa que os dados já estavam corretos.

Esse comportamento torna a operação **idempotente**: executar a função repetidamente leva ao mesmo estado final, sem criar registros duplicados.

---

## Linha 108

```php id="v1qraj"
set_config('supportemail', $supportemail);
```

Salva o e-mail de suporte nas configurações globais do Moodle.

## Linha 109

```php id="yb2onz"
set_config('timezone', $timezone);
```

Salva o fuso horário.

`set_config()` normalmente persiste esses valores na tabela global de configurações do Moodle.

Diferentemente dos três campos anteriores, esses valores não estão no registro especial de `course`, mas na configuração global. fileciteturn21file0L76-L111

---

# 9. Atualização do administrador

## Linha 115

```php id="n5m97q"
function update_admin_user(bool $firstinstall): stdClass {
```

Declara uma função que:

- recebe `$firstinstall`, indicando se é a primeira instalação;
- retorna um objeto genérico `stdClass` representando o administrador.

Em C# seria melhor retornar uma classe como:

```csharp id="jzxpls"
AdminUser UpdateAdminUser(bool firstInstall)
```

## Linha 116

```php id="etl6sm"
global $DB;
```

Disponibiliza o banco global dentro da função.

## Linha 118

```php id="9os9bs"
$username = env_default('MOODLE_ADMIN_USER', 'admin');
```

Lê o nome do administrador.

Se não estiver configurado, utiliza:

```text id="at0ozk"
admin
```

## Linha 119

```php id="pwigoj"
$admin = $DB->get_record(
    'user',
    ['username' => $username, 'deleted' => 0],
    '*',
    MUST_EXIST
);
```

Busca um usuário:

- com o nome configurado;
- que não esteja marcado como excluído;
- exigindo que ele exista.

Conceitualmente:

```csharp id="bb8pg3"
var admin = await dbContext.Users.SingleAsync(
    user => user.Username == username && !user.Deleted);
```

O administrador já deve ter sido criado anteriormente pelo instalador do Moodle.

---

## Linha 121

```php id="aujyxy"
$user = (object)[
```

Cria um array e o converte para objeto.

Em PHP:

```php id="er9ajk"
(object)[
    'id' => 1,
    'firstname' => 'Maxwell'
]
```

gera um `stdClass` com propriedades:

```php id="flod8l"
$user->id
$user->firstname
```

Em C# seria semelhante a:

```csharp id="m46147"
var user = new
{
    Id = admin.Id,
    FirstName = ...
};
```

Porém o objeto anônimo de C# é imutável. Para atualizar dados, normalmente seria utilizado um DTO.

## Linhas 122–128

```php id="on3g73"
'id' => $admin->id,
'firstname' => env_required('MOODLE_ADMIN_FIRSTNAME'),
'lastname' => env_required('MOODLE_ADMIN_LASTNAME'),
'email' => env_required('MOODLE_ADMIN_EMAIL'),
'city' => env_required('MOODLE_ADMIN_CITY'),
'country' => env_required('MOODLE_ADMIN_COUNTRY'),
'timezone' => env_required('MOODLE_ADMIN_TIMEZONE'),
```

Monta o objeto com os dados que devem ser atualizados.

O `id` é fundamental porque informa ao Moodle qual usuário será alterado.

---

## Linha 134

```php id="671iuk"
$resetpassword = env_bool(
    'MOODLE_ADMIN_RESET_PASSWORD',
    false
);
```

Verifica se o sistema deve redefinir a senha do administrador.

Por padrão, é `false`.

Isso evita que toda reinicialização do container redefina a senha do administrador para o valor presente no arquivo `.env`.

## Linhas 135–137

```php id="tbk8py"
if ($resetpassword) {
    $user->password = env_required('MOODLE_ADMIN_PASSWORD');
}
```

Somente adiciona a propriedade `password` ao objeto quando a redefinição foi explicitamente solicitada.

Esse comportamento é importante:

```text id="nhxym8"
MOODLE_ADMIN_RESET_PASSWORD=false
```

O script atualiza nome, e-mail e localização, mas não toca na senha.

```text id="f8q6u5"
MOODLE_ADMIN_RESET_PASSWORD=true
```

O script exige `MOODLE_ADMIN_PASSWORD` e redefine a senha.

---

## Linha 139

```php id="qke34l"
user_update_user($user, $resetpassword, false);
```

Chama uma função interna do Moodle para atualizar o usuário.

Os argumentos representam, conceitualmente:

```php id="1zlpv3"
user_update_user(
    usuário,
    deve_atualizar_senha,
    deve_disparar_evento
);
```

O terceiro argumento está como `false`, evitando o disparo normal do evento de atualização.

Uma comparação simplificada:

```csharp id="pn0qov"
await userService.UpdateAsync(
    user,
    updatePassword: resetPassword,
    triggerEvent: false);
```

É melhor utilizar `user_update_user()` do que atualizar diretamente a tabela, porque a função aplica regras internas do Moodle.

---

## Linha 143

```php id="6ouy3w"
if (
    $firstinstall
    && env_bool(
        'MOODLE_ADMIN_FORCE_PASSWORD_CHANGE_ON_INSTALL',
        true
    )
) {
```

A condição exige duas coisas:

1. ser a primeira instalação;
2. a configuração de troca obrigatória estar ativada.

Por padrão, a troca obrigatória está ativa.

## Linha 144

```php id="fabj05"
set_user_preference(
    'auth_forcepasswordchange',
    1,
    $admin->id
);
```

Define uma preferência para que o administrador seja obrigado a trocar a senha no primeiro login.

Parâmetros:

- nome da preferência;
- valor `1`, significando habilitado;
- ID do usuário.

## Linha 145

```php id="35n1ir"
bootstrap_log(
    "Admin password change will be required on first login."
);
```

Registra a configuração.

---

## Linha 148

```php id="wa3qcd"
bootstrap_log("Admin user profile configured: {$username}.");
```

Registra que o administrador foi configurado.

## Linha 149

```php id="td72c3"
return $DB->get_record(
    'user',
    ['id' => $admin->id],
    '*',
    MUST_EXIST
);
```

Busca novamente o usuário no banco e retorna a versão atualizada.

Isso garante que o objeto retornado contenha os dados persistidos, e não apenas o objeto parcial utilizado na atualização.

É semelhante a:

```csharp id="3upq2j"
await dbContext.Entry(admin).ReloadAsync();
return admin;
```

O administrador retornado será utilizado posteriormente como o criador do token. fileciteturn21file0L113-L151

---

# 10. Habilitação dos webservices

## Linha 154

```php id="g0mhas"
function ensure_webservice_settings(): void {
```

Declara a função que garante que a API esteja habilitada.

O uso de `ensure` no nome significa aproximadamente:

> Garanta que essa configuração exista; crie ou corrija se necessário.

## Linha 155

```php id="j9mn6p"
global $CFG;
```

Disponibiliza a configuração global do Moodle.

## Linha 157

```php id="tffhwq"
set_config('enablewebservices', '1');
```

Persiste a habilitação dos webservices.

O valor é uma string `'1'`, porque muitas configurações do Moodle são armazenadas como texto.

## Linha 158

```php id="ui1lbg"
$CFG->enablewebservices = '1';
```

Atualiza também o objeto de configuração que já está carregado na memória.

Por que as duas linhas são necessárias?

```php id="m0jlh6"
set_config(...)
```

Atualiza o banco.

```php id="rfibn1"
$CFG->enablewebservices = '1'
```

Atualiza o estado da execução atual.

Sem a segunda linha, o banco estaria correto, mas o restante do script poderia continuar vendo o valor antigo até a próxima execução.

É comparável a atualizar o banco e também atualizar o cache em memória.

---

## Linha 160

```php id="bspnik"
$protocols = empty($CFG->webserviceprotocols)
    ? []
    : split_csv($CFG->webserviceprotocols);
```

Usa o operador ternário.

Em C#:

```csharp id="ufzqqp"
var protocols = string.IsNullOrEmpty(config.WebServiceProtocols)
    ? Array.Empty<string>()
    : SplitCsv(config.WebServiceProtocols);
```

Se não houver protocolos configurados, cria um array vazio.

Caso contrário, transforma a string CSV em array.

## Linha 161

```php id="7by3az"
if (!in_array('rest', $protocols, true)) {
```

Verifica se `rest` ainda não está no array.

O `!` significa negação.

Em C#:

```csharp id="1seuc5"
if (!protocols.Contains("rest"))
```

## Linha 164

```php id="t9t8q5"
$protocols[] = 'rest';
```

Adiciona `rest` ao final do array.

Em C#:

```csharp id="tranvm"
protocols.Add("rest");
```

## Linha 165

```php id="hvhgzb"
set_config(
    'webserviceprotocols',
    implode(',', $protocols)
);
```

`implode()` junta o array utilizando vírgulas.

Exemplo:

```php id="n9oasy"
['soap', 'rest']
```

vira:

```text id="ozrt9k"
soap,rest
```

Depois, salva o valor no banco.

Em C#:

```csharp id="jhgne5"
string.Join(",", protocols)
```

## Linha 166

```php id="n1qmmv"
$CFG->webserviceprotocols = implode(',', $protocols);
```

Atualiza também a configuração em memória.

## Linha 167

```php id="uqyrin"
bootstrap_log("Enabled REST webservice protocol.");
```

Registra que REST foi habilitado.

## Linhas 168–170

```php id="d01k81"
} else {
    bootstrap_log("REST webservice protocol already enabled.");
}
```

Se REST já estava configurado, apenas registra essa informação.

Novamente, o comportamento é idempotente. fileciteturn21file0L153-L172

---

# 11. Criação do serviço externo

## Linha 175

```php id="1ylsg7"
function ensure_service(array $functions): stdClass {
```

Recebe a lista de funções REST que deverão fazer parte do serviço.

Retorna o serviço criado ou encontrado.

## Linha 176

```php id="e3oux0"
global $DB;
```

Disponibiliza o banco.

## Linha 178

```php id="z59ks4"
$manager = new webservice();
```

Cria uma instância da classe interna `webservice` do Moodle.

Em C#:

```csharp id="48kosa"
var manager = new WebServiceManager();
```

Essa classe foi disponibilizada por:

```php id="hz0k03"
require_once($CFG->dirroot . '/webservice/lib.php');
```

## Linha 179

```php id="u6kgpi"
$name = env_default(
    'MOODLE_WS_SERVICE_NAME',
    'W3Soft Student Sync'
);
```

Obtém o nome amigável do serviço.

## Linha 180

```php id="acfcsh"
$shortname = env_default(
    'MOODLE_WS_SERVICE_SHORTNAME',
    'w3soft_student_sync'
);
```

Obtém o identificador técnico do serviço.

É semelhante à diferença entre:

```text id="rydzb2"
Nome: W3Soft Student Sync
Identificador: w3soft_student_sync
```

---

## Linhas 184–188

```php id="6gh96x"
foreach ($functions as $function) {
    if (!$DB->record_exists(
        'external_functions',
        ['name' => $function]
    )) {
        bootstrap_fail(
            "External function does not exist..."
        );
    }
}
```

Percorre todas as funções REST configuradas.

Para cada uma, consulta a tabela de funções externas do Moodle.

Exemplo:

```text id="u5q1zs"
core_user_create_users
```

Se a função não existir, o script para.

Isso detecta:

- erro de digitação;
- função removida;
- plugin necessário ausente;
- incompatibilidade de versão.

Em C#:

```csharp id="0ibuco"
foreach (var function in functions)
{
    var exists = await db.ExternalFunctions
        .AnyAsync(item => item.Name == function);

    if (!exists)
        throw new InvalidOperationException(...);
}
```

---

## Linha 190

```php id="vian6k"
$service = $manager
    ->get_external_service_by_shortname($shortname);
```

Procura um serviço existente pelo identificador técnico.

Se não existir, provavelmente retorna `false` ou valor equivalente.

## Linha 191

```php id="j9ffix"
if (!$service) {
```

Se não encontrou o serviço, entra no fluxo de criação.

## Linha 194

```php id="mhpe4u"
$service = (object)[
```

Cria um objeto genérico com as configurações do serviço.

## Linha 195

```php id="xhjzkj"
'name' => $name,
```

Nome amigável.

## Linha 196

```php id="dicdrx"
'enabled' => 1,
```

Serviço habilitado.

## Linha 197

```php id="ggxlw7"
'requiredcapability' => '',
```

Não exige uma capability global adicional diretamente no serviço.

As permissões serão controladas pelo papel e pelo usuário autorizado.

## Linha 198

```php id="ilwenp"
'restrictedusers' => 1,
```

Determina que somente usuários explicitamente associados ao serviço poderão utilizá-lo.

Isso é mais restritivo do que permitir qualquer usuário com determinada permissão.

## Linha 199

```php id="uoczc0"
'component' => '',
```

Indica que não pertence diretamente a um plugin específico.

## Linha 200

```php id="lo1mqx"
'shortname' => $shortname,
```

Define o identificador técnico.

## Linhas 201–202

```php id="kea6w7"
'downloadfiles' => 0,
'uploadfiles' => 0,
```

Desabilita upload e download de arquivos para esse serviço.

O serviço será utilizado para dados de usuários, cursos e matrículas, e não para transferência geral de arquivos.

## Linha 204

```php id="npqziz"
$service->id = $manager
    ->add_external_service($service);
```

Cria o serviço no Moodle.

O método retorna o ID gerado, que é atribuído ao objeto.

Em C#:

```csharp id="ubudo5"
service.Id = await manager.AddExternalServiceAsync(service);
```

## Linha 205

```php id="wryyxe"
bootstrap_log("Created external service: {$shortname}.");
```

Registra a criação.

---

## Linha 206

```php id="e3r1h6"
} else {
```

Se o serviço já existia, inicia o fluxo de atualização.

## Linhas 209–214

```php id="pl8nv4"
$service->name = $name;
$service->enabled = 1;
$service->restrictedusers = 1;
$service->requiredcapability =
    $service->requiredcapability ?? '';
$service->downloadfiles = 0;
$service->uploadfiles = 0;
```

Normaliza o serviço para o estado desejado.

O operador:

```php id="ax1kyp"
??
```

é o operador de coalescência nula, igual ao C#.

```php id="pjkzuq"
$service->requiredcapability ?? ''
```

significa:

> Utilize o valor atual se ele não for nulo; caso contrário, use string vazia.

Em C#:

```csharp id="mef617"
service.RequiredCapability ?? string.Empty
```

## Linha 215

```php id="pdb4w4"
$manager->update_external_service($service);
```

Persiste as atualizações.

## Linha 216

```php id="oveipj"
bootstrap_log(
    "External service already exists: {$shortname}."
);
```

Registra que o serviço foi encontrado e normalizado.

---

## Linhas 220–225

```php id="03o2bp"
foreach ($functions as $function) {
    if (!$manager->service_function_exists(
        $function,
        $service->id
    )) {
        $manager->add_external_function_to_service(
            $function,
            $service->id
        );

        bootstrap_log(
            "Added function to service: {$function}."
        );
    }
}
```

Percorre cada função REST.

Antes de adicionar, verifica se ela já está associada ao serviço.

Isso evita duplicação.

Exemplo:

```text id="unc05a"
Serviço: w3soft_student_sync
Funções:
  core_user_create_users
  enrol_manual_enrol_users
```

Se uma função já estiver vinculada, ela é ignorada.

Observe que o script **adiciona funções ausentes**, mas não remove funções antigas que deixaram de aparecer na variável de ambiente. Portanto, sua sincronização é aditiva, não uma substituição completa.

## Linha 227

```php id="j40x1d"
return $manager->get_external_service_by_shortname(
    $shortname,
    MUST_EXIST
);
```

Busca novamente e retorna o serviço.

`MUST_EXIST` determina que, nesse momento, a ausência do serviço deve gerar erro. fileciteturn21file0L174-L181 fileciteturn22file0L3-L49

---

# 12. Criação do usuário técnico

## Linha 232

```php id="ii2y1g"
function ensure_ws_user(): stdClass {
```

Cria ou atualiza o usuário técnico que chamará a API REST.

Esse usuário não é uma pessoa. É uma identidade de serviço, semelhante a:

- service account;
- usuário de integração;
- client credential.

## Linha 233

```php id="o8qerd"
global $DB;
```

Acessa o banco.

## Linha 235

```php id="aahgyh"
$username = core_text::strtolower(
    env_required('MOODLE_WS_USER_USERNAME')
);
```

Lê o nome do usuário técnico e o converte para minúsculas.

`core_text::strtolower()` é um método estático do Moodle.

Em C#:

```csharp id="d35y7o"
var username = configuration["MOODLE_WS_USER_USERNAME"]
    .ToLowerInvariant();
```

Utilizar a função do Moodle ajuda no tratamento correto de caracteres Unicode.

## Linha 236

```php id="imhrt8"
$password = env_required('MOODLE_WS_USER_PASSWORD');
```

Lê a senha inicial ou a senha a ser aplicada quando houver reset.

## Linha 237

```php id="1bg3kg"
$existing = $DB->get_record(
    'user',
    [
        'username' => $username,
        'deleted' => 0
    ]
);
```

Procura um usuário ativo com esse nome.

Diferentemente de chamadas anteriores, não há `MUST_EXIST`. Portanto, a ausência do usuário é esperada.

---

## Linha 239

```php id="jy3b3w"
$user = (object)[
```

Cria o objeto de criação ou atualização.

## Linhas 240–249

```php id="i0xkt9"
'username' => $username,
'firstname' => env_required('MOODLE_WS_USER_FIRSTNAME'),
'lastname' => env_required('MOODLE_WS_USER_LASTNAME'),
'email' => env_required('MOODLE_WS_USER_EMAIL'),
'city' => env_required('MOODLE_WS_USER_CITY'),
'country' => env_required('MOODLE_WS_USER_COUNTRY'),
'timezone' => env_required('MOODLE_WS_USER_TIMEZONE'),
'auth' => 'manual',
'confirmed' => 1,
'mnethostid' => 1,
```

### `auth => manual`

O usuário utiliza autenticação manual do Moodle.

### `confirmed => 1`

O cadastro já é considerado confirmado.

Não depende de confirmação por e-mail.

### `mnethostid => 1`

Associa o usuário ao host local padrão do Moodle.

---

## Linha 252

```php id="xphans"
if ($existing) {
```

Se o usuário já existir, entra no fluxo de atualização.

## Linha 255

```php id="g5pcbi"
$user->id = $existing->id;
```

Adiciona o ID do usuário ao objeto.

A partir disso, `user_update_user()` saberá qual registro atualizar.

## Linha 256

```php id="jugotb"
$resetpassword = env_bool(
    'MOODLE_WS_USER_RESET_PASSWORD',
    false
);
```

Verifica se a senha técnica deve ser redefinida.

Por padrão, não.

## Linhas 257–259

```php id="fz3gap"
if ($resetpassword) {
    $user->password = $password;
}
```

Adiciona a senha ao objeto somente quando o reset estiver habilitado.

## Linha 260

```php id="zifhc4"
user_update_user($user, $resetpassword, false);
```

Atualiza o usuário técnico.

- atualiza a senha apenas se `$resetpassword` for verdadeiro;
- não dispara o evento normal de atualização.

## Linha 261

```php id="7n17aw"
bootstrap_log(
    "Technical webservice user already exists: {$username}."
);
```

Registra que o usuário já existia.

## Linha 262

```php id="5ehwpi"
return $DB->get_record(
    'user',
    ['id' => $existing->id],
    '*',
    MUST_EXIST
);
```

Retorna a versão persistida do usuário.

---

## Linha 265

```php id="e09n8j"
$user->password = $password;
```

Caso o usuário ainda não exista, a senha é obrigatória para criá-lo.

## Linha 266

```php id="d6bvqa"
$userid = user_create_user($user, true, false);
```

Cria o usuário.

O retorno é o novo ID.

Conceitualmente:

```csharp id="o07tfb"
var userId = await userService.CreateAsync(
    user,
    updatePassword: true,
    triggerEvent: false);
```

## Linha 267

```php id="rpdamn"
bootstrap_log(
    "Created technical webservice user: {$username}."
);
```

Registra a criação.

## Linha 268

```php id="xjx3yo"
return $DB->get_record(
    'user',
    ['id' => $userid],
    '*',
    MUST_EXIST
);
```

Busca e retorna o usuário completo. fileciteturn22file0L51-L90

---

# 13. Criação do papel de integração

## Linha 273

```php id="mplqqg"
function ensure_ws_role(stdClass $wsuser): int {
```

Recebe o usuário técnico e retorna o ID do papel configurado.

No Moodle, um **papel** é semelhante a uma `Role` em um sistema de autorização:

```text id="fmhekw"
Administrador
Professor
Estudante
W3Soft Integration
```

Uma **capability** é uma permissão específica:

```text id="wyziks"
criar usuário
visualizar curso
matricular estudante
usar REST
```

## Linha 274

```php id="01gocn"
global $DB;
```

Acessa o banco.

## Linhas 276–277

```php id="e3ko6a"
$shortname = env_default(
    'MOODLE_WS_ROLE_SHORTNAME',
    'w3soft_ws_integration'
);

$name = env_default(
    'MOODLE_WS_ROLE_NAME',
    'W3Soft webservice integration'
);
```

Obtém o identificador técnico e o nome amigável do papel.

## Linha 278

```php id="xkdos0"
$description =
    'Role managed by Moodle container bootstrap for REST integrations.';
```

Define uma descrição fixa.

## Linha 279

```php id="jjwyuz"
$systemcontext = context_system::instance();
```

Obtém o contexto global do Moodle.

No Moodle, permissões sempre existem dentro de um contexto. Exemplos:

```text id="l5f1qk"
Sistema inteiro
Categoria
Curso
Atividade
Usuário
```

Nesse caso, o papel técnico será atribuído no nível do sistema inteiro.

Em uma comparação simplificada com C#, seria como atribuir uma role global ao usuário, e não uma permissão apenas para um recurso específico.

---

## Linha 281

```php id="0ta66e"
$role = $DB->get_record(
    'role',
    ['shortname' => $shortname]
);
```

Procura o papel pelo identificador.

## Linha 282

```php id="cy1ze8"
if ($role) {
```

Se já existir, atualiza.

## Linhas 283–285

```php id="1nabnp"
$role->name = $name;
$role->description = $description;
$DB->update_record('role', $role);
```

Sincroniza nome e descrição.

## Linha 286

```php id="g59d80"
$roleid = (int)$role->id;
```

Converte o ID para inteiro.

Em C#:

```csharp id="n563lg"
var roleId = Convert.ToInt32(role.Id);
```

## Linha 287

```php id="4tk2i7"
bootstrap_log(
    "Webservice role already exists: {$shortname}."
);
```

Registra que o papel já existia.

## Linhas 288–291

```php id="hxrca1"
} else {
    $roleid = create_role($name, $shortname, $description);
    bootstrap_log("Created webservice role: {$shortname}.");
}
```

Se o papel não existe, cria e recebe o ID.

---

## Linha 295

```php id="gizc6z"
set_role_contextlevels($roleid, [CONTEXT_SYSTEM]);
```

Define que esse papel pode ser atribuído no contexto de sistema.

O segundo argumento é um array porque um papel poderia aceitar diferentes níveis de contexto.

Aqui aceita somente:

```php id="0l0jhw"
CONTEXT_SYSTEM
```

---

## Linhas 299–310

```php id="tcbq2a"
$capabilities = [
    'webservice/rest:use',
    'moodle/webservice:createtoken',
    'moodle/course:view',
    'moodle/course:viewhiddencourses',
    'moodle/user:create',
    'moodle/user:viewdetails',
    'moodle/user:viewhiddendetails',
    'moodle/course:useremail',
    'moodle/user:update',
    'enrol/manual:enrol',
];
```

Cria a lista padrão de permissões.

### `webservice/rest:use`

Permite utilizar o protocolo REST.

### `moodle/webservice:createtoken`

Permite criar tokens de webservice.

### `moodle/course:view`

Permite visualizar cursos.

### `moodle/course:viewhiddencourses`

Permite visualizar cursos ocultos.

### `moodle/user:create`

Permite criar usuários.

### `moodle/user:viewdetails`

Permite visualizar detalhes de usuários.

### `moodle/user:viewhiddendetails`

Permite visualizar detalhes normalmente ocultos.

### `moodle/course:useremail`

Permite acessar o e-mail dos usuários em contexto de curso.

### `moodle/user:update`

Permite atualizar usuários.

### `enrol/manual:enrol`

Permite realizar matrículas manuais.

Esse conjunto está alinhado às funções REST disponibilizadas para sincronização de estudantes.

---

## Linha 312

```php id="tdsqnp"
$extra = env_default(
    'MOODLE_WS_EXTRA_CAPABILITIES',
    ''
);
```

Permite acrescentar permissões por variável de ambiente.

Exemplo:

```env id="sch1q5"
MOODLE_WS_EXTRA_CAPABILITIES=moodle/course:create,moodle/course:update
```

## Linhas 313–316

```php id="fa9bn6"
if ($extra !== '') {
    $capabilities = array_merge(
        $capabilities,
        split_csv($extra)
    );
}
```

Se houver permissões extras:

1. divide o CSV;
2. combina com a lista padrão.

Em C#:

```csharp id="yc5onp"
if (extra != string.Empty)
{
    capabilities = capabilities
        .Concat(SplitCsv(extra))
        .ToArray();
}
```

---

## Linha 318

```php id="3iswog"
foreach (array_unique($capabilities) as $capability) {
```

Remove duplicidades com `array_unique()` e percorre cada permissão.

## Linha 321

```php id="t11ora"
if (!get_capability_info($capability)) {
```

Verifica se a capability realmente existe na instalação atual.

Se um plugin não estiver instalado ou o nome estiver incorreto, a função não encontrará informações.

## Linha 322

```php id="0y9rr0"
bootstrap_fail(
    "Capability does not exist in this Moodle installation: {$capability}"
);
```

Interrompe o provisionamento em caso de permissão inválida.

## Linha 324

```php id="zza1u7"
assign_capability(
    $capability,
    CAP_ALLOW,
    $roleid,
    $systemcontext->id,
    true
);
```

Atribui a permissão ao papel.

Parâmetros:

1. nome da capability;
2. efeito da permissão, `CAP_ALLOW`;
3. ID do papel;
4. ID do contexto;
5. sobrescrever ou atualizar a atribuição existente.

Conceitualmente:

```csharp id="odudea"
await roleManager.GrantPermissionAsync(
    roleId,
    capability,
    systemContextId);
```

---

## Linhas 329–334

```php id="cay5zp"
if (!$DB->record_exists('role_assignments', [
    'roleid' => $roleid,
    'contextid' => $systemcontext->id,
    'userid' => $wsuser->id,
])) {
```

Verifica se o usuário técnico já recebeu esse papel no contexto global.

A condição evita criar uma atribuição duplicada.

## Linha 334

```php id="cz1iqw"
role_assign(
    $roleid,
    $wsuser->id,
    $systemcontext->id
);
```

Atribui o papel ao usuário.

Isso conecta três elementos:

```text id="8cdwjo"
Usuário técnico
    ↓ recebe
Papel W3Soft Integration
    ↓ no
Contexto global
```

## Linha 335

```php id="h00gzp"
bootstrap_log(
    "Assigned webservice role to user: {$wsuser->username}."
);
```

Registra a atribuição.

---

## Linha 341

```php id="ip5m1g"
$targetshortname = env_default(
    'MOODLE_WS_ENROL_TARGET_ROLE_SHORTNAME',
    'student'
);
```

Define qual papel o usuário técnico poderá atribuir durante matrículas.

Por padrão:

```text id="ggs12a"
student
```

Ou seja, a integração poderá matricular usuários como estudantes.

## Linha 342

```php id="7z7qym"
$targetrole = $DB->get_record(
    'role',
    ['shortname' => $targetshortname],
    '*',
    MUST_EXIST
);
```

Busca o papel de destino.

Se `student` não existir, o provisionamento falha.

## Linha 343

```php id="l469i4"
if (!$DB->record_exists('role_allow_assign', [
    'roleid' => $roleid,
    'allowassign' => $targetrole->id
])) {
```

Verifica se o papel de integração já está autorizado a atribuir o papel de estudante.

Ter a permissão de matrícula não significa automaticamente poder atribuir qualquer papel. Essa relação precisa ser explicitamente autorizada.

## Linha 344

```php id="ya8c7k"
core_role_set_assign_allowed(
    $roleid,
    $targetrole->id
);
```

Autoriza o papel técnico a atribuir o papel de destino.

## Linha 345

```php id="batcft"
bootstrap_log(
    "Allowed webservice role to assign target role: {$targetshortname}."
);
```

Registra a autorização.

---

## Linha 350

```php id="y42om6"
accesslib_clear_all_caches(true);
```

Limpa os caches de permissões do Moodle.

Isso faz com que as alterações de papéis e capabilities sejam reconhecidas imediatamente.

Sem essa limpeza, o Moodle poderia continuar utilizando permissões antigas armazenadas em cache durante algum tempo.

## Linha 351

```php id="jrzylq"
return $roleid;
```

Retorna o ID do papel.

Neste script, o retorno não é utilizado posteriormente, mas pode ser útil para manutenção ou evolução. fileciteturn22file0L92-L173

---

# 14. Autorização do usuário no serviço

## Linha 356

```php id="zil3gs"
function authorize_service_user(
    stdClass $service,
    stdClass $wsuser
): void {
```

Recebe:

- o serviço externo;
- o usuário técnico.

A função associa explicitamente o usuário ao serviço.

Isso é necessário porque o serviço foi criado com:

```php id="1oiyn6"
restrictedusers => 1
```

## Linha 357

```php id="o5cv1u"
global $DB;
```

Acessa o banco.

## Linhas 359–362

```php id="mzy14n"
$record = $DB->get_record(
    'external_services_users',
    [
        'externalserviceid' => $service->id,
        'userid' => $wsuser->id,
    ]
);
```

Procura uma associação existente entre o serviço e o usuário.

É equivalente a uma tabela de relacionamento:

```text id="tzevy5"
external_services_users
--------------------------------
externalserviceid | userid
```

Em EF Core:

```csharp id="fxythz"
var record = await db.ExternalServiceUsers
    .SingleOrDefaultAsync(item =>
        item.ExternalServiceId == service.Id &&
        item.UserId == webServiceUser.Id);
```

## Linha 364

```php id="v49dj8"
$iprestriction = env_default(
    'MOODLE_WS_USER_IP_RESTRICTION',
    ''
);
```

Lê uma possível restrição de IP para o usuário no serviço.

String vazia significa ausência de restrição.

## Linha 365

```php id="pzjek2"
$validuntil = (int)env_default(
    'MOODLE_WS_USER_VALID_UNTIL',
    '0'
);
```

Lê a validade da autorização e converte para inteiro.

Valor `0` normalmente representa ausência de expiração.

---

## Linha 367

```php id="wva5aa"
if ($record) {
```

Se a autorização já existir, atualiza seus parâmetros.

## Linhas 370–371

```php id="eenfm6"
$record->iprestriction = $iprestriction;
$record->validuntil = $validuntil;
```

Sincroniza restrição de IP e validade.

## Linha 372

```php id="7ewm4y"
$DB->update_record(
    'external_services_users',
    $record
);
```

Persiste as mudanças.

## Linha 373

```php id="v089fw"
bootstrap_log(
    "Technical user already authorized for service."
);
```

Registra que a associação já existia.

## Linha 374

```php id="ckoh98"
return;
```

Encerra a função antecipadamente.

Em C#:

```csharp id="0t2urj"
return;
```

---

## Linha 377

```php id="8k0mg3"
$DB->insert_record(
    'external_services_users',
    (object)[
```

Se a associação não existe, cria um novo registro.

## Linhas 378–382

```php id="kku0ut"
'externalserviceid' => $service->id,
'userid' => $wsuser->id,
'iprestriction' => $iprestriction,
'validuntil' => $validuntil,
'timecreated' => time(),
```

Salva:

- serviço;
- usuário;
- restrição de IP;
- validade;
- data da criação.

## Linha 384

```php id="bjinrh"
bootstrap_log("Authorized technical user for service.");
```

Registra a nova autorização. fileciteturn22file0L175-L182 fileciteturn23file0L3-L26

---

# 15. Criação ou reutilização do token

## Linha 389

```php id="rha2kz"
function ensure_token(
    stdClass $service,
    stdClass $wsuser,
    stdClass $admin
): string {
```

Recebe:

- serviço externo;
- usuário técnico;
- administrador.

Retorna o token como string.

## Linha 390

```php id="yxmxl8"
global $DB;
```

Acessa o banco.

## Linha 392

```php id="0dpra3"
$now = time();
```

Obtém o timestamp atual.

---

## Linha 393

```php id="2j9i0s"
$token = $DB->get_record_sql(
```

Executa uma consulta SQL customizada usando a API de banco do Moodle.

## Linhas 394–400

```sql id="bc6cb9"
SELECT *
FROM {external_tokens}
WHERE userid = :userid
  AND externalserviceid = :serviceid
  AND tokentype = :tokentype
  AND (
      validuntil IS NULL
      OR validuntil = 0
      OR validuntil > :now
  )
ORDER BY id ASC
```

### `{external_tokens}`

As chaves são uma convenção do Moodle.

O Moodle substitui:

```text id="zp2739"
{external_tokens}
```

pelo nome físico com o prefixo configurado, por exemplo:

```text id="swbts6"
mdl_external_tokens
```

### `userid = :userid`

O token deve pertencer ao usuário técnico.

### `externalserviceid = :serviceid`

Deve pertencer ao serviço W3Soft.

### `tokentype = :tokentype`

Deve ser do tipo permanente.

### Validade

O token é aceito quando:

- `validuntil` é nulo;
- ou é zero;
- ou sua expiração está no futuro.

### `ORDER BY id ASC`

Caso existam vários tokens, prioriza o mais antigo.

---

## Linhas 401–406

```php id="s98bxl"
[
    'userid' => $wsuser->id,
    'serviceid' => $service->id,
    'tokentype' => EXTERNAL_TOKEN_PERMANENT,
    'now' => $now,
],
```

Fornece os valores dos parâmetros nomeados.

Isso evita concatenar valores diretamente no SQL.

Em C# com Dapper, seria algo como:

```csharp id="8nipqf"
connection.QueryFirstOrDefault<Token>(
    sql,
    new
    {
        userid = wsUser.Id,
        serviceid = service.Id,
        tokentype = ExternalTokenPermanent,
        now
    });
```

## Linha 407

```php id="jzchxx"
IGNORE_MULTIPLE
```

Instrui a API do Moodle a tolerar múltiplos resultados e utilizar um registro.

Sem isso, uma consulta que encontrasse vários tokens poderia lançar uma exceção por esperar apenas um.

---

## Linha 410

```php id="98schy"
if ($token) {
```

Se encontrou um token válido, reutiliza-o.

## Linha 413

```php id="imqc5m"
bootstrap_log(
    "Reusing active webservice token for service/user."
);
```

Registra a reutilização.

## Linha 414

```php id="bicg74"
return $token->token;
```

Retorna o valor do token existente.

Isso evita trocar a credencial a cada inicialização do container.

Caso o token mudasse a cada reinicialização, a aplicação externa deixaria de autenticar até receber a nova credencial.

---

## Linha 419

```php id="omj8si"
$tokenvalue = md5(
    uniqid(
        (string)random_int(0, PHP_INT_MAX),
        true
    )
);
```

Gera o valor do novo token.

A expressão tem várias camadas.

### `random_int(0, PHP_INT_MAX)`

Gera um inteiro aleatório.

### `(string)`

Converte esse inteiro para texto.

### `uniqid(..., true)`

Cria um identificador baseado no prefixo recebido e no tempo, com entropia adicional.

### `md5(...)`

Converte o resultado para uma string hexadecimal de 32 caracteres.

Exemplo:

```text id="43z8pw"
d41d8cd98f00b204e9800998ecf8427e
```

Aqui o MD5 não está sendo utilizado para armazenar senha, mas para produzir o formato final do token.

Mesmo assim, do ponto de vista arquitetural, a criação manual de um token e sua inserção direta na tabela acoplam o script à implementação interna do Moodle. Uma implementação futura poderia preferir uma API oficial do próprio Moodle para criar o token ou utilizar bytes aleatórios criptograficamente seguros de forma mais direta.

## Linha 420

```php id="crqhet"
$validuntil = (int)env_default(
    'MOODLE_WS_TOKEN_VALID_UNTIL',
    '0'
);
```

Obtém a validade do token.

`0` significa token sem expiração configurada.

---

## Linha 422

```php id="pj2tfm"
$DB->insert_record(
    'external_tokens',
    (object)[
```

Insere o token diretamente na tabela de tokens externos.

## Linha 423

```php id="dtym3b"
'token' => $tokenvalue,
```

Valor público usado nas chamadas REST.

## Linha 424

```php id="zf60vf"
'privatetoken' => random_string(64),
```

Gera um token privado adicional de 64 caracteres.

## Linha 425

```php id="q17cdt"
'tokentype' => EXTERNAL_TOKEN_PERMANENT,
```

Define o token como permanente.

## Linha 426

```php id="z9tucl"
'userid' => $wsuser->id,
```

Associa ao usuário técnico.

## Linha 427

```php id="dr95vu"
'externalserviceid' => $service->id,
```

Associa ao serviço W3Soft.

## Linha 428

```php id="n6joug"
'sid' => null,
```

Não associa o token a uma sessão web específica.

## Linha 429

```php id="c7ajur"
'contextid' => context_system::instance()->id,
```

Associa o token ao contexto global.

## Linha 430

```php id="f8nhx7"
'creatorid' => $admin->id,
```

Registra o administrador como criador do token.

Isso fornece rastreabilidade no banco.

## Linha 431

```php id="au2wax"
'iprestriction' => env_default(
    'MOODLE_WS_TOKEN_IP_RESTRICTION',
    ''
),
```

Permite restringir o token a um IP ou faixa configurada.

## Linha 432

```php id="fqy5lf"
'validuntil' => $validuntil,
```

Define a expiração.

## Linha 433

```php id="74k5y9"
'timecreated' => $now,
```

Registra a criação.

## Linha 434

```php id="emdnur"
'lastaccess' => null,
```

O token ainda nunca foi utilizado.

## Linha 435

```php id="wfbtps"
'name' => env_default(
    'MOODLE_WS_TOKEN_NAME',
    'W3Soft bootstrap token'
),
```

Define um nome identificável para o token.

## Linha 438

```php id="hujevq"
bootstrap_log(
    "Created new webservice token for service/user."
);
```

Registra a criação.

## Linha 439

```php id="2vp0o5"
return $tokenvalue;
```

Retorna o token recém-criado. fileciteturn23file0L28-L81

---

# 16. Gravação do token em arquivo

## Linha 444

```php id="mqqb5f"
function write_token_file(string $token): void {
```

Recebe o token e o grava em um arquivo.

A finalidade é permitir que outro processo leia a credencial sem acessar diretamente o banco do Moodle.

## Linha 445

```php id="usyt4j"
$tokenfile = env_default(
    'MOODLE_WS_TOKEN_FILE',
    '/var/www/moodledata/w3soft/ws-token.txt'
);
```

Obtém o caminho do arquivo.

Por padrão:

```text id="vd4apg"
/var/www/moodledata/w3soft/ws-token.txt
```

Como `moodledata` é um volume persistente, o arquivo continua existindo mesmo se o container for recriado.

---

## Linha 449

```php id="ul9x9h"
if (!str_starts_with($tokenfile, '/')) {
```

Verifica se o caminho começa com `/`.

Em Linux, isso indica caminho absoluto.

Exemplos:

```text id="pqrdqa"
/var/www/token.txt     absoluto
token.txt              relativo
./token.txt            relativo
```

## Linha 450

```php id="hyw436"
bootstrap_fail(
    'MOODLE_WS_TOKEN_FILE must be an absolute path.'
);
```

Interrompe o script se o caminho for relativo.

Isso evita que o arquivo seja criado em um diretório inesperado.

---

## Linha 453

```php id="uf0knj"
$directory = dirname($tokenfile);
```

Extrai o diretório do caminho.

Exemplo:

```text id="flaafy"
Arquivo:
/var/www/moodledata/w3soft/ws-token.txt

Diretório:
/var/www/moodledata/w3soft
```

## Linha 454

```php id="yexbuv"
if (
    !is_dir($directory)
    && !mkdir($directory, 0700, true)
    && !is_dir($directory)
) {
```

Essa condição tenta garantir que o diretório exista.

### `!is_dir($directory)`

O diretório ainda não existe.

### `mkdir($directory, 0700, true)`

Tenta criá-lo.

Parâmetros:

- caminho;
- permissão `0700`;
- `true` para criar diretórios intermediários.

### Segundo `!is_dir($directory)`

Confirma novamente que o diretório continua inexistente.

Essa verificação adicional ajuda em situações de concorrência. Outro processo poderia criar o diretório entre a primeira verificação e o `mkdir()`.

A condição completa significa:

> Se o diretório não existe, a tentativa de criação falhou e ele ainda continua inexistente, então há um erro real.

## Linha 455

```php id="7j2la7"
bootstrap_fail(
    "Could not create token directory: {$directory}"
);
```

Encerra em caso de falha.

---

## Linha 460

```php id="sbtusz"
chmod($directory, 0700);
```

Define permissões Unix para o diretório.

`0700` significa:

```text id="qbny2z"
Dono: leitura, escrita e execução
Grupo: nenhuma permissão
Outros: nenhuma permissão
```

Representação:

```text id="sy60xo"
rwx------
```

A permissão de execução em diretório significa poder entrar ou atravessar o diretório.

---

## Linha 462

```php id="yhzwme"
if (
    file_put_contents(
        $tokenfile,
        $token . PHP_EOL,
        LOCK_EX
    ) === false
) {
```

Grava o token no arquivo.

### `$token . PHP_EOL`

Adiciona uma quebra de linha ao final.

### `LOCK_EX`

Solicita bloqueio exclusivo durante a escrita.

Isso reduz o risco de dois processos escreverem no mesmo arquivo simultaneamente.

### `=== false`

`file_put_contents()` retorna:

- quantidade de bytes gravados em caso de sucesso;
- `false` em caso de erro.

É necessário comparar estritamente com `false`, porque gravar zero bytes poderia retornar `0`, que também é considerado falso em uma comparação não estrita.

## Linha 463

```php id="7rgmj8"
bootstrap_fail(
    "Could not write webservice token file: {$tokenfile}"
);
```

Interrompe se a gravação falhar.

## Linha 466

```php id="p8ffjj"
chmod($tokenfile, 0600);
```

Define permissões para o arquivo:

```text id="ya7q56"
Dono: leitura e escrita
Grupo: nenhuma
Outros: nenhuma
```

Representação:

```text id="opy0ax"
rw-------
```

Isso é importante porque o arquivo contém uma credencial capaz de acessar a API.

## Linha 467

```php id="r522dc"
bootstrap_log(
    "Webservice token persisted at: {$tokenfile}"
);
```

Registra o caminho, mas não imprime o token.

Não registrar o valor do token nos logs é uma boa prática, pois logs podem ser acessados por mais pessoas e ferramentas. fileciteturn23file0L83-L109

---

# 17. Fluxo principal do script

Até aqui, o arquivo apenas declarou funções. A execução efetiva começa no final.

Isso é parecido com declarar classes e métodos em C# e depois iniciar a aplicação no `Main`.

## Linha 472

```php id="cv3576"
$firstinstall = env_bool(
    'MOODLE_BOOTSTRAP_FIRST_INSTALL',
    false
);
```

Lê se esta execução ocorreu logo após a primeira instalação.

Essa variável é exportada anteriormente pelo `docker-entrypoint.sh`.

## Linha 474

```php id="90f02j"
bootstrap_log('Starting tenant provisioning.');
```

Registra o início do provisionamento da instituição.

## Linha 475

```php id="xzmqeh"
update_site_identity();
```

Atualiza:

- nome completo;
- nome curto;
- descrição;
- e-mail de suporte;
- fuso horário.

## Linha 476

```php id="5ok2l5"
$admin = update_admin_user($firstinstall);
```

Configura o administrador e guarda o objeto retornado.

O administrador será utilizado como criador do token.

## Linha 477

```php id="gudy8y"
ensure_webservice_settings();
```

Habilita:

- webservices;
- protocolo REST.

---

## Linha 480

```php id="kl1wfn"
$functions = split_csv(
    env_default(
        'MOODLE_WS_FUNCTIONS',
        'core_webservice_get_site_info,...'
    )
);
```

Lê a lista de funções REST.

Caso a variável não exista, usa a lista padrão:

```text id="eo2ez3"
core_webservice_get_site_info
core_course_get_courses
core_course_get_courses_by_field
core_user_get_users_by_field
core_user_create_users
enrol_manual_enrol_users
```

Depois transforma o CSV em array.

Em C#:

```csharp id="q98rvb"
var functions = SplitCsv(
    GetEnvironmentVariableOrDefault(
        "MOODLE_WS_FUNCTIONS",
        "core_webservice_get_site_info,..."));
```

## Linha 481

```php id="861klx"
$service = ensure_service($functions);
```

Cria ou atualiza o serviço REST e associa as funções.

## Linha 482

```php id="js7ws2"
$wsuser = ensure_ws_user();
```

Cria ou atualiza o usuário técnico.

## Linha 483

```php id="gymzin"
ensure_ws_role($wsuser);
```

Cria o papel, configura as permissões e o atribui ao usuário.

O retorno do ID do papel é ignorado porque não será necessário no restante do script.

Em C#, também é permitido ignorar um retorno:

```csharp id="bfpqsa"
EnsureWebServiceRole(webServiceUser);
```

## Linha 484

```php id="p1sl4m"
authorize_service_user($service, $wsuser);
```

Autoriza explicitamente o usuário técnico a utilizar o serviço restrito.

## Linha 485

```php id="my9y0r"
$token = ensure_token(
    $service,
    $wsuser,
    $admin
);
```

Procura um token válido.

- se existir, reutiliza;
- se não existir, cria.

## Linha 486

```php id="yn23o7"
write_token_file($token);
```

Grava o token no volume persistente.

## Linha 487

```php id="im4cly"
bootstrap_log('Tenant provisioning finished.');
```

Registra o fim bem-sucedido do provisionamento. fileciteturn23file0L111-L128

---

# Fluxo completo em linguagem de C#

O comportamento geral poderia ser representado assim:

```csharp id="azrfx7"
public static void Main()
{
    var firstInstall = GetBooleanEnvironmentVariable(
        "MOODLE_BOOTSTRAP_FIRST_INSTALL",
        false);

    Log("Starting tenant provisioning.");

    UpdateSiteIdentity();

    var admin = UpdateAdminUser(firstInstall);

    EnsureWebServiceSettings();

    var functions = SplitCsv(
        GetEnvironmentVariableOrDefault(
            "MOODLE_WS_FUNCTIONS",
            DefaultFunctions));

    var service = EnsureService(functions);

    var integrationUser = EnsureWebServiceUser();

    EnsureWebServiceRole(integrationUser);

    AuthorizeServiceUser(service, integrationUser);

    var token = EnsureToken(
        service,
        integrationUser,
        admin);

    WriteTokenFile(token);

    Log("Tenant provisioning finished.");
}
```

# Relação entre os objetos criados

```text id="nwu1rf"
Serviço REST
W3Soft Student Sync
        │
        ├── contém funções REST
        │     ├── consultar cursos
        │     ├── consultar usuários
        │     ├── criar usuários
        │     └── matricular estudantes
        │
        └── autoriza o usuário técnico
                    │
                    └── recebe o papel
                        W3Soft Integration
                              │
                              └── contém capabilities
```

O token conecta:

```text id="45jeez"
Token
 ├── Usuário técnico
 ├── Serviço REST
 ├── Contexto global
 └── Administrador que o criou
```

# Principal diferença para uma aplicação C# convencional

Em uma aplicação .NET estruturada, provavelmente existiriam classes como:

```text id="md52ug"
TenantProvisioningService
SiteConfigurationService
MoodleUserService
WebServiceConfigurationService
TokenService
```

E dependências seriam injetadas:

```csharp id="ttzmmd"
public TenantProvisioningService(
    IMoodleDatabase database,
    IConfiguration configuration,
    ILogger<TenantProvisioningService> logger)
```

Neste script PHP:

- `$DB` é global;
- `$CFG` é global;
- funções procedurais executam as operações;
- objetos `stdClass` funcionam como DTOs dinâmicos;
- a ordem das chamadas no final funciona como o método `Main`.

A abordagem é menos orientada a objetos que uma aplicação C# típica, mas é adequada para um script curto de provisionamento executado no startup do container.
