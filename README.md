# Monitorização da Qualidade do Ar e Meteorologia (ETL com Pentaho Kettle)

Trabalho Prático I de **Integração de Sistemas de Informação** (2026/27)
Licenciatura em Engenharia de Sistemas Informáticos, Escola Superior de Tecnologia do IPCA

**Autor:** `Afonso Ferreira` (n.º `27981`)
**Ferramenta ETL:** Pentaho Data Integration (Kettle / Spoon)
**Base de dados:** PostgreSQL

---

## Descrição

Este projeto implementa um processo **ETL (Extract, Transform, Load)** que recolhe dados meteorológicos e de qualidade do ar para vários locais do Minho, limpa e valida a informação, cruza as duas fontes e carrega o resultado numa base de dados, em ficheiros exportados e num dashboard.

O objetivo é demonstrar a integração de dados vindos de **fontes e formatos heterogéneos**, com validação, normalização, anonimização, orquestração e alertas automáticos quando a qualidade do ar é má.

## Arquitetura

```
 FASE 1: EXTRACT              FASE 2: TRANSFORM                FASE 3: LOAD
┌──────────────────┐     ┌─────────────────────────┐     ┌──────────────────────┐
│ locais.csv       │     │ Limpeza (nulos, dups)   │     │ PostgreSQL           │
│ limites.xml/csv  │ ──► │ Regex e normalização    │ ──► │ Exportação CSV/JSON  │
│ API meteorologia │     │ Anonimização de emails  │     │ SFTP, email, alertas │
│ API qualidade ar │     │ Join, lookup, índice    │     │ Dashboard (Grafana)  │
└──────────────────┘     │ Agregações diárias      │     └──────────────────────┘
                         └─────────────────────────┘
                     Orquestração: Job principal + logs
```

## Fontes de dados

| Fonte | Formato | Tipo |
|---|---|---|
| `data/locais.csv` (locais, coordenadas, email de contacto) | CSV | Ficheiro local |
| `data/limites.xml` (limites de referência por poluente) | XML | Ficheiro local |
| [Open-Meteo Weather API](https://open-meteo.com/en/docs) | JSON | API REST |
| [Open-Meteo Air Quality API](https://open-meteo.com/en/docs/air-quality-api) | JSON | API REST |

As APIs são gratuitas e não requerem chave de acesso.

## O que o projeto demonstra

- **Formatos:** CSV, JSON e XML (leitura e exportação)
- **Fontes:** ficheiros locais e APIs REST
- **Limpeza e validação:** valores nulos, remoção de duplicados, validação de intervalos
- **Expressões regulares:** normalização de nomes, validação de emails
- **Anonimização:** mascaramento de emails (`j***@gmail.com`)
- **Cruzamento:** joins e lookups entre meteorologia, qualidade do ar e limites
- **Cálculos:** índice de qualidade do ar (Bom / Moderado / Mau)
- **Agregações:** médias, máximos e mínimos diários
- **Persistência:** PostgreSQL com chaves únicas para evitar duplicados
- **Orquestração:** Job principal com tratamento de erros e logs
- **Notificações:** email de alerta e envio por SFTP
- **Visualização:** dashboard com os resultados

## Estrutura do repositório

```
.
├── data/              # Ficheiros de entrada (locais.csv, limites.xml)
├── transformacoes/    # Transformações do Kettle (.ktr)
├── jobs/              # Jobs do Kettle (.kjb)
├── sql/               # Scripts SQL (criação de tabelas)
├── export/            # Ficheiros gerados pelo processo
├── logs/              # Registos de execução
├── docs/              # Relatório e capturas de ecrã
└── README.md
```

## Transformações e Jobs

| Ficheiro | Descrição | Estado |
|---|---|---|
| `T1_extract_meteo.ktr` | Lê os locais, constrói o URL e extrai a meteorologia (JSON) | Concluído |
| `T2_extract_qualidade_ar.ktr` | Extrai PM10, PM2.5, NO2 e O3 por local e hora (JSON) | Concluído |
| `T3_limpeza_locais.ktr` | Limpeza, validação com Regex, anonimização e ramo de rejeitados | Em curso |
| `T4_transformacao.ktr` | Join, lookup dos limites, índice de qualidade e agregações | Por fazer |
| `T5_carga_bd.ktr` | Carga no PostgreSQL | Por fazer |
| `J_pipeline_principal.kjb` | Job que orquestra tudo, com logs e alertas | Por fazer |

## Modelo de dados (PostgreSQL)

Base de dados: `qualidade_ar`

- **`medicoes`**: `local`, `data_hora`, `temperatura`, `humidade`, `pm2_5`, `pm10`, `no2`, `o3`, `indice` (chave única em `local` + `data_hora`)
- **`resumo_diario`**: `local`, `data`, `temp_media`, `pm2_5_media`, `pm2_5_max`, `pm10_media`, `estado` (chave única em `local` + `data`)
- **`log_execucoes`**: `data_execucao`, `estado`, `mensagem`

O script de criação das tabelas está em `sql/criar_tabelas.sql`.

## Como executar

### Requisitos
- Java JDK 17 (ou o exigido pela versão do PDI usada)
- [Pentaho Data Integration (Community Edition)](https://pentaho.com/)
- PostgreSQL 14 ou superior, com o driver JDBC do PostgreSQL na pasta `lib` do PDI
- Ligação à Internet (para as APIs)

### Passos
1. Clonar o repositório:
   ```bash
   git clone <URL_DO_REPOSITORIO>
   ```
2. Criar a base de dados `qualidade_ar` no PostgreSQL e executar `sql/criar_tabelas.sql`.
3. Abrir o Spoon e configurar a conexão PostgreSQL (`localhost`, porta `5432`, base `qualidade_ar`).
4. Ajustar os caminhos dos ficheiros de entrada em `data/` nas transformações, se necessário.
5. Abrir e executar `jobs/J_pipeline_principal.kjb`.

> **Nota:** as credenciais (base de dados, email, SFTP) não estão incluídas no repositório. Devem ser configuradas localmente.

## Dashboard

`<Captura de ecrã do dashboard>`

## Notas e limitações

- Os limites de qualidade do ar usados são **valores simplificados** para fins académicos. Para uso real, devem ser confirmados com as fontes oficiais (OMS / UE).
- Os dados do ficheiro `locais.csv` contêm **erros propositados** (espaços, maiúsculas, duplicados, emails inválidos) para demonstrar as operações de limpeza.

## Trabalhos futuros

- Integração de streaming de sensores via MQTT / Node-RED
- Geração de relatório diário em PDF
- Mais locais e mais poluentes
- Agendamento automático da execução do Job

## Referências

- Open-Meteo, *Weather Forecast API*: https://open-meteo.com/en/docs (acedido a 12/10/2026)
- Open-Meteo, *Air Quality API*: https://open-meteo.com/en/docs/air-quality-api (acedido a 12/10/2026)
- Pentaho Data Integration, documentação oficial
- PostgreSQL, documentação oficial: https://www.postgresql.org/docs/
