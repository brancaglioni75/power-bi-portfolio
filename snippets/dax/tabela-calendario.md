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

## 🔹 2° Forma — DAX - IA Claude.IA

```powerquery
let
    // 1. Identifica a data mínima e máxima da sua tabela fato "Receitas"
    DataMinima = List.Min(Receitas[Data]),
    DataMaxima = List.Max(Receitas[Data]),

    // 2. Determina o início do primeiro ano e o fim do último ano
    DataInicio = Date.StartOfYear(DataMinima),
    DataFim = Date.EndOfYear(DataMaxima),

    // 3. Calcula a quantidade de dias entre as datas
    Duracao = Duration.Days(Duration.From(DataFim - DataInicio)) + 1,

    // 4. Gera a lista de datas
    Fonte = List.Dates(DataInicio, Duracao, #duration(1, 0, 0, 0)),

    // 5. Converte a lista em Tabela
    ConversaoEmTabela = Table.FromList(Fonte, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    ColunaRenomeada = Table.RenameColumns(ConversaoEmTabela, {{"Column1", "Data"}}),
    TipoAlterado = Table.TransformColumnTypes(ColunaRenomeada, {{"Data", type date}}),
    #"Ano Inserido" = Table.AddColumn(TipoAlterado, "Ano", each Date.Year([Data]), Int64.Type),
    #"Mês Inserido" = Table.AddColumn(#"Ano Inserido", "Mês", each Date.Month([Data]), Int64.Type),
    #"Nome do Mês Inserido" = Table.AddColumn(#"Mês Inserido", "Nome do Mês", each Date.MonthName([Data]), type text),
    #"Dia Inserido" = Table.AddColumn(#"Nome do Mês Inserido", "Dia", each Date.Day([Data]), Int64.Type),
    #"Nome do Dia Inserido" = Table.AddColumn(#"Dia Inserido", "Nome do Dia", each Date.DayOfWeekName([Data]), type text),
    #"Trimestre Inserido" = Table.AddColumn(#"Nome do Dia Inserido", "Trimestre", each Date.QuarterOfYear([Data]), Int64.Type),

    // ── NOVAS COLUNAS ──────────────────────────────────────────────────────────

    // 6. Semestre
    #"Semestre Inserido" = Table.AddColumn(#"Trimestre Inserido", "Semestre", each
        if Date.Month([Data]) <= 6 then 1 else 2, Int64.Type),

    // 7. Estação do Ano (hemisfério sul — Brasil)
    #"Estação Inserida" = Table.AddColumn(#"Semestre Inserido", "Estação do Ano", each
        let m = Date.Month([Data]), d = Date.Day([Data])
        in
            if   (m = 3  and d >= 20) or (m = 4) or (m = 5) or (m = 6  and d < 21) then "Outono"
            else if (m = 6  and d >= 21) or (m = 7) or (m = 8) or (m = 9  and d < 23) then "Inverno"
            else if (m = 9  and d >= 23) or (m = 10) or (m = 11) or (m = 12 and d < 21) then "Primavera"
            else "Verão",
        type text),

    // 8. Feriados Nacionais do Brasil (lista fixa + Páscoa/Carnaval móveis)
    FeriadosFixos = (ano as number) as list =>
        {
            #date(ano, 1,  1),   // Confraternização Universal
            #date(ano, 4, 21),   // Tiradentes
            #date(ano, 5,  1),   // Dia do Trabalho
            #date(ano, 9,  7),   // Independência do Brasil
            #date(ano, 10, 12),  // Nossa Sra. Aparecida
            #date(ano, 11,  2),  // Finados
            #date(ano, 11, 15),  // Proclamação da República
            #date(ano, 11, 20),  // Consciência Negra (lei federal desde 2024)
            #date(ano, 12, 25)   // Natal
        },

    // Cálculo da Páscoa pelo algoritmo de Meeus/Jones/Butcher
    Pascoa = (ano as number) as date =>
        let
            a = Number.Mod(ano, 19),
            b = Number.IntegerDivide(ano, 100),
            c = Number.Mod(ano, 100),
            d = Number.IntegerDivide(b, 4),
            e = Number.Mod(b, 4),
            f = Number.IntegerDivide(b + 8, 25),
            g = Number.IntegerDivide(b - f + 1, 3),
            h = Number.Mod(19 * a + b - d - g + 15, 30),
            i = Number.IntegerDivide(c, 4),
            k = Number.Mod(c, 4),
            l = Number.Mod(32 + 2 * e + 2 * i - h - k, 7),
            m = Number.IntegerDivide(a + 11 * h + 22 * l, 451),
            mes = Number.IntegerDivide(h + l - 7 * m + 114, 31),
            dia = Number.Mod(h + l - 7 * m + 114, 31) + 1
        in
            #date(ano, mes, dia),

    // Monta a lista completa de feriados (fixos + móveis) para um dado ano
    TodosFeriados = (ano as number) as list =>
        let
            pascoa      = Pascoa(ano),
            sextaSanta  = Date.AddDays(pascoa, -2),
            corpusChristi = Date.AddDays(pascoa, 60),
            carnaval2   = Date.AddDays(pascoa, -47),   // segunda de carnaval
            carnaval3   = Date.AddDays(pascoa, -48)    // terça de carnaval (ponto facultativo, mas muito comum)
        in
            FeriadosFixos(ano) &
            {pascoa, sextaSanta, corpusChristi, carnaval2, carnaval3},

    // 9. Nome do Feriado Nacional
    NomeFeriado = (data as date) as text =>
        let
            ano     = Date.Year(data),
            pascoa  = Pascoa(ano),
            pares   = {
                {#date(ano, 1,  1),                        "Confraternização Universal"},
                {Date.AddDays(pascoa, -48),                "Carnaval (Segunda)"},
                {Date.AddDays(pascoa, -47),                "Carnaval (Terça)"},
                {Date.AddDays(pascoa, -2),                 "Sexta-feira Santa"},
                {pascoa,                                   "Páscoa"},
                {#date(ano, 4, 21),                        "Tiradentes"},
                {#date(ano, 5,  1),                        "Dia do Trabalho"},
                {Date.AddDays(pascoa, 60),                 "Corpus Christi"},
                {#date(ano, 9,  7),                        "Independência do Brasil"},
                {#date(ano, 10, 12),                       "Nossa Sra. Aparecida"},
                {#date(ano, 11,  2),                       "Finados"},
                {#date(ano, 11, 15),                       "Proclamação da República"},
                {#date(ano, 11, 20),                       "Consciência Negra"},
                {#date(ano, 12, 25),                       "Natal"}
            },
            encontrado = List.Select(pares, each _{0} = data)
        in
            if List.Count(encontrado) > 0 then encontrado{0}{1} else "",

    #"Nome do Feriado Inserido" = Table.AddColumn(#"Estação Inserida", "Nome do Feriado Nacional", each
        NomeFeriado([Data]), type text),

    // 10. Dia Útil (segunda a sexta, excluindo feriados nacionais)
    #"Dia Útil Inserido" = Table.AddColumn(#"Nome do Feriado Inserido", "É Dia Útil", each
        let
            diaSemana = Date.DayOfWeek([Data], Day.Monday), // 0=Seg … 4=Sex, 5=Sab, 6=Dom
            eFimDeSemana = diaSemana >= 5,
            eFeriado = List.Contains(TodosFeriados(Date.Year([Data])), [Data])
        in
            not eFimDeSemana and not eFeriado,
        type logical)

in
    #"Dia Útil Inserido"
```

## 🔹 3° Forma — DAX - GEMINI IA

```powerquery
let
    DataMinima = List.Min(Receitas[Data]),
    DataMaxima = List.Max(Receitas[Data]),

    DataInicio = Date.StartOfYear(DataMinima),
    DataFim = Date.EndOfYear(DataMaxima),
    Duracao = Duration.Days(DataFim - DataInicio) + 1,

    Fonte = List.Dates(DataInicio, Duracao, #duration(1, 0, 0, 0)),
    Tabela = Table.FromList(Fonte, Splitter.SplitByNothing(), {"Data"}),
    TipoAlterado = Table.TransformColumnTypes(Tabela, {{"Data", type date}}),

    Ano = Table.AddColumn(TipoAlterado, "Ano", each Date.Year([Data]), Int64.Type),
    Mes = Table.AddColumn(Ano, "Mês", each Date.Month([Data]), Int64.Type),
    NomeMes = Table.AddColumn(Mes, "Nome do Mês", each Date.MonthName([Data]), type text),
    Dia = Table.AddColumn(NomeMes, "Dia", each Date.Day([Data]), Int64.Type),
    NomeDia = Table.AddColumn(Dia, "Nome do Dia", each Date.DayOfWeekName([Data]), type text),
    Trimestre = Table.AddColumn(NomeDia, "Trimestre", each Date.QuarterOfYear([Data]), Int64.Type),
    Semestre = Table.AddColumn(Trimestre, "Semestre", each if [Mês] <= 6 then 1 else 2, Int64.Type),

    Estacao = Table.AddColumn(
        Semestre,
        "Estação",
        each 
            if ([Mês] = 3 and [Dia] >= 20) or ([Mês] > 3 and [Mês] < 6) or ([Mês] = 6 and [Dia] < 21) then "Outono"
            else if ([Mês] = 6 and [Dia] >= 21) or ([Mês] > 6 and [Mês] < 9) or ([Mês] = 9 and [Dia] < 22) then "Inverno"
            else if ([Mês] = 9 and [Dia] >= 22) or ([Mês] > 9 and [Mês] < 12) or ([Mês] = 12 and [Dia] < 21) then "Primavera"
            else "Verão",
        type text
    ),

    NomeFeriado = Table.AddColumn(
        Estacao,
        "Nome do Feriado",
        each
            let
                DataAtual = [Data],
                AnoAtual = Date.Year(DataAtual),

                a = Number.Mod(AnoAtual, 19),
                b = Number.IntegerDivide(AnoAtual, 100),
                c = Number.Mod(AnoAtual, 100),
                d = Number.IntegerDivide(b, 4),
                e = Number.Mod(b, 4),
                f = Number.IntegerDivide(b + 8, 25),
                g = Number.IntegerDivide(b - f + 1, 3),
                h = Number.Mod(19 * a + b - d - g + 15, 30),
                i = Number.IntegerDivide(c, 4),
                k = Number.Mod(c, 4),
                l = Number.Mod(32 + 2 * e + 2 * i - h - k, 7),
                m = Number.IntegerDivide(a + 11 * h + 22 * l, 451),
                MesPascoa = Number.IntegerDivide(h + l - 7 * m + 114, 31),
                DiaPascoa = Number.Mod(h + l - 7 * m + 114, 31) + 1,
                DataPascoa = #date(AnoAtual, MesPascoa, DiaPascoa),

                Nome =
                    if DataAtual = #date(AnoAtual, 1, 1) then "Ano Novo"
                    else if DataAtual = Date.AddDays(DataPascoa, -47) then "Carnaval"
                    else if DataAtual = Date.AddDays(DataPascoa, -2) then "Sexta-feira Santa"
                    else if DataAtual = #date(AnoAtual, 4, 21) then "Tiradentes"
                    else if DataAtual = #date(AnoAtual, 5, 1) then "Dia do Trabalho"
                    else if DataAtual = Date.AddDays(DataPascoa, 60) then "Corpus Christi"
                    else if DataAtual = #date(AnoAtual, 9, 7) then "Independência"
                    else if DataAtual = #date(AnoAtual, 10, 12) then "Nossa Sra. Aparecida"
                    else if DataAtual = #date(AnoAtual, 11, 2) then "Finados"
                    else if DataAtual = #date(AnoAtual, 11, 15) then "Proclamação da República"
                    else if DataAtual = #date(AnoAtual, 11, 20) then "Consciência Negra"
                    else if DataAtual = #date(AnoAtual, 12, 25) then "Natal"
                    else null
            in
                Nome,
        type text
    ),

    DiaUtil = Table.AddColumn(
        NomeFeriado,
        "Dia Útil",
        each if Date.DayOfWeek([Data], Day.Monday) < 5 and [Nome do Feriado] = null then "Sim" else "Não",
        type text
    )
in
    DiaUtil
```

## 🔹 4° Forma — DAX -  É a mais funcional pra mim

```DAX
dCalendario = CALENDARAUTO()


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

## 🔹 5° Forma — DAX Dinâmico

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

## 🔹 6° Forma — Baseado na Tabela Fato

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
