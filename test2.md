Top. Omdat `sharepoint_extraction_docint.py` nu **naast je notebook in de root staat**, kunnen we alle repo-/Git-/padlogica weggooien.

Hier is de opgecleanede notebook in **5 stappen**. Doel: bestaande `SharePointDataFetcher` en bestaande DI-client hergebruiken, documenten via de bestaande flow ophalen, `result.tables` bekijken en één compacte inventory opleveren.

### Stap 1 — Imports

```python
# ============================================================
# 1. IMPORTS
# ============================================================

import inspect
import pandas as pd

from azure.ai.documentintelligence.models import DocumentContentFormat
from sharepoint_extraction_docint import SharePointDataFetcher

print("✓ Imports OK")
```

### Stap 2 — Bestaande fetcher initialiseren

```python
# ============================================================
# 2. BESTAANDE CHATAPG FETCHER
#
# Deze klasse bevat al:
# - configuratie
# - credentials
# - document-download
# - Azure Document Intelligence client
# ============================================================

print("SharePointDataFetcher constructor:")
print(inspect.signature(SharePointDataFetcher))

fetcher = SharePointDataFetcher()

print("✓ SharePointDataFetcher klaar")
print("✓ Document Intelligence client aanwezig:", hasattr(fetcher, "docint_client"))
```

### Stap 3 — Bestaande documentflow draaien

Deze cell zoekt alleen naar de bestaande publieke flow; hij bouwt **geen nieuwe downloadroute**.

```python
# ============================================================
# 3. BESTAANDE DOCUMENTFLOW GEBRUIKEN
# ============================================================

preferred_entrypoints = [
    "fetch",
    "load",
    "load_data",
    "fetch_data",
    "process",
    "process_data",
    "get_data",
]

entrypoint = next(
    (
        name
        for name in preferred_entrypoints
        if hasattr(fetcher, name) and callable(getattr(fetcher, name))
    ),
    None,
)

if entrypoint is None:
    available = [
        name
        for name in dir(fetcher)
        if not name.startswith("_") and callable(getattr(fetcher, name))
    ]
    raise RuntimeError(
        "Geen bekende publieke entrypoint gevonden.\n"
        f"Beschikbare methods:\n{available}"
    )

print(f"✓ Bestaande entrypoint gevonden: {entrypoint}()")

getattr(fetcher, entrypoint)()

documents = [
    item
    for item in getattr(fetcher, "data_to_be_processed", [])
    if item.get("temp_ref")
]

print(f"✓ Documenten beschikbaar: {len(documents)}")
```

### Stap 4 — Document Intelligence-tabellen inventariseren

Dit is feitelijk jouw spike: we pakken dezelfde `prebuilt-layout` call, maar kijken nu naar `result.tables` in plaats van alleen `result.content`.

```python
# ============================================================
# 4. DOCUMENT INTELLIGENCE — TABLE INVENTORY
# ============================================================

inventory = []

for document_index, item in enumerate(documents, start=1):

    document_name = (
        item.get("name")
        or item.get("title")
        or item.get("filename")
        or item.get("temp_ref")
    )

    try:
        with open(item["temp_ref"], "rb") as f:
            result = fetcher.docint_client.begin_analyze_document(
                "prebuilt-layout",
                body=f,
                output_content_format=DocumentContentFormat.MARKDOWN,
            ).result()

        for table_index, table in enumerate(result.tables or [], start=1):

            cells = table.cells or []

            inventory.append({
                "document": document_name,
                "table": table_index,
                "rows": table.row_count,
                "columns": table.column_count,
                "cells": len(cells),

                "header_cells": sum(
                    getattr(cell, "kind", None)
                    in {"columnHeader", "rowHeader", "stubHead"}
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

        print(
            f"[{document_index}/{len(documents)}] "
            f"{document_name}: {len(result.tables or [])} tabel(len)"
        )

    except Exception as exc:
        print(f"SKIP — {document_name}: {exc}")


tables_df = pd.DataFrame(inventory)

print("\n✓ Document Intelligence analyse afgerond")
```

### Stap 5 — Resultaat en eerste impactsignalen

```python
# ============================================================
# 5. RESULTAAT — QUICK SCAN
# ============================================================

print("=" * 72)
print("DOCUMENT INTELLIGENCE — TABLE IMPACT QUICK SCAN")
print("=" * 72)

print(f"Documenten bekeken : {len(documents)}")
print(f"Tabellen gevonden  : {len(tables_df)}")

if tables_df.empty:

    print("\nGeen tabellen gevonden.")

else:

    display(tables_df)

    print("\nSNELLE SIGNALEN")
    print("-" * 40)

    print(f"Grootste tabel           : {tables_df['rows'].max()} rijen")
    print(f"Meeste kolommen          : {tables_df['columns'].max()}")
    print(f"Tabellen met headers     : {(tables_df['header_cells'] > 0).sum()}")
    print(f"Tabellen met merged cells: {(tables_df['merged_cells'] > 0).sum()}")
    print(f"Tabellen met lege cellen : {(tables_df['empty_cells'] > 0).sum()}")

    print("\nEERSTE IMPACT OP DE PIPELINE")
    print("-" * 40)
    print("1. Grote tabellen kunnen invloed hebben op chunk-grootte.")
    print("2. Merged cells / complexe headers kunnen structuur verliezen bij flattening.")
    print("3. Lege cellen moeten onderscheiden blijven van 0 of '-'.")
    print("4. Headers/context moeten waarschijnlijk behouden blijven in table-nodes.")
    print("5. Werkelijke tabelvormen bepalen of 'atomic table block' praktisch blijft.")
```

Dit is nu de notebook die ik zou bewaren als jouw **junior-DS spike**. Geen Git, geen repo-routing, geen handmatige documentpaden en geen productiecode wijzigen.

Als stap 2 of 3 nog één fout geeft, hoef je daarna ook niet opnieuw het hele ontwerp te veranderen: die fout vertelt ons exact welke bestaande constructor/entrypoint jouw gekopieerde klasse gebruikt.
