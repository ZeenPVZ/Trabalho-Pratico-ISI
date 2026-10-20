# Monitorização da Qualidade do Ar e Meteorologia

Trabalho Prático I de **Integração de Sistemas de Informação** (2026/27) — Licenciatura em Engenharia de Sistemas Informáticos, Escola Superior de Tecnologia do IPCA.

**Autor:** Afonso Ferreira (n.º 27981)  
**ETL:** Pentaho Data Integration (Spoon / Kettle)  
**Base de dados:** PostgreSQL (`qualidade_ar`)

Representação videográfica do trabalho prático: https://youtu.be/5Dmj7uK0YDw

## Objetivo

O projeto integra dados meteorológicos e de qualidade do ar para vários locais. O fluxo lê os locais de um CSV, consulta as APIs Open-Meteo, limpa e valida os dados de entrada, cruza as medições por local e hora e grava os resultados no PostgreSQL. Uma segunda saída agrega as medições por local e dia.

## Fluxo ETL implementado

```text
locais.csv
   │
   ▼
T3_limpeza_locais ──► locais_deduplicados.csv
   │                         │
   ├──► locais_aceites.csv    ├──► T1_extract_meteo ──► stg_meteo
   └──► locais_rejeitados.csv└──► T2_extract_qualidade_ar ──► stg_qualidade_ar
                                                               │
                                                               ▼
                                                        T4_transformacao
                                                         ├──► medicoes
                                                         └──► resumo_diario

J_pipeline_principal: T3 → T1 → T2 → T4
```

### Transformações e Job

| Ficheiro | Função | Estado |
|---|---|---|
| `transformacoes/T3_limpeza_locais.ktr` | Lê `data/locais.csv`, ordena e remove duplicados, valida dados e emails, e exporta ficheiros deduplicados, aceites e rejeitados. | Executado |
| `transformacoes/T1_extract_meteo.ktr` | Usa os locais deduplicados para consultar a API meteorológica e carregar temperatura e humidade em `stg_meteo`. | Executado |
| `transformacoes/T2_extract_qualidade_ar.ktr` | Consulta a API de qualidade do ar e carrega PM10, PM2.5, NO₂ e O₃ em `stg_qualidade_ar`. | Executado |
| `transformacoes/T4_transformacao.ktr` | Junta as tabelas de staging por local e data/hora; carrega `medicoes` e calcula o resumo diário em `resumo_diario`. | Executado |
| `transformacoes/J_pipeline_principal.kjb` | Orquestra T3, T1, T2 e T4, pela ordem indicada. | Executado com sucesso |

## Fontes de dados

| Fonte | Formato | Utilização |
|---|---|---|
| `data/locais.csv` | CSV | Lista de locais, coordenadas e dados de contacto usados no processo. |
| [Open-Meteo Weather API](https://open-meteo.com/en/docs) | JSON via REST | Temperatura e humidade por local e hora. |
| [Open-Meteo Air Quality API](https://open-meteo.com/en/docs/air-quality-api) | JSON via REST | PM10, PM2.5, NO₂ e O₃ por local e hora. |

As APIs Open-Meteo utilizadas não requerem chave de acesso.

## Base de dados

A base de dados usada é `qualidade_ar`, através da ligação Spoon `PostgreSQL_qualidade_ar`. O processo utiliza as seguintes tabelas:

- **`stg_meteo`** — dados meteorológicos extraídos: local, data/hora, temperatura e humidade.
- **`stg_qualidade_ar`** — dados de qualidade do ar extraídos: local, data/hora e poluentes.
- **`medicoes`** — resultado do cruzamento das duas fontes, com chave única em `(local, data_hora)`.
- **`resumo_diario`** — agregação por `(local, data)`, incluindo temperatura média, mínima e máxima, PM2.5 médio e máximo, PM10 médio e estado.
- **`log_execucoes`** — tabela existente na base de dados; o Job atual não está documentado como escrevendo nesta tabela.

Na última execução confirmada, `medicoes` continha **252 registos** e `resumo_diario` **21 registos**. Estes valores são uma referência da execução observada e podem variar com os dados devolvidos pelas APIs.

## Estrutura do projeto

```text
.
├── data/
│   ├── locais.csv
│   └── limites.xml
├── export/
│   ├── locais_deduplicados.csv
│   ├── locais_aceites.csv
│   └── locais_rejeitados.csv
├── transformacoes/
│   ├── J_pipeline_principal.kjb
│   ├── T1_extract_meteo.ktr
│   ├── T2_extract_qualidade_ar.ktr
│   ├── T3_limpeza_locais.ktr
│   └── T4_transformacao.ktr
└── README.md
```

`data/limites.xml` existe na pasta de dados, mas não está ligado ao fluxo ETL descrito acima. Não há atualmente um dashboard documentado neste repositório.

## Como executar

### Requisitos

- Pentaho Data Integration / Spoon instalado.
- PostgreSQL instalado e acessível.
- Ligação à Internet para consultar as APIs.
- Tabelas de staging e destino criadas na base de dados `qualidade_ar`.

### Preparação e execução

1. Abra no Spoon os ficheiros da pasta `transformacoes/`.
2. Configure a ligação `PostgreSQL_qualidade_ar` com os dados do seu PostgreSQL.
3. Confirme os caminhos dos ficheiros de entrada e saída nas transformações, em particular `data/locais.csv` e os CSV da pasta `export/`. Os ficheiros `.ktr` guardados podem conter caminhos absolutos do computador onde foram criados; altere-os se executar o projeto noutro local.
4. Confirme que as tabelas `stg_meteo`, `stg_qualidade_ar`, `medicoes` e `resumo_diario` existem e têm colunas compatíveis com as transformações. O T1 e o T2 voltam a carregar as respetivas tabelas de staging.
5. Execute `transformacoes/J_pipeline_principal.kjb` no Spoon.
6. Verifique os registos em `medicoes` e `resumo_diario` no pgAdmin ou através de consultas SQL.

As credenciais do PostgreSQL não são incluídas neste README. Configure-as localmente no Spoon.

## Verificação dos resultados

Para contar os registos das tabelas finais no pgAdmin:

```sql
SELECT 'medicoes' AS tabela, COUNT(*) AS registos
FROM public.medicoes
UNION ALL
SELECT 'resumo_diario' AS tabela, COUNT(*) AS registos
FROM public.resumo_diario;
```

As restrições únicas impedem registos repetidos para a mesma chave. Se executar manualmente T4 novamente sem limpar ou atualizar os dados de destino, poderão surgir erros de chave duplicada.

## Limitações e trabalho futuro

- O ficheiro `limites.xml` ainda não participa no fluxo atual; não é calculado um índice de qualidade do ar com base nesse ficheiro.
- O Job guarda os ficheiros `.ktr` através de caminhos locais absolutos, pelo que estes podem ter de ser ajustados noutro computador.
- Dashboard, alertas por email, envio por SFTP e anonimização de emails não estão implementados no fluxo documentado.
- Melhorias possíveis: tornar os caminhos relativos, definir uma estratégia de atualização/recarga das tabelas finais, integrar os limites de referência e criar visualizações dos resumos.

## Referências

- Open-Meteo, [Weather Forecast API](https://open-meteo.com/en/docs).
- Open-Meteo, [Air Quality API](https://open-meteo.com/en/docs/air-quality-api).
- Pentaho Data Integration, documentação oficial.
- PostgreSQL, [documentação oficial](https://www.postgresql.org/docs/).
