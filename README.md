# FlatFiles <img src="icon.png" alt="FlatFiles Logo" width="50"/>

Lê e grava CSV, tamanho fixo e outros formatos de arquivo simples com foco na definição, configuração e velocidade do esquema. Suporta mapeamento diretamente entre arquivos e classes.

Baixe usando NuGet: [FlatFiles] (http://nuget.org/packages/FlatFiles)


## Visão geral
Os formatos de texto simples vêm principalmente em duas variações: delimitado (CSV, TSV, etc.) e de largura fixa. FlatFiles vem com suporte para trabalhar com ambos os formatos. Ao contrário da maioria das outras bibliotecas, FlatFiles concentra-se na definição do esquema. Você constrói e passa um esquema para um leitor ou escritor e ele usará o esquema para extrair ou gravar seus valores.

Um esquema é definido especificando quais colunas de dados estão em seu arquivo. Uma coluna possui um nome, um tipo e uma posição ordinal no arquivo. A ordem corresponde a qualquer ordem em que você adiciona as colunas ao esquema, portanto, resta apenas especificar o nome e o tipo. Além disso, você tem muito controle sobre o comportamento de análise / formatação ao ler e escrever, respectivamente. Na maioria das vezes, as opções prontas para usar * funcionarão * também. Mas quando você precisa desse nível de controle extra, não precisa se dobrar para trás para contornar a API, como acontece com muitas outras bibliotecas. FlatFiles foi projetado para tornar o tratamento de casos extremos excêntricos mais fácil.

Se você estiver trabalhando com classes de dados, definir esquemas é ainda mais fácil. Você pode usar os mapeadores de tipo para mapear suas propriedades diretamente. Isso evita que você tenha que especificar nomes ou tipos de coluna, uma vez que ambos podem ser derivados da propriedade. Para aqueles que trabalham com ADO.NET, há até suporte para `DataTable`s e` IDataReader`. Possível ler e escrever valores usando `object []` diretamente.

## Índice
* [Visão geral] (# visão geral)
* [Mapeadores de tipo] (# mapeadores de tipo)
    * [Mapeamento automático] (# mapeamento automático)
* [Esquemas] (# esquemas)
* [Arquivos delimitados] (# arquivos delimitados)
* [Arquivos de comprimento fixo] (# arquivos de comprimento fixo)
* [Tratamento de nulos] (# tratamento de nulos)
    * [Valores padrão] (# valores padrão)
    * [Formatadores nulos] (# formatadores nulos)
* [Campos ignorados] (# campos ignorados)
* [Metadados] (# metadados)
    * [Escrevendo metadados] (# escrevendo metadados)
    * [Crie seus próprios metadados] (# criar suas próprias colunas de metadados)
* [Skipping Records] (# skipping-records)
* [Tratamento de erros] (# tratamento de erros)
* [Arquivos contendo vários esquemas] (# arquivos contendo vários esquemas)
* [Mapeamento personalizado] (# mapeamento personalizado)
* [Mapeamento em tempo de execução] (# mapeamento em tempo de execução)
* [Disabling Optimization] (# disabling-optimization)
* [Classes e membros não públicos] (# classes e membros não públicos)
* [ADO.NET DataTables] (# adonet-datatables)
* [FlatFileDataReader] (# flatfiledatareader)
* [Licença] (# licença)

## Type Mappers
Usando os mapeadores de tipo, você pode ler diretamente o conteúdo do arquivo em suas classes:

```csv
customer_id,name,created,avg_sales
1,bob,20120321,12.34
2,Susan,20130108,13.88
3,Tom,20180519,88.23
```

```csharp
var mapper = SeparatedValueTypeMapper.Define<Customer>();
mapper.Property(c => c.CustomerId).ColumnName("customer_id");
mapper.Property(c => c.Name).ColumnName("name");
mapper.Property(c => c.Created).ColumnName("created").InputFormat("yyyyMMdd");
mapper.Property(c => c.AverageSales).ColumnName("avg_sales");
using (var reader = new StreamReader(File.OpenRead(@"C:\path\to\file.csv")))
{
    var options = new SeparatedValueOptions() { IsFirstRecordSchema = true };
    var customers = mapper.Read(reader, options).ToList();
}
```

Para definir o esquema ao trabalhar com o tipo 'mapeadores', chame `Property` na ordem em que os campos aparecem no arquivo. O tipo da coluna é determinado pelo tipo da propriedade mapeada. Cada configuração de propriedade oferece opções para controlar a maneira como FlatFiles lida com strings, números, data / horas, GUIDs, enums e muito mais. Uma vez que as propriedades são configuradas, você pode chamar `Read` ou` Write` no mapeador de tipo.

* Nota * O método `Read` só recupera registros do arquivo subjacente sob demanda. Para trazer o arquivo inteiro para a memória de uma vez, basta chamar `ToList` ou` ToArray`, ou fazer um loop sobre os registros dentro de um `foreach`. Esta é uma boa notícia para quem trabalha com arquivos enormes!

Gravar em um arquivo:

```csharp
mapper.Property(c => c.Created).OutputFormat("yyyyMMdd");
mapper.Property(c => c.AverageSales).OutputFormat("N2");
using (var writer = new StreamWriter(File.OpenCreate(@"C:\path\to\file2.csv")))
{
    var options = new SeparatedValueOptions() { IsFirstRecordSchema = true };
    mapper.Write(writer, customers, options);
}
```

*Nota* É possível personalizar o `OutputFormat` das propriedades que foram configuradas anteriormente. A primeira vez que `Property` é chamado em uma propriedade, FlatFiles assume que é a próxima coluna a aparecer no arquivo simples. No entanto, a configuração subsequente na propriedade não altera a ordem das colunas nem redefine quaisquer outras configurações.

### Mapeamento automático
Se seu arquivo delimitado (CSV, TSV, etc.) tem um esquema com nomes de coluna que correspondem aos nomes de propriedade de sua classe, você pode usar o método `GetAutoMappedReader` como um atalho. Este método irá ler o esquema do seu arquivo e mapear as colunas para as propriedades automaticamente, retornando um leitor para recuperar os dados. É importante observar que você não pode personalizar o comportamento de análise de nenhuma das colunas, ponto em que é melhor definir explicitamente o esquema. Felizmente, o FlatFiles usa uma análise bastante liberal pronta para uso, portanto, a maioria dos formatos comuns funcionam.

Por padrão, as colunas e propriedades são correspondidas por nome (não diferencia maiúsculas de minúsculas). Se você precisa de mais controle sobre como as colunas e propriedades são combinadas, você pode passar seu próprio `IAutoMapMatcher`. Dado um `IColumnDefinition` e um` MemberInfo`, um matcher deve determinar se os dois mapeiam um ao outro. Por conveniência, você também pode usar o método `AutoMapMatcher.For` para passar um delegado` Func <IColumnDefinition, MemberInfo, bool> `em vez de implementar a interface.

Da mesma forma, use o método `GetAutoMappedWriter` para escrever automaticamente um arquivo delimitado. Observe que não há como controlar a formatação da coluna. No entanto, você pode controlar o nome e a posição das colunas passando um `IAutoMapResolver`. A interface `IAutoMapResolver` fornece os métodos` GetPosition` e `GetColumnName`, ambos aceitando` MemberInfo`. Por conveniência, você também pode usar o método `AutoMapResolver.For` para passar delegados para determinar os nomes / posições, em vez de implementar a interface.

## Schemas
Nos bastidores, o mapeamento de tipo define internamente um esquema, dando a cada coluna um nome, ordem e tipo no arquivo simples. Você pode obter acesso ao esquema chamando `GetSchema` no mapeador.

Você pode trabalhar diretamente com esquemas se não planeja usar os mapeadores de tipo. Por exemplo, é assim que definiríamos um esquema de arquivo CSV:

```csharp
var schema = new SeparatedValueSchema();
schema.AddColumn(new Int64Column("customer_id"))
      .AddColumn(new StringColumn("name"))
      .AddColumn(new DateTimeColumn("created") { InputFormat = "yyyyMMdd", OutputFormat = "yyyyMMdd" })
      .AddColumn(new DoubleColumn("avg_sales") { OutputFormat = "N2" });
```

Ou, se o esquema for para um arquivo de comprimento fixo:

```csharp
var schema = new FixedLengthSchema();
schema.AddColumn(new Int64Column("customer_id"), 10)
  .AddColumn(new StringColumn("name"), 255)
  .AddColumn(new DateTimeColumn("created") { InputFormat = "yyyyMMdd", OutputFormat = "yyyyMMdd" }, 8)
  .AddColumn(new DoubleColumn("avg_sales") { OutputFormat = "N2" }, 10);
```

A classe `FixedLengthSchema` é igual à classe` SeparatedValueSchema`, exceto que associa uma `Window` a cada coluna. Uma `janela` registra a` largura` da coluna no arquivo. Também permite que você especifique o `Alinhamento` (à esquerda ou à direita) nos casos em que o valor não preenche toda a largura da coluna (o padrão é alinhado à esquerda). A propriedade `FillCharacter` pode ser usada para dizer qual caractere é usado como preenchimento. Você também pode definir o `TruncationPolicy` para dizer se deve cortar a frente ou o verso dos valores que excedem sua largura.

*Nota* Alguns arquivos de comprimento fixo podem ter colunas que não são usadas. O esquema de comprimento fixo não fornece uma maneira de especificar um índice inicial para uma coluna. Veja a seção [Ignored Fields] (# Ignored% 20Fields) abaixo para aprender sobre as maneiras de lidar com isso.

## Arquivos Delimitados
Se você estiver trabalhando com arquivos delimitados, como arquivos separados por vírgula (CSV) ou separados por tabulação (TSV), você deseja usar `SeparatedValueTypeMapper`. Internamente, o mapeador usa as classes `SeparatedValueReader` e` SeparatedValueWriter`, ambas as quais funcionam em termos de matrizes de objetos brutos. Na verdade, tudo o que o mapeador faz é mapear os valores na matriz para as propriedades em seus objetos de dados. Essas classes lêem dados de um `TextReader`, como` StreamReader` ou `StringReader`, e gravam dados em um` TextWriter`, como `StreamWriter` ou` StringWriter`. Internamente, o mapeador construirá um `SeparatedValueSchema` com base na configuração da propriedade / coluna; é aqui que você personaliza o esquema para corresponder ao formato do arquivo. Para configurações mais globais, há também um objeto `SeparatedValueOptions` que permite personalizar o comportamento de leitura / gravação para atender às suas necessidades.

Em arquivos de valores separados, os campos podem ser colocados entre aspas duplas (`" `). Dessa forma, eles podem incluir o separador dentro do campo. Você pode substituir o caractere" aspas "na classe` SeparatedValueOptions`, se necessário. As `SeparatedValueOptions `class suporta uma propriedade` Separator` para especificar a string / caractere que separa seus campos. Uma vírgula (`,`) é o separador padrão; você obviamente deseja alterar isso para tab (`\ t`) para um TSV Arquivo.

A propriedade `RecordSeparator` especifica qual string / caractere é usado para separar registros. Por padrão, FlatFiles irá procurar por `\ r`,` \ n` ou `\ r \ n`, que são os separadores de linha padrão para Mac, Linux e Windows, respectivamente. Ao gravar arquivos, `Environment.NewLine` é usado por padrão; isso significa que, por padrão, você obterá resultados diferentes se executar o mesmo código em plataformas diferentes. Se você precisa direcionar uma plataforma específica, certifique-se de definir a propriedade `RecordSeparator` explicitamente.

Ao trabalhar diretamente com a classe `SeparatedValueReader`, a opção` IsFirstRecordSchema` diz ao leitor para tratar o primeiro registro no arquivo como o esquema. Isso é útil quando você não conhece o esquema com antecedência ou está apenas despejando arquivos em tabelas de preparação. Se você fornecer um esquema, esta configuração instrui o leitor a simplesmente pular o primeiro registro. Se você estiver usando para determinar o esquema do arquivo, o leitor trata cada coluna como uma `string` - ele não tenta interpretar o tipo dos dados. Embora ocasionalmente útil, você normalmente desejará fornecer um esquema para fornecer o máximo de controle sobre o processo de análise - é nisso que o FlatFiles é bom!

Ao trabalhar diretamente com a classe `SeparatedValueWriter`, definir` IsFirstRecordSchema` para a opção `true` faz com que um cabeçalho seja escrito no arquivo ao gravar o primeiro registro.

## Arquivos de comprimento fixo
Se você tiver um arquivo com colunas de comprimento fixo, deverá usar a classe `FixedLengthTypeMapper`. Internamente, o mapeador usa as classes `FixedLengthReader` e` FixedLengthWriter`, ambas funcionam em termos de matrizes de objetos brutos. Na verdade, tudo o que o mapeador faz é mapear os valores na matriz para as propriedades em seus objetos de dados. Essas classes lêem dados de um `TextReader`, como` StreamReader` ou `StringReader`, e gravam dados em um` TextWriter`, como `StreamWriter` ou` StringWriter`. Internamente, o mapeador construirá um `FixedLengthSchema` com base na configuração da propriedade / coluna; é aqui que você personaliza o esquema para corresponder ao formato do arquivo. Para configurações mais globais, há também um objeto `FixedLengthOptions` que permite personalizar o comportamento de leitura / gravação para atender às suas necessidades.

Como cada coluna tem um comprimento fixo, FlatFiles oferece opções de configuração para especificar como lidar com valores muito curtos ou muito longos: `FillCharacter`,` Alignment` e `TruncationPolicy`. `FillCharacter` especifica qual caractere é usado para preencher os valores à esquerda ou à direita, usando espaço (` `) por padrão. Você pode configurar se o preenchimento deve ir para a esquerda ou para a direita usando a propriedade `Alignment`, colocando o preenchimento à direita por padrão (` LeftAligned`). `TruncationPolicy` diz ao FlatFiles como cortar os valores que excedem a largura de sua coluna ao gravar em um arquivo, removendo os caracteres iniciais por padrão. Essas opções podem ser especificadas globalmente no objeto `FixedLengthOptions` ou substituídas no nível da coluna usando o objeto` Window`.

A propriedade `RecordSeparator` especifica qual string / caractere é usado para separar registros. Por padrão, FlatFiles irá procurar por `\ r`,` \ n` ou `\ r \ n`, que são os separadores de linha padrão para Mac, Linux e Windows, respectivamente. Ao gravar arquivos, `Environment.NewLine` é usado por padrão; isso significa que, por padrão, você obterá resultados diferentes se executar o mesmo código em plataformas diferentes. Se você precisa direcionar uma plataforma específica, certifique-se de definir a propriedade `RecordSeparator` explicitamente.

Por padrão, FlatFiles assume que há uma string / caractere separador entre cada registro. Se você definir `HasRecordSeparator` como` false`, FlatFiles lerá o próximo registro imediatamente após o último caractere do registro anterior. Ao escrever, não irá inserir um separador, escrevendo imediatamente após o último caractere do registro anterior.

Se a propriedade `IsFirstRecordHeader` de` FixedLengthOptions` estiver definida como `true`, o primeiro registro no arquivo será ignorado durante a leitura. Ao contrário do `SeparatedValueReader`, você deve * sempre fornecer um esquema para arquivos de comprimento fixo *, uma vez que a largura das colunas não pode ser determinada a partir do formato do arquivo. Ao gravar, um cabeçalho será gravado no arquivo ao gravar o primeiro registro.

## Tratamento de nulos
Cada coluna pode ser marcada como "anulável", usando a propriedade `IsNullable`. Por padrão, todas as colunas podem ser anuladas, o que significa que `nulo` é considerado um valor válido. Definir `IsNullable` como` false` fará com que FlatFiles lance uma exceção sempre que um `null` for encontrado.

Os mapeadores de tipo configurarão automaticamente as colunas para serem anuláveis ​​com base no tipo da propriedade. Se você precisa suportar nulos, certifique-se de usar tipos anuláveis ​​para suas propriedades, por exemplo `int?` Em vez de `int`.

### Valores padrão
Ao trabalhar com colunas não anuláveis, você pode especificar um valor padrão a ser usado sempre que um `nulo` for encontrado, em vez de lançar uma exceção. Você simplesmente define um `IDefaultValue` personalizado na coluna. A classe `DefaultValue` fornece métodos auxiliares para gerar valores padrão. Por exemplo:

```csharp
column.DefaultValue = DefaultValue.Use(0);
```

Ou, se estiver usando mapeamentos de tipo, você pode simplesmente usar o método `DefaultValue`:

```csharp
mapper.Property(c => c.Amount)
    .ColumnName("amount")
    .DefaultValue(DefaultValue.Use(0));
```

### Formatadores nulos
Por padrão, FlatFiles tratará strings em branco ou vazias como `null`. Se os nulos forem representados de forma diferente em seu arquivo, você pode definir um `INullFormatter` personalizado na coluna. Se for um valor fixo, você pode usar o método `NullFormatter.ForValue`.

```csharp
var dateColumn = new DateTimeColumn("created") 
{
    NullFormatter = NullFormatter.ForValue("NULL") 
};
```

Ou, se estiver usando mapeadores de tipo, você pode simplesmente usar o método `NullFormatter`:

```csharp
mapper.Property(c => c.Created)
    .ColumnName("created")
    .NullFormatter(NullFormatter.ForValue("NULL"));
```

Você pode implementar a interface `INullFormatter` se precisar suportar algo mais complexo.



## Campos Ignorados
Na maioria das vezes, não temos controle sobre o arquivo simples com o qual estamos trabalhando. Isso geralmente leva a colunas com as quais nosso código simplesmente não se importa. Os mapeadores de tipo e os esquemas esperam que as colunas sejam listadas na ordem em que aparecem no documento. Para mapeadores de tipo, isso significaria definir propriedades em suas classes que você nunca usaria (por exemplo, Ignored1, Ignored2, etc.). Outro problema comum com arquivos de comprimento fixo é que eles separam os campos com barras verticais (`|`) ou outros caracteres, embora tenham comprimento fixo.

Os mapeadores de tipos suportam o método `Ignored` que dirá ao FlatFiles para simplesmente ignorar a próxima coluna no arquivo. Ao contrário de outros métodos de mapeamento, ele não leva uma propriedade, uma vez que o valor é simplesmente descartado. Você pode opcionalmente chamar `ColumnName` se quiser fornecer um nome de coluna ao escrever esquemas.

Ao trabalhar com o `IFixedLengthTypeMapper`,` Ignored` leva uma `janela`. Esta é uma ótima maneira de pular as seções não utilizadas do documento. Você pode até definir a propriedade `FillCharacter` para inserir barras verticais (` | `), ou outro caractere, entre os campos.

Sob o capô, mapeadores de tipo estão adicionando um `IgnoredColumn` ao esquema subjacente. `IgnoredColumn` tem um construtor para especificar opcionalmente um nome de coluna. `IgnoredColumn` afeta a maneira como você trabalha com leitores e escritores. Os leitores irão inicialmente recuperar todas as colunas do documento e, em seguida, jogar fora quaisquer valores ignorados. Você só verá valores para as colunas que não são ignoradas. Além disso, você não precisa fornecer valores aos gravadores para colunas ignoradas; o esquema automaticamente se encarregará de escrever espaços em branco para eles. Do ponto de vista do desenvolvimento, é como se essas colunas não existissem no documento subjacente.

## Metadados
Freqüentemente, é útil incorporar metadados aos registros que você está lendo de um arquivo. O exemplo mais comum é rastrear o número da linha de um registro, para que os usuários possam ser informados onde procurar em seus arquivos quando algo der errado.

Atualmente, a única coluna de metadados pronta para uso é `RecordNumberColumn`; no entanto, é realmente fácil criar suas próprias colunas de metadados personalizados (mais sobre isso abaixo).

A classe `RecordNumberColumn` fornece opções para controlar como o número do registro é gerado. A propriedade `IncludeSchema` indica se o esquema ou linha de cabeçalho deve ser incluído na contagem. A propriedade `IncludeSkippedRecords` especifica se deve-se contar os registros que são [ignorados] (# skipping-records).

Por padrão, `RecordNumberColumn` contará apenas os registros que são realmente retornados, começando com` 1`, então `2`,` 3`, `4`,` 5` e assim por diante. Se você contar o esquema, os registros sempre começarão em `2`. Incluir o esquema e os registros ignorados é o que você provavelmente deseja se estiver tentando simular o número da linha. A única vez em que o número do registro não seria igual ao número da linha é se um registro ocupar várias linhas. Aqui está um exemplo que mostra como capturar este * pseudo * número de linha:

```csharp
var mapper = SeparatedValueTypeMapper.Define(() => new Person());
mapper.Property(x => x.Name);
mapper.CustomMapping(new RecordNumberColumn("RecordNumber")
{
    IncludeSchema = true,
    IncludeSkippedRecords = true
}).WithReader(p => p.RecordNumber);

var options = new SeparatedValueOptions() { IsFirstRecordSchema = true };
var results = mapper.Read(reader, options).ToArray();
```

### Gravando metadados
Ainda não descobri um motivo pelo qual você deseja escrever metadados, mas forneço suporte para isso de qualquer maneira (sinta-se à vontade para me fornecer um exemplo!).

```csharp
var mapper = FixedLengthTypeMapper.Define(() => new Person());
mapper.Property(x => x.Name, 10);
mapper.Ignored(1);
// No need to define a reader or a writer - the underlying schema handles writing the metadata 
mapper.CustomMapping(new RecordNumberColumn("RecordNumber") { IncludeSchema = true }, 10);
mapper.Ignored(1);
mapper.Property(x => x.CreatedOn, 10).OutputFormat("MM/dd/yyyy");
```

### Criando suas próprias colunas de metadados
FlatFiles fornece a classe base abstrata `MetadataColumn <T>` para permitir que você crie suas próprias colunas de metadados. Para implementar essa interface, você deve implementar os métodos:

```csharp
T OnParse(IColumnContext context);

string OnFormat(IColumnContext context);
```

Dentro de `IColumnContext`, as seguintes informações são fornecidas atualmente:

* `PhysicalIndex` - O índice da coluna no arquivo.
* `LogicalIndex` - O índice da coluna, excluindo colunas ignoradas.
* `RecordContext` - Detalhes sobre o registro ao qual esta coluna pertence.
    * `PhysicalRecordNumber` - O número real de registros lidos do arquivo.
    * `LogicalRecordNumber` - O número de registros que não foram ignorados. * Esta contagem ainda não inclui o registro atual. *
    * `ExecutionContext` - Detalhes sobre a operação de leitura / gravação atual.
        * `Schema` - O esquema sendo usado para analisar o arquivo.
        * `Opções` - As opções passadas para o leitor / gravador.

## Pulando Registros
Se você trabalha diretamente com `SeparatedValueReader` ou` FixedLengthReader`, pode chamar `Skip` para pular registros arbitrariamente no arquivo de entrada. No entanto, você geralmente precisa da capacidade de inspecionar o registro para determinar se ele precisa ser ignorado. Mas e se você estiver tentando pular os registros * porque * eles não podem ser analisados? Se você precisar de mais controle sobre quais registros ignorar, FlatFiles fornece eventos para inspecionar registros durante o processo de análise. Esses eventos podem ser conectados quer você use mapeadores de tipo ou esteja trabalhando diretamente com leitores.

A análise de um registro passa pelo seguinte ciclo de vida:
1) Leia o texto até que um terminador de registro (geralmente uma nova linha) seja encontrado.
2) Para registros de comprimento fixo, particione o texto em colunas de string com base nas janelas configuradas.
3) Converta as colunas da string para os tipos de coluna designados, conforme definido no esquema.

Para arquivos CSV, a divisão de um registro em colunas é executada automaticamente durante a busca pelo terminador de registro. Antes de tentar converter o texto em ints, data / horas, etc., FlatFiles oferece a oportunidade de inspecionar os valores de string brutos e pular registros.

A classe `SeparatedValueReader` fornece um evento` RecordRead`, que permite pular registros indesejados. Por exemplo, você pode usar o código a seguir para localizar e pular registros CSV que faltam o número necessário de colunas:

```csharp
reader.RecordRead += (sender, e) =>
{
    e.IsSkipped = e.Values.Length < 10;
};
```

Arquivos de comprimento fixo vêm em dois sabores: aqueles com e sem terminadores de registro. Se não houver terminador de registro, pressupõe-se que * todos os registros tenham o mesmo comprimento *. Caso contrário, cada registro pode ter um comprimento diferente. Por esse motivo, FlatFiles oferece uma oportunidade extra de filtrar os registros antes de dividir o texto em colunas. Isso é útil para filtrar registros que não atendem a um requisito de comprimento mínimo ou aqueles que usam um caractere para indicar algo como comentários.

A classe `FixedLengthReader` fornece o evento` RecordRead`, que permite pular registros indesejados. Por exemplo, você pode usar o código abaixo para encontrar e pular registros que começam com o símbolo `#`:

```csharp
reader.RecordRead += (sender, e) => 
{
    e.IsSkipped = e.Record.StartsWith("#");
};
```

Semelhante aos arquivos CSV, você também pode filtrar registros de comprimento fixo * depois * de serem divididos em colunas. No entanto, é importante observar que o registro deve se ajustar às janelas configuradas.

Novamente, a classe `FixedLengthReader` fornece o evento` RecordPartitioned`, que permite pular registros indesejados. Por exemplo, você pode usar o código abaixo para localizar e pular registros cuja terceira coluna tem um flag:

```csharp
reader.RecordPartitioned += (sender, e) => 
{
    e.IsSkipped = e.Values[2] == "ERROR";
};
```

## Manipulação de erros
As classes leitor e gravador suportam dois eventos para tratamento de erros: `RecordError` e` ColumnError`. O evento `ColumnError` é gerado sempre que ocorre um erro durante a leitura / gravação de uma coluna; por exemplo, quando um valor não pode ser analisado. Nesse caso, uma instância de `ColumnErrorEventArgs` será enviada ao (s) ouvinte (s), que fornece acesso ao contexto (` ColumnContext`), o valor que causou o erro (`ColumnValue`) e a exceção que foi lançada ( `Exceção`).

Além disso, a propriedade `IsHandled` pode ser definida como` true` para informar FlatFiles que a exceção * não * deve ser propagada. Nesse caso, um valor deve ser fornecido para a propriedade `Substitution`. Esta propriedade é um `objeto`, então é importante certificar-se de que o valor substituído é do mesmo tipo que a coluna ou um erro de tempo de execução pode ocorrer.

O código a seguir pode ser usado para detectar problemas com um arquivo para que os erros possam ser relatados para um usuário:

```csharp
public class ErrorDetail
{
    public int RecordNumber { get; set; }
    public string ColumnName { get; set; }
    public string ErrorValue { get; set; }
    public string ErrorMessage { get; set; }
} 
//...
var details = new List<ErrorDetail>();
var csvReader = new SeparatedValueReader(textReader, schema);
csvReader.ColumnError += (sender, e) =>
{
    var columnContext = e.ColumnContext;
    var detail = new ErrorDetail()
    {
        RecordNumber = columnContext.RecordContext.PhysicalRecordNumber,
        ColumnName = columnContext.ColumnDefinition.ColumnName,
        ErrorValue = e.ColumnValue?.ToString(),
        ErrorMessage = e.Exception.InnerException?.Message ?? e.Exception.Message
    };
    details.Add(detail);
    e.IsHandled = true;
    e.Substitution = null;  // May not work for non-nullable value types
};
```
Se uma exceção no nível da coluna não for tratada, a exceção se propagará. Isso, junto com outros erros de nível de registro, fará com que o evento `RecordError` seja gerado. Os ouvintes receberão uma instância de `RecordErrorEventArgs`, que fornece acesso ao contexto (` RecordContext`) e à exceção que foi lançada (`Exception`). Da mesma forma, ele fornece uma propriedade `IsHandled`, que quando definida como` true`, impedirá a propagação da exceção. No caso de erros no nível do registro, isso faz com que o registro seja ignorado durante a leitura ou gravação.

## Arquivos contendo vários esquemas
Alguns formatos de arquivo simples conterão vários esquemas. Freqüentemente, os dados aparecem dentro de "blocos" com um cabeçalho e talvez um rodapé. Pode ser extremamente útil analisar cada tipo de registro usando um esquema diferente e retorná-los na mesma ordem em que aparecem no arquivo.

FlatFiles fornece suporte para esses formatos de arquivo usando as classes `SchemaSelector` e` TypeMapperSelector`. Depois de definir os esquemas ou mapeadores de tipo, você pode registrá-los com o seletor.

```csharp
var selector = new SeparatedValueTypeMapperSelector();
var dataMapper = getDataTypeMapper();
selector.When(x => x.Length == 10).Use(dataMapper);
selector.When(x => x.Length == 2).Use(getHeaderTypeMapper());
selector.When(x => x.Length == 3).Use(getFooterTypeMapper());
selector.WithDefault(dataMapper);
```

Cada tipo de 'mapeador' está associado a um predicado, que aceita o registro pré-processado. Se o predicado for bem-sucedido, esse mapeador de tipo será usado para analisar o registro. Cada predicado é testado na ordem em que é registrado, portanto, certifique-se de tornar seus predicados inteligentes o suficiente para identificar corretamente o tipo de registro.

* Nota: Você deve ter notado que `dataMapper` está registrado duas vezes, uma com antecedência com uma condição específica e depois como padrão. Isso não é necessário; no entanto, certificar-se de que o esquema mais comum seja tentado primeiro pode ajudar no desempenho em alguns casos. *

Uma vez que seu seletor esteja configurado, você pode chamar `GetReader`, passando o` TextReader` e o objeto options. A partir daí, você pode registrar eventos e ler os objetos do arquivo. Uma vez que diferentes tipos podem ser retornados, o leitor retornará instâncias de `objeto`.

Se você deseja trabalhar diretamente com esquemas e leitores, pode construir um `SchemaSelector` registrando esquemas com predicados de maneira semelhante. As classes `SeparatedValueReader` e` FixedLengthReader` fornecem um construtor que aceita um `SchemaSelector`. * Observe que sempre que você trabalhar com seletores, as chamadas para `GetSchema` retornarão` null`. *

```csharp
var selector = new SeparatedValueSchemaSelector();
var recordSchema = getDataSchema();
selector.When(values => values.Length == 10).Use(recordSchema);
selector.When(values => values.Length == 2).Use(getHeaderSchema());
selector.When(values => values.Length == 3).Use(getFooterSchema());
selector.WithDefault(recordSchema);

var reader = new SeparatedValueReader(fileStream, selector);
while (reader.Read())
{
    object[] values = reader.GetValues();
    processRecord(values);
}
```

Se você deseja * criar * arquivos com vários esquemas, existem equivalentes de "injetores" para cada classe de "seletores". Por exemplo:

```csharp
var selector = new FixedLengthTypeMapperInjector();
selector.WithDefault(getRecordTypeMapper());
selector.When<HeaderRecord>().Use(getHeaderTypeMapper());
selector.When<FooterRecord>().Use(getFooterTypeMapper());

var stringWriter = new StringWriter();
var writer = injector.GetWriter(stringWriter);
writer.Write(new HeaderRecord() { BatchName = "First Batch", RecordCount = 2 });
writer.Write(new DataRecord() { Id = 1, Name = "Bob Smith", CreatedOn = new DateTime(2018, 06, 04), TotalAmount = 12.34m });
writer.Write(new DataRecord() { Id = 2, Name = "Jane Doe", CreatedOn = new DateTime(2018, 06, 05), TotalAmount = 34.56m });
writer.Write(new FooterRecord() { TotalAmount = 46.9m, AverageAmount = 23.45m, IsCriteriaMet = true });
```

O método `When` aceita um predicado se você precisar de mais do que apenas o tipo de registro para decidir que tipo de mapeador / esquema usar.

## Mapeamento Personalizado
Os métodos `Property` são melhores quando o esquema do seu arquivo combina praticamente um-para-um com suas classes. Uma coluna vai para uma propriedade. Freqüentemente, porém, suas classes são mais estruturadas do que seus arquivos simples. A partir do FlatFiles 3.0, você pode fornecer sua própria lógica de mapeamento para controlar a serialização e desserialização de seus objetos.

Primeiro, vamos ver como usar o método `CustomMapping` para simular os métodos` Property`:

```csharp
var mapper = SeparatedValueTypeMapper.Define(() => new Person());
mapper.CustomMapping(new StringColumn("FirstName"))
    .WithReader(p => p.FirstName)
    .WithWriter(p => p.FirstName);
```

Este mapeamento diz ao FlatFiles para usar um `StringColumm` para ler / escrever os valores do arquivo. Em seguida, ele diz para ler e escrever os valores da propriedade `FirstName` de sua classe.

É importante notar que o tipo retornado por `IColumnDefinition` deve corresponder exatamente ao tipo da propriedade. Você deve lidar com `null`s e conversões por conta própria. Existem várias outras sobrecargas que fornecem mais controle - este exemplo específico seria melhor tratado com os métodos `Property`.

Aqui está um exemplo mais complicado onde duas colunas são usadas para construir uma propriedade `Geolocation`.

```csharp
public class Geolocation { public decimal Longitude; public decimal Latitude; }
public class Person { public Geolocation Location { get; set; } }
//...
var mapper = SeparatedValueTypeMapper.Define(() => new Person() 
{ 
    Location = new Geolocation() 
});
mapper.CustomMapping(new DecimalColumn("Longitude"))
    .WithReader((p, v) => p.Location.Longitude = (decimal)v)  // long way
    .WithWriter(p => p.Location.Longitude);
mapper.CustomMapping(new DecimalColumn("Latitude"))
    .WithReader(p => p.Location.Latitude)  // short way
    .WithWriter(p => p.Location.Latitude);
```

Finalmente, aqui está um exemplo de configuração, onde várias colunas são armazenadas em uma coleção.

```csharp
public class Contact
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<string> PhoneNumbers { get; set; } = new List<string>();
}
//...
var mapper = FixedLengthTypeMapper.Define(() => new Contact());
mapper.CustomMapping(new Int32Column("Id"), 10)
    .WithReader(c => c.Id)
    .WithWriter(c => c.Id);
mapper.CustomMapping(new StringColumn("Name"), 10)
    .WithReader(c => c.Name)
    .WithWriter(c => c.Name);
mapper.CustomMapping(new StringColumn("Phone1"), 12)
    .WithReader(PhoneReader)
    .WithWriter(c => c.PhoneNumbers.Count > 0 ? c.PhoneNumbers[0] : null);
mapper.CustomMapping(new StringColumn("Phone2"), 12)
    .WithReader(PhoneReader)
    .WithWriter(c => c.PhoneNumbers.Count > 1 ? c.PhoneNumbers[1] : null);
//...
public void PhoneReader(Contact contact, string phoneNumber)
{
    if (phoneNumber != null)
    {
        contact.PhoneNumbers.Add(phoneNumber);
    }
}
```

Em benchmarks, usar `CustomMapping` é apenas um pouco mais lento do que usar` Property`, tornando-o uma ótima opção quando você precisa de um pouco de controle extra.

Existem versões de `WithReader` e` WithWriter` que fornecem informações contextuais (`IColumnContext`), para que você possa acessar metadados enquanto lê e escreve. O método `WithWriter` também fornece uma sobrecarga ao longo do array subjacente que está sendo escrito, então você pode escrever em várias colunas simultaneamente ou inspecionar valores previamente serializados.

## Mapeamento de tempo de execução
Mesmo se você não souber o tipo de uma classe em tempo de compilação, ainda pode ser benéfico usar os mapeadores de tipo para preencher esses objetos a partir de um arquivo. Ou, se você estiver trabalhando em uma linguagem sem suporte para árvores de expressão, ficará feliz em saber que FlatFiles oferece uma maneira alternativa de configurar mapeadores de tipo.

O código a seguir ilustra como você pode definir um mapeador para um tipo que só é conhecido em tempo de execução:

```csharp
// Assume there is a runtime-generated type called "entityType" and a "TextReader".
var mapper = SeparatedValueMapper.DefineDynamic(entityType);
mapper.Int32Property("Id");
mapper.StringProperty("Name");
var entities = mapper.Read(reader).ToArray();
// Do something with the entities.
```

## Desativando Otimização
Os mapeadores de tipo do FlatFile podem serializar e desserializar extremamente rapidamente, gerando código em tempo de execução, usando classes no namespace `System.Reflection.Emit`. Para a maioria de nós, isso é uma notícia incrível porque significa que mapear valores de e para suas entidades é quase tão rápido como se você tivesse feito o mapeamento manualmente. No entanto, existem alguns ambientes, como Mono em execução no iOS, que não oferecem suporte a JIT'ing em tempo de execução, portanto, FlatFiles não funcionaria.

A partir da versão 1.0, os mapeadores suportam um novo método `OptimizeMapping` que pode ser usado para alternar para a reflexão (A.K.A., lenta). Por exemplo:

```csharp
var mapper = SeparatedValueTypeMapper.Define<Person>(() => new Person());
mapper.Property(x => x.Id);
mapper.Property(x => x.Name);
mapper.OptimizeMapping(false);  // Use normal reflection to get and set properties
```

## Classes e membros não públicos
A partir do FlatFiles 3.0, você não pode mais mapear para classes e membros não públicos (também conhecidos como `internal`,` protected` ou `private`) sem realizar etapas adicionais. A solução mais simples é tornar suas classes e membros `públicos`. Alternativamente, você pode [desativar otimizações] (# Disabling% 20Optimization), o que fará com que FlatFiles use reflexão normal, que deve ser capaz de acessar qualquer coisa, ao custo de alguma sobrecarga de tempo de execução.

Outra opção é conceder acesso a FlatFiles às suas classes e membros `internos` adicionando a seguinte linha ao seu arquivo` Assembly.cs`:

```csharp
[assembly: InternalsVisibleTo("FlatFiles.DynamicAssembly,PublicKey=00240000048000009400000006020000002400005253413100040000010001009b9e44f637b293021ec4d8625071e5fe1682eeb167c233b46314cca79bf2769606285d5d1225cba8ce1e75be9e8ab7251d17eaf2c3b00fde5eac50a0f7dc7fec2f70279ff71c72341ad2738661babfdc6792479f14fd64d841285644d5c09c2902e9467f574e0d369161caee632087c5d819c3c36f76622306b09a4f868230c1")]
```

Observe que isso só concede acesso a tipos / membros `internos`. Você ainda não será capaz de acessar membros `privados`.

Como uma opção final, você pode usar o método `CustomMapping`, passando delegados para ler / escrever seus membros. Como os delegados fazem parte do seu projeto, eles podem acessar membros não públicos sem problemas. Quase não há sobrecarga de tempo de execução usando `CustomMapping` em vez de` Property`.


## ADO.NET DataTables
Se você estiver usando `DataTable`s, você pode ler e escrever em um` DataTable` usando os métodos de extensão `ReadFlatFile` e` WriteFlatFile`. Basta passar o objeto leitor ou gravador correspondente.

```csharp
var customerTable = new DataTable("Customer");
using (var streamReader = new StreamReader(File.OpenRead(@"C:\path\to\file.csv")))
{
    var reader = new SeparatedValueReader(streamReader, schema);
    customerTable.ReadFlatFile(reader);
}
```

## FlatFileDataReader
Para leitura de arquivo ADO.NET de baixo nível, você pode usar a classe `FlatFileDataReader`. Ele fornece uma interface `IDataReader` para os registros no arquivo, tornando-o compatível com outras interfaces ADO.NET.

```csharp
// The DataReader Approach
using (var fileReader = new StreamReader(File.OpenRead(@"C:\path\to\file.csv"))
{
    var csvReader = new SeparatedValueReader(fileReader, schema);
    var dataReader = new FlatFileDataReader(csvReader);
    var customers = new List<Customer>();
    while (dataReader.Read())
    {
        var customer = new Customer();
        customer.CustomerId = dataReader.GetInt32(0);
        customer.Name = dataReader.GetString(1);
        customer.Created = dataReader.GetDateTime(2);
        customer.AverageSales = dataReader.GetDouble(3);
        customers.Add(customer);
    }
    return customers;
}
```

Normalmente, em casos como esse, é apenas mais fácil usar os mapeadores de tipo. No entanto, isso pode ser útil se você estiver trocando uma chamada real de banco de dados por dados CSV dentro de um teste de unidade.

FlatFiles também fornece métodos de extensão úteis na interface `IDataReader` para facilitar a extração de dados. Ele fornece variantes `GetNullable *` dos métodos `IDataReader`, então você não precisa chamar` IsDBNull` constantemente. Existem também variantes de cada método que aceita o nome da coluna em vez da posição ordinal.

Existem também métodos `GetValue <T>` genéricos que podem lidar com conversões de tipo automaticamente para você. Por exemplo, sempre que você lê um arquivo CSV sem fornecer o esquema, o FlatFiles assume que cada coluna é uma `string`. Ao chamar `GetValue <int> (" Id ")` ou `GetValue <DateTime?> (" CreatedOn ")`, FlatFiles tentará converter os valores para você. Isso é extremamente útil quando você não deseja fornecer (* ou não pode fornecer *) um esquema, mas ainda pode determinar os tipos de coluna. Por exemplo, quando você conhece os nomes das colunas e seus tipos, mas sua ordem pode ser diferente entre as execuções.

## Licença
Este é um software gratuito e desimpedido lançado em domínio público.

Qualquer pessoa é livre para copiar, modificar, publicar, usar, compilar, vender ou
distribuir este software, seja na forma de código-fonte ou como um compilado
binário, para qualquer finalidade, comercial ou não comercial, e por qualquer
meios.

Em jurisdições que reconhecem as leis de direitos autorais, o autor ou autores
deste software dedica todo e qualquer direito autoral no
software para o domínio público. Fazemos essa dedicação pelo benefício
do público em geral e em detrimento de nossos herdeiros e
sucessores. Pretendemos que essa dedicação seja um ato aberto de
renúncia perpétua de todos os direitos presentes e futuros a este
software sob a lei de direitos autorais.

O SOFTWARE É FORNECIDO "COMO ESTÁ", SEM QUALQUER TIPO DE GARANTIA,
EXPRESSA OU IMPLÍCITA, INCLUINDO, MAS NÃO SE LIMITANDO ÀS GARANTIAS DE
COMERCIALIZAÇÃO, ADEQUAÇÃO A UMA FINALIDADE ESPECÍFICA E NÃO VIOLAÇÃO.
EM NENHUMA HIPÓTESE OS AUTORES SERÃO RESPONSÁVEIS POR QUALQUER RECLAMAÇÃO, DANOS OU
OUTRAS RESPONSABILIDADES, SEJA EM AÇÃO DE CONTRATO, DELITO OU DE OUTRA FORMA,
DECORRENTE DE, FORA DE OU EM CONEXÃO COM O SOFTWARE OU O USO OU
OUTRAS NEGOCIAÇÕES NO SOFTWARE.

Para obter mais informações, consulte <http://unlicense.org>