## 4.14.0 (2021-06-22)
** Resumo ** - O comportamento de `TypedWriter.WriteAll` é um tanto não intuitivo quando chamado sem registros. A expectativa é que o cabeçalho seja escrito durante a execução dessa operação em massa; caso contrário, o chamador deve chamar explicitamente `WriteSchema` de antemão. Isso altera um pouco o comportamento do código, de modo que pode resultar em cabeçalhos / esquema sendo escritos nos casos em que o arquivo estava em branco antes. No entanto, quando `IsFirstRecordSchema` é` true`, é extremamente improvável que os consumidores esperem que um arquivo em branco seja gerado.

Durante meus testes, também descobri um bug em que o esquema estava sendo definido, e depois desfeito, quando o registro de cabeçalho / esquema era o único registro no arquivo. Você deve ser capaz de tentar `Ler` o primeiro registro de um arquivo vazio e obter` false` de volta, então obter o esquema via `GetSchema`; no entanto, meu código estava lançando uma `InvalidOperationException` ou, pior, uma` NullReferenceException`.


