Mooi. Dan is je doel nu scherp genoeg. We gebruiken **geen incremental/datumroute**, wijzigen **geen productiecode**, kiezen automatisch maximaal **5 bestaande documenten**, gebruiken de bestaande downloader + DI-client en geven alleen een compacte inventory en impactsignalen.

Gebruik dit als je opgeschoonde notebook:

```python
# ============================================================
# 1. SETUP
# ============================================================

import os
import pandas as pd

os.environ["CONFIG"] = "config/config.yml"

from sharepoint_extraction import SharePointDataFetcher

fetcher = SharePointDataFetcher()

KENNISBANK = "PU"
CATEGORY = "PWRI"
MAX_DOCUMENTS = 5

print("✓ Setup klaar")
```

```python
# ============================================================
# 2. BESTAANDE METADATA OPHALEN — GEEN DATUMFILTER
# ============================================================

metadata = fetcher.get_indexable_source_metadata(
    KENNISBANK,
    CATEGORY,
)

items = list(metadata.values())

documents = [
    item
    for item in items
    if item.get("file_type") == "document"
][:MAX_DOCUMENTS]

print(f"Metadata-items : {len(items)}")
print(f"Testdocumenten : {len(documents)}")

if not documents:
    raise RuntimeError("Geen documenten gevonden in de bestaande metadata.")
```

```python
# ============================================================
# 3. BESTAANDE DOWNLOADROUTE GEBRUIKEN
# ============================================================

fetcher.data_to_be_processed = documents
fetcher._batch_download_documents()

documents = [
    item
    for item in fetcher.data_to_be_processed
    if item.get("temp_ref")
]

print(f"✓ Gedownload: {len(documents)} documenten")

if not documents:
    raise RuntimeError("Geen documenten succesvol gedownload.")
```

```python
# ============================================================
# 4. DOCUMENT INTELLIGENCE — TABLE INVENTORY
# ============================================================

rows = []

for item in documents:

    document_name = (
        item.get("name")
        or item.get("title")
        or item.get("source_page")
        or item.get("temp_ref")
    )

    with open(item["temp_ref"], "rb") as f:
        result = fetcher.docint_client.begin_analyze_document(
            "prebuilt-layout",
            body=f,
        ).result()

    for table_nr, table in enumerate(result.tables or [], start=1):

        cells = table.cells or []

        rows.append({
            "document": document_name,
            "table": table_nr,
            "rows": table.row_count,
            "columns": table.column_count,
            "cells": len(cells),

            "headers": sum(
                "header" in str(getattr(cell, "kind", "")).lower()
                for cell in cells
            ),

            "merged_cells": sum(
                (getattr(cell, "row_span", 1) or 1) > 1
                or (getattr(cell, "column_span", 1) or 1) > 1
                for cell in cells
            ),

            "empty_cells": sum(
                not (getattr(cell, "content", "") or "").strip()
                for cell in cells
            ),
        })


tables_df = pd.DataFrame(rows)
```

```python
# ============================================================
# 5. RESULTAAT + EERSTE IMPACTSIGNALEN
# ============================================================

print("=" * 70)
print("DOCUMENT INTELLIGENCE — TABLE QUICK SCAN")
print("=" * 70)

print(f"Documenten onderzocht : {len(documents)}")
print(f"Tabellen gevonden     : {len(tables_df)}")

if tables_df.empty:
    print("\nGeen tabellen gevonden in deze sample.")

else:
    display(tables_df)

    print("\nSNELLE SIGNALEN")
    print("-" * 45)

    print(f"Grootste tabel        : {tables_df['rows'].max()} rijen")
    print(f"Meeste kolommen       : {tables_df['columns'].max()}")
    print(f"Met headers           : {(tables_df['headers'] > 0).sum()}")
    print(f"Met merged cells      : {(tables_df['merged_cells'] > 0).sum()}")
    print(f"Met lege cellen       : {(tables_df['empty_cells'] > 0).sum()}")

    print("\nIMPACT VOOR VERVOLGONDERZOEK")
    print("-" * 45)

    print("1. Tabelomvang vs. huidige chunk size / atomic-block aanpak.")
    print("2. Headers moeten bij tabel- of rijnodes behouden blijven.")
    print("3. Merged cells kunnen betekenis verliezen bij simpele flattening.")
    print("4. Lege cellen moeten onderscheiden blijven van 0 en '-'.")
    print("5. Tabelcontext moet behouden blijven richting retrieval en LLM.")
```

Dit sluit nu ook aan op wat je laatste screenshots daadwerkelijk bewijzen: `get_indexable_source_metadata(kennisbank, category)` bouwt de volledige metadata zonder incremental-datumfilter, `_batch_download_documents()` werkt op `self.data_to_be_processed` en vult `temp_ref`, en daarna kunnen we direct naar de bestaande `docint_client`.

**Dit is de spike.** Geen extra infrastructuur meer.
