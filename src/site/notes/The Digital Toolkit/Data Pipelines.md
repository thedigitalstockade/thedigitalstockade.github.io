---
{"dg-publish":true,"permalink":"/the-digital-toolkit/data-pipelines/","title":"Building Data Pipelines That Honor the Past","tags":["methodology","python","historical-data","digital-humanities","digital-toolkit"]}
---



---

# The Immutable Archive: Building Archival Data Pipelines in Python


After transcribing Australian Colonial records, we inevitably end up with spreadsheets. They're useful for preliminary analysis, but to unlock deeper insights, to ask questions that span multiple domains and time periods, the data must move into a structured database. The challenge is that historical records are messy, and that messiness matters.

The historical record is filled with blank dates, misspelled names, human error, and editorial marks that standard data tools can't see. That's fine, but from a database point of view these may not play nicely with our analysis goals.

This leaves us with a few options. The first and most dangerous option when faced with an inconsistent CSV or Excel file is to manually "fix" it before import. This can leave you in a world of pain. Doing this effectively means we are overwriting the original archivist's reality with our own, obscuring the very real uncertainties of the past and destroying analytical data along the way. Name misspellings can give us clues about dialects or education levels for instance. If you must modify the source, make a copy of the column you are editing and make the corrections in the copy. The downside is that manual cleaning is not repeatable. When you discover an error in your logic three months later, you have to remember what you changed and why. Consistency across large datasets becomes nearly impossible.

Another way is to import the data as-is and deal with it within the database itself. This is a perfectly sound methodology if your database design accounts for it, and has been deployed on many large scale analysis projects I have witnessed. The advantage is that your source remains untouched and all your cleaning logic lives in one place. The disadvantage is that you need a more sophisticated database design to handle the messiness. If you're doing heavy transformation at query time, your analysis becomes slower. And if your database is being used by non-technical researchers, they may struggle to understand why the data looks the way it does.

There's also another option worth mentioning: using a dedicated data cleaning tool like OpenRefine or Trifacta. These sit between your spreadsheet and your database, offering a visual interface for cleaning and transformation and creating logs of all the activities undertaken. They're excellent for one-off projects or when you need to involve non-programmers in the cleaning process. The trade-off is that they can be harder to version control (especialy if multiple people are involved) and audit than code-based pipelines.

The last option, and the one we are talking about here, is to leave the mess in the source and clean it as part of a repeatable, documented import process. This is a nice halfway place between the other options. It allows for better documentation and repeatability than manual cleaning, but allows you to quickly spin up an analysis database with pre-tidied data, thus simplifying the database design process. Your cleaning logic is version controlled, auditable, and repeatable. When you discover a new dataset that follows the same pattern, you can reuse 90 percent of your code.

This documented cleaning is what is commonly called a **Data Pipeline**.

In software engineering, an ETL (Extract, Transform, Load) pipeline automates three steps: extracting raw data, transforming it into a standard format, and loading it into a database. In historical research, it serves a different purpose. It acts as a transparent, repeatable translation layer between the messy reality of the archive and the rigid architecture of our databases.

For this project, we're structuring the data into an Entity-Attribute-Value (EAV) format. This is a flexible model that captures historical facts as discrete, queryable records. (I'll cover EAV in detail in a separate post, but for now, think of it as a way to store "Person X had Y amount of Z in year W" as a single, standardized row.) Later, we flatten this EAV structure into a researcher-friendly index that makes querying intuitive for non-technical users.

## The Architecture: Separating the Rules from the Engine

A common trap in digital history is writing a brand-new, custom **Python** (insert your language of choice but I'm generally a Python guy) script for every single spreadsheet. If a project relies on 50+ different archival ledgers, managing 500 different processing scripts quickly becomes a maintenance nightmare.

Instead, a robust data pipeline utilizes strict Separation of Concerns. It divides the workflow into two distinct components: the Blueprints (the historical rules) and the Engine (the processing logic).

Here's how data actually flows through the Python pipeline for early Colonial datasets that I made recently:

```text
=========================================================================
                    Example ARCHIVAL DATA PIPELINE
=========================================================================

 [THE BLUEPRINTS]
 (Python config dictionaries)
 - sheet_name, skip_rows, nrows
 - uid_col, year_col, domain
 - facts array with transform rules
            |
            v
 [THE EXECUTION QUEUE]
 (Process blueprints in sequence)
            |
            +---> Blueprint 1 (1805 Land Muster)
            |     Blueprint 2 (1809 Hobart Muster)
            |     Blueprint 3 (1798 Maize)
            |     Blueprint 4 (1798 Swine)
            |     ... (n tabs+ total)
            |
            v
 [Master Excel File]
 (Reads all tabs, immutable source)
            |
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
 [pandas: Read Data]         [openpyxl: Read Visual]  [Identify Dataset
 (Respecting skip_rows       (Detect strikethrough    Region (skip_rows
  and nrows from blueprint)   marks from researchers)  and nrows)]
            |                     |                     |
            +---------------------+---------------------+
                          |
                          v
            [THE STRIKETHROUGH GATEKEEPER]
            (Skip rows marked invalid by researchers)
                          |
                          v
            [THE TRANSFORMATION ROUTER]
            (Apply transform rules from blueprint)
            - direct: pass value through
            - sum: aggregate multiple columns
            - regex_extract: parse text patterns
                          |
                          v
            [BUILD EAV RECORDS + JSON PAYLOAD]
            (Entity_ID, Fact_Name, Fact_Value_Numeric,
             Fact_Value_Text, Fact_Qualifier, Raw JSON Payload of original)                      |
                          v
            [ACCUMULATE RESULTS]
            (Append to results from previous blueprints)
                          |
            (Loop back to next blueprint in queue)
                          |
                          v
 [CONCATENATE ALL BATCHES]
 (Merge results from all  blueprints)
            |
            v
 [GENERATE OUTPUTS]
 - EAV_Import_Ready.txt (tab-delimited)
 - Data_Dictionary.csv (unique fact names)


```

By understanding this top-down architecture, we can establish five golden rules for processing historical data. These rules ensure we structure the past without erasing its complexities.

## Rule 1: The Principle of Immutability

Notice in the flowchart that data flows only away from the raw Excel file. The raw transcription is immutable, a read-only artifact. If an 1805 muster contains a typo, that typo is a historical fact. We preserve it.

We never alter the source files. Instead, the pipeline's extraction engine reads the messy reality into memory and translates it there. Whilst it is not shown in the code snippet, to guarantee absolute traceability, the pipeline does something crucial: as it reads the raw, unmodified data, it serializes the entire **original Excel row as a JSON string** and stores it directly alongside the newly extracted EAV fact. This means a researcher looking at the final database doesn't just see the cleaned data; they can instantly view the exact, untouched text that generated it. If our interpretation of a colonial date format turns out to be flawed next year, our original data remains untouched. We simply update the Python script and run the pipeline again.


```python
# The raw Excel file is never modified
df = pd.read_excel(
    master_excel_file,
    sheet_name=config['sheet_name'],
    skiprows=config.get('skip_rows', 0),
    nrows=config.get('nrows', None)
)
# All transformations happen on this in-memory copy
```


## Rule 2: The Translation Layer (The Blueprints)

To bridge the gap between human messiness and database architecture, the pipeline utilizes Python dictionaries as processing Blueprints.

Instead of writing dense code that hides historical assumptions, we map out exactly how the engine should interpret a specific document. Here's a real example: a blueprint designed to parse an 1805 Norfolk Island land muster. It tells the engine exactly which Excel tab to target, how to identify the individual, and how to translate specific columns into measurable database attributes.




```python
config_1805_land = {
    'sheet_name': '24) a land record sheet',
    'skip_rows': 2,
    'uid_col': 'uid',
    'source_uid_col': 'doc_uid',
    'year_col': 'event_date',
    'is_state': 1,
    'domain': 'Agricultural',
    'facts': [
        {'source_col': 'Land Culivation', 'fact_name': 'Acres_Cultivated', 'type': 'numeric'},
        {'source_col': 'Bulls and Cows', 'fact_name': 'Livestock_Cattle', 'type': 'numeric'},
        {'source_col': 'Sheep', 'fact_name': 'Livestock_Sheep', 'type': 'numeric'}
    ]
}

```

By reading this blueprint, the overarching logic becomes entirely transparent.

Targeting the Data: `sheet_name` and `skip_rows` direct the engine to the exact Excel tab and instruct it to bypass the first two rows of messy header notes.

Anchoring the Entity: `uid_col` and `year_col` ensure that every extracted fact is permanently tied to a specific historical individual and a specific point in time.

Translating the Facts: The `facts` array acts as the actual translation dictionary. It explicitly tells the engine, "When you read the colonial column labeled 'Bulls and Cows', translate it to `Livestock_Cattle` in the database, and ensure the value is strictly numeric."

Classifying the Record: `is_state` (1 = a snapshot of conditions; 0 = a singular event) and `domain` (Agricultural, Judicial, Vital, etc.) add historical context that researchers need.

Anyone reviewing the code can see precisely how the data was derived. Furthermore, when a newly transcribed 1807 muster is uncovered, we do not need to rewrite the software engine. We simply write a new 15-line blueprint and add it to the Execution Queue.

## Rule 3: The Transformation Router (Beyond Direct Mapping)

The simplest blueprint uses `'transform': 'direct'`. It reads a column and writes its value as-is. But real historical data is rarely that straightforward. The Transformation Router handles three distinct transformation types, each solving a different archival problem.

### Transform Type 1: Direct Mapping

The most common case. Read a column, validate it exists, and pass it through.
```python
{'source_col': 'Pardon', 'fact_name': 'Pardon_Type', 'type': 'text'}
```

The engine checks: Does this cell have a value? If yes, extract it without modification and map it to a database field. If no, skip the record.

### Transform Type 2: Sum (Aggregating Multiple Columns)

Sometimes historical clerks split a single concept across multiple columns. In this 1809 Muster example, swine counts are recorded separately by sex.

```python
config_1809_hobart_muster = {
    'sheet_name': '30) 1809 Muster',
    'skip_rows': 2,
    'uid_col': 'uid',
    'year_col': 'event_date',
    'is_state': 1,
    'domain': 'Agricultural',
    'facts': [
        {'source_cols': ['Swine Male', 'Swine Female'], 
         'fact_name': 'Livestock_Swine', 
         'type': 'numeric',
         'qualifier': 'Total',
         'transform': 'sum'}
    ]
}
```

The engine reads both columns, cleans the numeric values, sums them, and creates a single EAV record with `Fact_Qualifier: 'Total'`. This preserves the original structure while creating a queryable aggregate.

### Transform Type 3: Regex Extract (Parsing Messy Text)

Colonial clerks often embedded data within prose. A sentence might read "Sentenced to 50 lashes" when we only want the number. The regex (with which I have a love hate relationship) transform below extracts it.

```python
config_1791_punishments = {
    'sheet_name': '07) punishments 1791 - 1794',
    'skip_rows': 2,
    'uid_col': 'uid',
    'year_col': 'Year',
    'is_state': 0,
    'domain': 'Judicial',
    'facts': [
        {'source_col': 'Sentence', 
         'fact_name': 'Lashes_Sentenced', 
         'type': 'numeric',
         'transform': 'regex_extract',
         'pattern': r'^(\d+)'}
    ]
}
```

The pattern `r'^(\d+)'` means: "Find one or more digits at the start of the string and extract them." If the cell contains "50 lashes", the engine extracts `50`. If it contains "remitted" or is blank, the record is skipped.

Each transformation type is a deliberate choice. By making these choices explicit in the blueprint, we document our historical interpretation. Future researchers can see exactly how we parsed ambiguous data.

## Rule 4: Detecting Researcher Edits (The Strikethrough Gatekeeper)

The blueprint-and-engine architecture handles standard extraction well. But here's where it breaks down. During the transcription and review process, researchers sometimes mark records as invalid by drawing a strikethrough in Excel. This is not the original clerk's mark. It's our editorial decision, made during data preparation. When `pandas` reads that Excel file, it sees only the text, not the strikethrough. To the computer, a crossed-out row is still a valid event. Our engine needs to see what the researcher marked as invalid.

This is crucial for data integrity. If a researcher identified a duplicate entry, a transcription error, or a record that doesn't belong in the dataset, they marked it with a strikethrough. Our pipeline must respect that editorial decision and exclude those records from the database.

To prevent our database from populating with records that researchers flagged as invalid, the Master Processing Engine must read both the textual data and the visual intent. Inside the engine, the script runs `pandas` to process the text, while simultaneously utilizing the `openpyxl` library to inspect the visual formatting of the researcher's edits.

```python
# ==========================================
# THE STRIKETHROUGH GATEKEEPER (Inside the Engine)
# ==========================================
if ws:
    excel_row_num = config.get('skip_rows', 0) + index + 2
    is_crossed_out = False

    # Check the first 5 columns for a strikethrough font
    for col_idx in range(1, 6):
        cell = ws.cell(row=excel_row_num, column=col_idx)
        if cell.font and cell.font.strike:
            is_crossed_out = True
            break

    if is_crossed_out:
        continue  # Skip this record entirely
# ==========================================
```

### Anatomy of the Gatekeeper

To understand how this script respects researcher edits, let's break down what's happening.

**1. Synchronizing the Two Worlds**

Our pipeline reads the Excel file twice simultaneously: once with `pandas` (for the raw text) and once with `openpyxl` (for the visual formatting). The problem is that `pandas` counts rows starting at 0, while Excel starts counting at 1. Furthermore, our blueprint told `pandas` to skip the first two rows of messy headers. To ask `openpyxl` to check the visual formatting of the current row, we have to calculate exactly where that row sits in the physical spreadsheet.

```python
excel_row_num = skip_rows + pandas_index + 2
```

This formula aligns the two counting systems so we're checking the same row in both libraries.

**2. The Default Assumption**

We start by assuming the historical record is valid and active (`is_crossed_out = False`). Only evidence of a strikethrough changes this assumption.

**3. Scanning the Ink**

We don't need to check every single cell in a 30-column ledger to know if a row was marked invalid. When a researcher struck a line through an entry, they almost always crossed out the individual's name or ID in the first few columns. To save processing power, we instruct the script to only look at the first five columns (`range(1, 6)`).

**4. Detecting the Mark**

This is the core of the extraction. The script looks at the underlying XML metadata of the specific Excel cell. It asks two questions: Does this cell have custom font formatting? And if so, does that formatting include a strikethrough? If it finds a line drawn through the text, it flips our assumption to `True` and breaks the scanning loop.

**5. Honoring the Edit**

In Python, the `continue` command tells a loop to immediately stop what it is doing and move on to the next item. If our gatekeeper detected a strikethrough, hitting `continue` means the script completely abandons extracting that row. The record is never translated, and the researcher-marked invalid event never makes it into our clean database.

By walking through the logic line-by-line, we demystify the programming. It proves that handling complex archival data doesn't require impenetrable computer science. It just requires a logical, methodical translation of a researcher's editorial process into digital steps.

## Rule 5: Multiple Datasets on a Single Sheet (Handling Stacked Data)

Sometimes a single Excel tab contains multiple logical datasets stacked vertically. These datasets have different column headers and are separated by blank rows. For example, a single tab might contain production data for two different crops, each with its own header row and column structure.

Rather than manually splitting the file or writing separate transcriptions, we use the `skip_rows` and `nrows` parameters to create separate blueprints for each dataset region.

```python
config_1798_maize = {
    'sheet_name': 'sheet with Maize and Swine',
    'skip_rows': 2,
    'nrows': 18,  # Only read 18 rows
    'uid_col': 'uid',
    'is_state': 1,
    'domain': 'Agricultural',
    'facts': [
        {'source_col': 'Maize (Bushels)', 'fact_name': 'Crop_Maize_Bushels', 'type': 'numeric'}
    ]
}

config_1798_swine = {
    'sheet_name': 'sheet with Maize and Swine',
    'skip_rows': 20,  # Start at row 20, after blank space
    'uid_col': 'uid',
    'is_state': 1,
    'domain': 'Agricultural',
    'facts': [
        {'source_col': 'Swine Count', 'fact_name': 'Livestock_Swine', 'type': 'numeric'}
    ]
}
```

The engine processes the same physical tab twice, but with different `skip_rows` and `nrows` parameters. The first blueprint skips the first 2 rows and reads exactly 18 rows of maize data. The second blueprint skips the first 20 rows (accounting for the blank space between datasets) and reads the swine data from that point onward.

This approach keeps the source file clean and immutable while allowing us to logically partition the data as needed. Each dataset gets its own blueprint, its own transformation rules, and its own domain classification. When researchers later ask "How many swine were recorded in 1798?", the database knows exactly which records came from which dataset region and can answer with confidence.

## The Execution Queue: Batch Processing Multiple Blueprints

Rather than running individual scripts for each dataset, we queue all blueprints and process them in sequence.

```python
processing_queue = [
    config_1802_births_deaths,
    config_1803_victualling,
    config_1805_land,
    config_1807_land_muster,
    config_1809_hobart_muster,
    config_1791_punishments,
    config_old_system_pardons,
    config_1807_vdl_evacuation,
    config_1792_stowaways,
    config_adm36_navy_musters,
    config_1798_maize,
    config_1798_swine,
    config_ap_1800_part1
]

for config in processing_queue:
    df_result = process_historical_sheet(master_file, config)
    if not df_result.empty:
        all_eav_dataframes.append(df_result)
```

The engine processes each blueprint in order, accumulating the results. At the end, all EAV records are concatenated into a single tab-delimited file ready for database import. The pipeline also generates a `Database_Data_Dictionary.csv` containing every unique `Fact_Name` across all datasets. This reference guide helps researchers understand what facts are available in the database and where they came from.

## Conclusion: One Path Among Many

This pipeline works for our project. It might work for yours. But it's not the only way to solve this problem.

If you're building something similar, you have options. Tools like OpenRefine, Talend, and Apache NiFi can handle much of this work without writing Python code. For smaller projects, a well-designed database with careful query logic might be all you need. For larger ones, you might want to invest in a proper data warehouse solution. The point isn't to evangelize Python or pipelines specifically. The point is to be intentional about where your cleaning happens and to document it thoroughly.

Handling historical data at scale requires more than just careful transcription; it requires rigorous, repeatable methodologies. While visual cleaning tools or complex database queries have their place, a code-based data pipeline offers unparalleled transparency and auditability for historical research.

By explicitly separating our historical assumptions (the blueprints) from our computational logic (the engine), we create a system that is robust, scalable, and inherently documented. This architecture allows us to capture the nuances of colonial records—from tracking strikethroughs to serializing raw JSON payloads—without ever compromising the immutable source material.

If you are drowning in spreadsheets and trying to figure out how to get them into a database without destroying the historical context, the solution is not to manually clean harder. The solution is to invest your time in proper tooling and documentation. Future researchers will thank you for it.


---
### Cite this post

If you found this methodology useful for your own work, you can cite it here:

> McLean, Mark A. "The Immutable Archive: Building Archival Data Pipelines in Python." *The Digital Stockade*. Published 2026-03-28. https://thedigitalstockade.github.io/the-digital-toolkit/data-pipelines/