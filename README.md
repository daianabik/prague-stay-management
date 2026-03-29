# Prague Stay Management — Big Data Project

Datový projekt pro fiktivní firmu **Prague Stay Management**, která se zabývá správou a krátkodobým pronájmem bytů v Praze.

## Cíl projektu

Využít data z platformy Airbnb a otevřených zdrojů k optimalizaci portfolia nemovitostí prostřednictvím datové analytiky — dynamické oceňování, snižování ztrát a strategický výběr nemovitostí.

---

## Struktura projektu

```
prague-stay-management/
│
├── data/
│   ├── raw/                          # Zdrojová data (neupravená)
│   │   ├── listings_csv.gz           # Airbnb listings – Praha (Inside Airbnb)
│   │   ├── calendar_csv.gz           # Airbnb kalendář dostupnosti
│   │   └── metro_stations.csv        # Stanice metra Praha (AI-generováno)
│   │
│   ├── synthetic/                    # Syntetická data
│   │   ├── bookings.csv              # Rezervace odvozené z calendar + syntetický status
│   │   └── expenses.csv             # Provozní náklady (syntetické, korelované s listings)
│   │
│   └── processed/                    # Předzpracovaná data připravená pro analýzu
│       └── listings_clean.csv
│
├── database/
│   └── prague_stay.duckdb            # Finální databáze (odevzdání)
│
├── notebooks/
│   ├── 01_data_understanding_eda.ipynb   # EDA, distribuce, mapa, korelace
│   ├── 02_data_preparation.ipynb         # Čištění, syntetika, načtení do DuckDB
│   └── 03_kpi_analysis.ipynb             # Agregace a výpočet KPI
│
├── docs/
│   └── index.html                    # Průvodní dokumentace (odevzdání)
│
├── requirements.txt
└── README.md
```

---

## KPI

| # | KPI | Definice |
|---|-----|----------|
| 1 | **RevPAR** | Průměrný roční příjem na dostupný byt; vliv vzdálenosti od metra |
| 2 | **Vacancy Loss Rate** | Podíl prázdných dní v sezóně; identifikace rizikových měsíců |
| 3 | **Capacity Utilization** | Využití kapacity velkých bytů (6+ osob) vs. malých |

---

## Datové zdroje

| Dataset | Zdroj | Typ |
|---------|-------|-----|
| `listings_csv.gz` | [Inside Airbnb](http://insideairbnb.com) | Reálná |
| `calendar_csv.gz` | [Inside Airbnb](http://insideairbnb.com) | Reálná |
| `metro_stations.csv` | Generováno AI | Syntetická |
| `bookings.csv` | Odvozeno z calendar + syntetický status | Semi-syntetická |
| `expenses.csv` | Generováno AI (korelováno s listings) | Syntetická |

---

## Instalace

```bash
pip install -r requirements.txt
```

---

## Spuštění

```bash
jupyter notebook
```

Otevřete notebooky v pořadí: `01` → `02` → `03`.

---

## Tým

Projekt vypracován v rámci předmětu **Big Data** — skupina 3 členů.

---

## Nástroje

- Python 3.11
- Pandas, NumPy, Matplotlib, Seaborn, Folium
- DuckDB
- Jupyter Notebook
- Generativní AI (Claude by Anthropic)
