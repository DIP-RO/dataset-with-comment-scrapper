# Bangladesh Cricketer YouTube Stance Dataset Pipeline

This project collects YouTube comments, filters for complete comments that explicitly mention one target player, and labels stance data for:

- `shakib`
- `mushfiq`
- `riyad`
- `tamim`
- `murtaza`

Each CSV uses this schema:

```csv
id,text,target,stance
```

Allowed stance labels are `FAVOR`, `AGAINST`, and `NEUTRAL`.

## Pipeline Diagram

```mermaid
flowchart TD
    A[Define target cricketers and aliases] --> B[Generate YouTube search queries]
    B --> C[Discover controversy, politics, BCB, selection, retirement, and performance videos]
    C --> D{Layered network extraction}
    D --> D1[Primary video search and comment retrieval]
    D1 --> D2[Secondary retrieval fallback for failed videos]
    D2 --> D3{Video/comment extraction successful?}
    D3 -->|No| R0[Skip failed candidate and continue]
    R0 --> B
    D3 -->|Yes| E[Clean and normalize comment text]
    E --> F{Quality filter}
    F -->|Reject| R1[Discard short, emoji-only, link-only, spam-like, or incomplete comments]
    F -->|Accept| G{Specific player mention?}
    G -->|No or multiple players| R2[Discard ambiguous target comments]
    G -->|Exactly one player| H{Target-specific exclusion}
    H -->|Actor/movie Shakib Khan context| R3[Discard non-cricketer Shakib comments]
    H -->|Cricket context| I[Assign target player]
    I --> J[Annotate stance]
    J --> K{Stance label}
    K --> L[FAVOR]
    K --> M[AGAINST]
    K --> N[NEUTRAL]
    L --> O{Balance quota check}
    M --> O
    N --> O
    O -->|Quota full| R4[Discard to preserve class balance]
    O -->|Quota available| P[Save row: id, text, target, stance]
    P --> Q[Write per-player checkpoint CSV]
    Q --> S{3000 valid rows per player?}
    S -->|No| B
    S -->|Yes| T[Export final per-player CSV dataset]
```

For a paper, the pipeline can be summarized as query generation, layered network extraction, fault-tolerant video skipping, comment quality filtering, target-player validation, non-cricketer Shakib removal, stance annotation, class balancing, checkpointing, and final CSV export.

## Data Quality Rules

The scraper rejects comments that:

- have 6 words or fewer
- do not mention exactly one target player alias
- are emoji-only, link-only, spam-like, or mostly non-text
- do not look like a complete sentence
- mention multiple target players, because the target becomes ambiguous

## Video Discovery

Search is controversy-focused by default. The scraper looks for videos around:

- political and election-related cricket discussion
- BCB, selection, captaincy, retirement, ban, and scandal controversy
- performance criticism after major tournaments and matches
- press conferences, interviews, news analysis, debates, and fan reactions

All aliases for each player are used in search, including full names and spelling variants.
Bangla player-name aliases and Bangla controversy search terms are included.

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

For transformer fallback support:

```bash
pip install -r requirements-transformer.txt
```

Your `.env` should contain:

```bash
GEMINI_API_KEY=...
GEMINI_MODEL=gemini-2.5-flash
ANNOTATION_BATCH_SIZE=50
RANDOM_SEED=42
```

`GEMINI_MODEL` is optional. If omitted, the script uses `gemini-2.5-flash`.

## Run

Gemini primary with transformer fallback:

```bash
python scrape_stance_dataset.py
```

Gemini primary with keyword fallback:

```bash
python scrape_stance_dataset.py --fallback local
```

No API, local keyword rules only:

```bash
python scrape_stance_dataset.py --annotator local
```

Local transformer only:

```bash
python scrape_stance_dataset.py --annotator transformer
```

Outputs are saved in `data/`:

- `data/shakib.csv`
- `data/mushfiq.csv`
- `data/riyad.csv`
- `data/tamim.csv`
- `data/murtaza.csv`

Progress is saved continuously in `data/checkpoints/`, so you can stop and restart safely.

## Useful Options

```bash
python scrape_stance_dataset.py --per-player 3000
python scrape_stance_dataset.py --min-words 7
python scrape_stance_dataset.py --comments-per-video 800
python scrape_stance_dataset.py --videos-per-query 30
python scrape_stance_dataset.py --output-dir data
```

The scraper uses a layered network extraction design: search discovery, comment retrieval, text filtering, target validation, stance labeling, balancing, and checkpointed export are separated into recoverable stages. If a video or query fails, the pipeline skips it and continues from the next candidate without losing previously collected rows.

Run one player at a time:

```bash
python scrape_stance_dataset.py --players shakib
python scrape_stance_dataset.py --players mushfiq
python scrape_stance_dataset.py --players riyad
python scrape_stance_dataset.py --players tamim
python scrape_stance_dataset.py --players murtaza
```
