# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static HTML file, `moco-xml-tool.html`, with no build step and no dependencies. It runs entirely client-side in the browser: a user drops in an XML export from MOCO (a project/invoicing tool) and the tool enriches it so it's ready for import into ABACUS (Swiss accounting software), then offers the result as an XML download and an optional `.xlsx` summary ("Abrechnungsliste").

There is no package.json, no bundler, no linter, and no test runner. All HTML/CSS/JS lives inline in `moco-xml-tool.html`.

## Commands

- **Run the tool**: open `moco-xml-tool.html` directly in a browser (no server needed).
- **Run the self-test suite**: open the file, expand "Selbsttests" and click "Selbsttests ausführen" — or run `runMocoSelfTests()` in the browser devtools console (it's exposed on `window`). There is no CLI test runner; verifying changes to the processing logic means running this suite in an actual browser.
- When changing the core transform logic, add a corresponding `testN` function (see the existing `test1`...`test26`) and register it in the `tests` array inside `runMocoSelfTests()` — this is the only regression safety net in the project.

## Architecture

### Input format

The expected input is an `AbaConnectContainer` XML export containing one or more `Task` elements, each with one or more `Transaction` children. Two task shapes matter:

- `Task > Transaction > Document` — one invoice ("Abrechnung") per transaction.
- `Task > Transaction > DocumentDossier > Dossier` — one file/dossier entry per transaction, optional (a whole export can have zero of these).

All DOM traversal is done by `localName`, never by prefixed tag name (`getDirectChildrenByLocalName` / `getDirectChildTextByLocalName`), because the namespace prefix varies between MOCO exports. Don't reintroduce namespace-prefix-based lookups.

### CONFIG-driven transform

`CONFIG` (top of the `<script>`) is the single source of truth for element names, the fields to enforce, and the DocumentName→Number rename. Business-rule changes (new required field, new value, different rename) should go through `CONFIG` and the generic helpers (`ensureField`, `renameElement`), not one-off code paths.

### Processing pipeline (`processXmlString`)

Runs in this order, and the order matters because later steps depend on values captured before earlier steps mutate them:

1. Parse and validate the root element.
2. Find all `Document` invoices; for each, apply `CONFIG.fields` (currently `Division`, `PaymentOrderProcedure`) via `ensureField` — existing correct values are left in place, existing wrong values are corrected, missing fields are inserted after a configured anchor sibling. The invoice's **original** `Number` value is captured here (`originalNumber`) before anything can change it.
3. Find all dossier entries; rename `DocumentName` → `Number` inside `Dossier` (`CONFIG.rename`). Entries that fail (duplicate/missing field) are excluded from the next step.
4. Transfer the MOCO document number from each successfully-renamed dossier entry into the matching invoice's `Document > Number`, joined via the *pre-transform* invoice `Number` against the outer `DocumentDossier > Number` (the "Beleg-Nummer"). A dossier entry with no matching invoice is not an error (dossier data is optional); two dossier entries claiming the same Beleg-Nummer is an error and fails only that invoice.
5. Build supplementary lookups (customer master data by `CustomerNumber`, dossier/file info by the pre-transform Beleg-Nummer — **not** the now-overwritten `Number`) and merge them into a flattened per-invoice row. This only feeds the `.xlsx` export; it never touches the XML output.
6. Serialize back to XML, re-attaching the original `<?xml ...?>` declaration.

Failures are per-item, not fatal: a structurally broken invoice or dossier entry is skipped with a recorded error message, and the rest of the batch still processes. `result` aggregates counts (`processedCount`, `errorCount`, `dossierRenamedCount`, `documentNumberStats`, …) plus per-error label/message lists, which the UI renders as-is in the result overview.

### XLSX export

`buildAbrechnungslisteXlsx` hand-rolls a minimal OOXML package (Content_Types, workbook, one worksheet) and zips it with a from-scratch, uncompressed ("stored" method) ZIP writer (`crc32`, `buildZip`). There is no external library involved — don't assume one is available or add one; the whole point was zero dependencies.

### Deployment / embedding constraint

The tool is embedded in Confluence Cloud via its native "Iframe" macro. That macro sandboxes the iframe without `allow-downloads`, which silently breaks the JS-triggered file download (`Blob` + `<a download>` click) inside Confluence — this is a Confluence-imposed browser restriction, not something fixable from inside the page alone, and the native macro exposes no sandbox/permission settings. No workaround is implemented yet; if asked to fix Confluence downloads, the viable path is an in-page "open in a real tab" escape hatch, not a sandbox-attribute change.

## Repository layout notes

- `example/` contains real-world-shaped MOCO XML samples used for manual verification against the actual `Task`/`Transaction`/`Document`/`DocumentDossier` structure — useful for understanding the schema, not part of the shipped tool.
