---
title: 'Reading FinRobot as an Engineer: What Its Equity Research Pipeline Actually Does'
description: 'Lorem ipsum dolor sit amet'
pubDate: 'May 14 2026'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

# Reading FinRobot as an Engineer: What Its Equity Research Pipeline Actually Does

Recently, I spent some time reading and running FinRobot, an open-source AI agent platform for financial analysis. FinRobot describes itself as an AI agent platform tailored for financial applications, combining LLMs, financial data, quantitative analytics, and report-generation workflows. Its GitHub repository also presents FinRobot Pro as an equity research assistant that can fetch financial data, run LLM-based analysis, and generate professional equity research reports. ([GitHub][1])

At first glance, this sounds like a full financial research agent. But after running the equity report workflow and reviewing the implementation, I think a more precise description is:

> FinRobot’s equity report module is a two-stage report-generation pipeline powered by financial APIs, default forecast assumptions, LLM-generated text sections, and an HTML rendering layer.

This is not meant as a dismissal. In fact, it is useful precisely because it is runnable. But the code also reveals an important gap between the high-level “financial agent” narrative and the actual engineering structure underneath.

This post summarizes what I found while tracing the data input/output chain.

---

## 1. The High-Level Workflow: A Two-Stage Pipeline

The equity report flow is organized around two main scripts:

```text
Stage 1: generate_financial_analysis.py
Stage 2: create_equity_report.py
```

The project’s README also shows this two-step command-line workflow: first run `generate_financial_analysis.py` to produce financial analysis, then run `create_equity_report.py` to generate the report from the intermediate outputs. ([GitHub][1])

Conceptually, the pipeline looks like this:

```text
User Inputs + Config
        ↓
Stage 1: Financial Analysis Generation
        ↓
Intermediate CSV / JSON / TXT Artifacts
        ↓
Stage 2: Report Assembly and Rendering
        ↓
Final HTML Equity Research Report
```

My own code tracing confirms the same structure: `generate_financial_analysis.py` acts as the data and analysis production layer, while `create_equity_report.py` consumes the generated files, optionally fetches additional market data, generates charts, and renders the final HTML report. 

---

## 2. Stage 1: `generate_financial_analysis.py` as the Data Production Layer

The first stage takes a small number of user inputs and a few configuration values, then automatically pulls financial data and generates intermediate artifacts.

The required user inputs are simple:

```text
--company-ticker
--company-name
```

The optional inputs include:

```text
--peer-tickers
--years-limit
--period
--output-dir
--generate-text-sections
--enable-sensitivity-analysis
--enable-catalyst-analysis
--enable-enhanced-news
```

There are also forecast-related command-line arguments:

```text
--revenue-growth-2025
--revenue-growth-2026
--revenue-growth-2027
--margin-improvement
--sga-margin-improvement
```

In the uploaded source code, these default to 5% revenue growth for 2025E, 6% for 2026E, 4% for 2027E, 1% annual margin improvement, and -0.5% SG&A margin change. 

The script loads API keys from `config.ini`, including FMP, OpenAI, and optionally Adanos. It then calls FMP to retrieve the company’s financial data, processes historical metrics, generates forecasts, optionally fetches peer data, news, sentiment data, sensitivity analysis, catalyst analysis, and AI-generated text sections. 

The main Stage 1 outputs include:

```text
financial_metrics_and_forecasts.csv
income_statement_raw_data.csv
balance_sheet_raw_data.csv
cash_flow_raw_data.csv
ratios_raw_data.csv
key_metrics_raw_data.csv
peer_ebitda_comparison.csv
peer_ev_ebitda_comparison.csv
company_news.json
enhanced_news.json
retail_sentiment.json
sensitivity_analysis.json
catalyst_analysis.json
tagline.txt
company_overview.txt
investment_overview.txt
valuation_overview.txt
risks.txt
competitor_analysis.txt
major_takeaways.txt
analysis_summary.json
```

The most important artifact is probably:

```text
financial_metrics_and_forecasts.csv
```

This is the core derived dataset. It is generated from historical FMP financial data plus forecast assumptions, and it later feeds the report tables, charts, takeaways, and narrative sections. 

---

## 3. The Main Data Sources

From the implementation, the equity report pipeline mainly depends on a few sources.

### FMP: Financial Modeling Prep

FMP is the core financial data provider in the current implementation. The repository README says the first step fetches income statements, balance sheets, and cash flows via FMP API, then processes and forecasts financial data. ([GitHub][1])

In the traced pipeline, FMP provides:

```text
income statement
balance sheet
cash flow statement
ratios
key metrics
peer EBITDA
peer EV/EBITDA
company news
market metrics
technical indicator inputs
```

This means the current implementation is much closer to an API-powered financial report generator than to a fully independent SEC filing parser or source-grounded research system.

### OpenAI

OpenAI is used to generate text sections when text generation is enabled. These include sections such as company overview, investment overview, valuation overview, risks, competitor analysis, and major takeaways. 

### Adanos

Adanos is optional and is used for retail sentiment data. The Stage 1 code supports fetching retail sentiment insights if an Adanos API key is configured. 

---

## 4. Stage 2: `create_equity_report.py` as the Report Assembly Layer

The second stage consumes the artifacts produced by Stage 1 and renders the final report.

At the CLI level, it requires:

```text
--company-ticker
--company-name
--analysis-csv
--ratios-csv
--tagline-file
--company-overview-file
--investment-overview-file
--valuation-overview-file
--risks-file
--competitor-analysis-file
--major-takeaways-file
```

However, many of these “inputs” are not really manually written user inputs in the standard workflow. They are usually generated by Stage 1 and passed into Stage 2. This is an important distinction: the report-generation script looks like it requires many files, but the pipeline is designed so that most of those files are intermediate artifacts. 

The second stage then does several things:

```text
1. Load analysis CSV and ratios CSV
2. Load text sections
3. Optionally validate or regenerate text sections
4. Auto-fetch market data from FMP if not skipped
5. Compute technical indicators
6. Generate charts
7. Assemble a report_data dictionary
8. Render the final HTML report
```

The implementation also defines a priority rule for many market data fields:

```text
CLI override > auto-fetched FMP value > hard-coded default
```

For example, if the user provides a share price manually, the script uses that. Otherwise, it tries to use the auto-fetched FMP value. If neither exists, it falls back to a default. 

This is convenient, but it also introduces an important reproducibility issue.

---

## 5. Why Do Forecasts Need Default Values?

At first, I found the default forecast assumptions confusing. Why should a financial agent need hard-coded defaults like 5%, 6%, and 4% revenue growth?

The answer is partly reasonable and partly problematic.

Forecasting always requires assumptions. Historical financial data only tells us what happened in the past. To estimate future revenue, margins, EBITDA, EPS, or valuation, the system must assume something about future growth and profitability.

For example:

```text
2025E Revenue = 2024A Revenue × (1 + revenue growth assumption)
```

So the existence of assumptions is not the problem.

The real issue is the source of those assumptions.

In FinRobot’s current implementation, the default forecast assumptions are command-line defaults. If the user does not explicitly provide different values, the system uses the built-in defaults. 

That makes the forecast layer:

```text
assumption-driven
```

rather than:

```text
evidence-derived
```

A more robust financial research agent should derive or justify forecast assumptions from sources such as:

```text
historical CAGR
recent quarterly trends
management guidance
earnings call transcripts
consensus estimates
peer trends
industry growth rates
segment-level operating drivers
macro assumptions
```

The system could still allow manual overrides, but every assumption should have metadata:

```json
{
  "metric": "Revenue Growth",
  "period": "2026E",
  "value": 0.06,
  "source": "manual_override",
  "rationale": "User-provided base case assumption",
  "confidence": "medium"
}
```

Or, if system-derived:

```json
{
  "metric": "Revenue Growth",
  "period": "2026E",
  "value": 0.045,
  "source": "historical_5y_cagr + management_guidance",
  "rationale": "5-year revenue CAGR was 4.2%; management guided low-single-digit growth.",
  "confidence": "medium"
}
```

This distinction matters. A default value is useful for demos, but dangerous for investment research if it silently drives valuation outputs.

---

## 6. Why Does Stage 2 Fetch More Data?

The second architectural question is more important:

> If Stage 1 is the data generation stage, why does Stage 2 still fetch external data?

In `create_equity_report.py`, Stage 2 can automatically fetch market data if `--skip-auto-fetch` is not set and an FMP API key is available. It fetches fields such as share price, target price, rating, market cap, volume, forward P/E, P/B, dividend yield, free float, ROE, net debt to equity, sector, 52-week range, and shares outstanding. It also computes technical indicators by fetching historical price data. 

This makes the actual pipeline look like this:

```text
Stage 1:
Fetch financial statements, ratios, key metrics, peer data, news, sentiment, and text sections

Stage 2:
Fetch market snapshot data and technical indicator inputs, then render the report
```

This is convenient from a demo engineering perspective. The report page needs fresh market snapshot fields, and it is easy to fetch them right before rendering.

But from a data engineering perspective, this creates a lineage problem:

```text
The final report is not fully determined by the Stage 1 artifacts.
```

If I run Stage 1 today and Stage 2 tomorrow, the final report may include different share price, market cap, volume, rating, or technical indicators. That means the report is not a pure rendering of a frozen data snapshot.

For a financial research workflow, this is not ideal.

A research report should be reproducible as of a specific date:

```text
As of report date X, using data snapshot Y, the system generated report Z.
```

If the rendering layer keeps calling external APIs, then the report becomes harder to audit and reproduce.

---

## 7. A Cleaner Architecture: Freeze Data Before Rendering

In my view, a cleaner architecture would separate the pipeline into deterministic stages:

```text
Stage 1: Data Ingestion
- Fetch all external data
- Save raw API responses
- Save source metadata
- Save timestamps
- Save a frozen data snapshot

Stage 2: Analysis
- Calculate historical metrics
- Generate forecasts
- Run peer comparison
- Run valuation
- Run sensitivity analysis
- Compute technical indicators

Stage 3: Narrative Generation
- Generate text from structured facts
- Attach supporting facts to each claim
- Prevent unsupported statements

Stage 4: Report Rendering
- Read frozen artifacts only
- Generate charts
- Render HTML/PDF
- No external API calls
```

If the project must remain two-stage, then Stage 1 should fetch all external data and Stage 2 should be a pure rendering layer:

```text
Stage 1: Data + Analysis Generation
Stage 2: Pure Report Rendering
```

The key principle is:

```text
render_report.py should be deterministic.
```

Given the same input artifacts, it should always generate the same report.

---

## 8. What FinRobot Does Well

Despite these concerns, I do not think FinRobot is useless. Quite the opposite: it is a helpful working reference.

Its value is that it demonstrates a complete equity report pipeline:

```text
financial data fetching
historical metric processing
forecast generation
peer comparison
news integration
sentiment integration
LLM-generated narrative sections
chart generation
HTML report rendering
```

The GitHub README also presents it as a locally deployed assistant that fetches financial data, runs multi-agent LLM analysis, and generates equity research reports. ([GitHub][1])

For someone learning financial AI engineering, this is useful because it shows the minimum working skeleton of a financial report generator.

But it should be read as an engineering blueprint, not as a fully production-grade buy-side research agent.

---

## 9. Main Limitations I Observed

After tracing the data flow, I would summarize the limitations as follows.

### 1. Forecasts are assumption-driven

The forecast defaults make the system runnable, but they are not strongly grounded in company-specific evidence. Different companies and sectors should not silently share the same growth and margin assumptions.

### 2. Data lineage is split

Some data comes from Stage 1 artifacts. Some data is fetched during Stage 2. Some text is generated in Stage 1. Some text can be regenerated in Stage 2. This makes the final report harder to audit.

### 3. The rendering layer has too many responsibilities

`create_equity_report.py` is not only rendering. It also fetches market data, computes technical indicators, validates text, optionally regenerates text, generates charts, and assembles report data. 

### 4. Text is weakly grounded

The generated narrative sections are not strongly tied to explicit supporting facts. A better system should connect every numerical claim to a metric, formula, source file, API response, and timestamp.

### 5. Data source disclosure can be broader than the actual pipeline

The report-level data source text may mention sources like company filings, FMP, Yahoo Finance, and AI4Finance estimates, but in the traced two-script workflow, the core financial data path is primarily FMP-based. 

---

## 10. Lessons for Building My Own Financial Data Agent

The biggest lesson I take from FinRobot is not “use more agents.” It is that a financial agent lives or dies by its data layer.

For my own project, I would design around these principles:

```text
1. Fetch all external data before rendering.
2. Save raw data snapshots.
3. Make every metric formula explicit.
4. Attach source metadata to every number.
5. Separate structured facts from narrative text.
6. Make forecast assumptions explicit and traceable.
7. Allow manual overrides, but label them clearly.
8. Generate narratives from grounded facts, not loose CSV context.
9. Make the report renderer deterministic.
```

A minimal source-grounded fact object could look like this:

```json
{
  "fact_id": "revenue_growth_fy2024",
  "company": "Example Corp",
  "metric": "Revenue Growth",
  "period": "FY2024",
  "value": 0.082,
  "formula": "(Revenue_FY2024 - Revenue_FY2023) / Revenue_FY2023",
  "source": {
    "provider": "FMP",
    "file": "income_statement_raw_data.csv",
    "fields": ["revenue"],
    "periods": ["FY2023", "FY2024"],
    "fetched_at": "2026-05-15T00:00:00Z"
  }
}
```

Then a narrative sentence such as:

```text
Revenue grew 8.2% year over year in FY2024.
```

should point back to that fact object.

This is the difference between a report generator and a research agent.

---

## Conclusion

FinRobot is a useful open-source reference for understanding how an LLM-powered equity research report pipeline can be assembled. It shows how to combine financial APIs, metric calculation, peer comparison, news, sentiment, LLM-generated text, charts, and HTML rendering into a complete workflow.

However, after running it and tracing the code, I would describe its equity module as a two-stage report-generation pipeline rather than a fully source-grounded financial research agent.

Its main engineering value is that it works end to end.

Its main weakness is that data lineage, forecast assumptions, runtime auto-fetching, and narrative generation are not cleanly separated.

For a production-grade buy-side agent, I would want a stricter architecture:

```text
frozen data snapshot
explicit formulas
traceable assumptions
source-grounded facts
deterministic rendering
auditable narrative claims
```

That is where the real engineering challenge begins.

[1]: https://github.com/AI4Finance-Foundation/FinRobot "GitHub - AI4Finance-Foundation/FinRobot: FinRobot: An Open-Source AI Agent Platform for Financial Analysis using LLMs    · GitHub"
