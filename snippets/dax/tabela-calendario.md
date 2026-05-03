# Tabela Calendário

## 🔹 1° Forma — Power Query

### Descrição
Cria uma tabela calendário dinâmica com base nas datas de uma tabela qualquer.

### Passo a passo
1. Power Query  
2. Nova Fonte  
3. Consulta Nula  
4. Editor Avançado  
5. Colar o código  

### Código

```powerquery
let
    DataMinima = List.Min(Receitas[Data]),
    DataMaxima = List.Max(Receitas[Data]),

    DataInicio = Date.StartOfYear(DataMinima),
    DataFim = Date.EndOfYear(DataMaxima),

    Duracao = Duration.Days(Duration.From(DataFim - DataInicio)) + 1,

    Fonte = List.Dates(DataInicio, Duracao, #duration(1, 0, 0, 0)),

    ConversaoEmTabela = Table.FromList(Fonte, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    ColunaRenomeada = Table.RenameColumns(ConversaoEmTabela, {{"Column1", "Data"}}),
    TipoAlterado = Table.TransformColumnTypes(ColunaRenomeada, {{"Data", type date}})
in
    TipoAlterado
```

## 🔹 2° Forma — DAX

```DAX
dCalendario = CALENDARAUTO()
```

ou

```DAX
dCalendario = CALENDAR(DATE(2018,1,1), DATE(2022,12,31))
```

```DAX
Ano = YEAR(dCalendario[Date])
Mes = MONTH(dCalendario[Date])
Dia = DAY(dCalendario[Date])
NomeMes = FORMAT(dCalendario[Date], "MMM", "PT-BR")
Trimestre = "T" & QUARTER(dCalendario[Date])
```

## 🔹 3° Forma — DAX Dinâmico

```DAX
dCalendario = 
ADDCOLUMNS(
    CALENDAR(DATE(2018,1,1), DATE(2025,12,31)),
    "Ano", YEAR([Date]),
    "Mes", MONTH([Date]),
    "NomeMes", FORMAT([Date], "MMMM","PT-BR"),
    "MesAbrev", FORMAT([Date], "MMM"),
    "Trimestre", QUARTER([Date])
)
```

## 🔹 4° Forma — Baseado na Tabela Fato

```DAX
dCalendario = 
VAR MinAno = YEAR(MIN(fNotasFiscais[Data Faturamento]))
VAR MaxAno = YEAR(MAX(fNotasFiscais[Data Faturamento]))
RETURN
ADDCOLUMNS(
    FILTER(
        CALENDARAUTO(),
        YEAR([Date]) >= MinAno &&
        YEAR([Date]) <= MaxAno
    ),
    "Ano", YEAR([Date]),
    "Mes", MONTH([Date]),
    "NomeMes", FORMAT([Date], "MMMM","PT-BR"),
    "MesAbrev", FORMAT([Date], "MMM"),
    "Trimestre", QUARTER([Date])
)
```
