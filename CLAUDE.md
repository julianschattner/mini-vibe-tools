# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static HTML file, `moco-xml-tool.html`, with no build step and no dependencies. It runs entirely client-side in the browser: a user drops in an XML export from MOCO (a project/invoicing tool) and the tool enriches it so it's ready for import into ABACUS (Swiss accounting software), then offers the result as an XML download and an optional `.xlsx` summary ("Abrechnungsliste").

There is no package.json, no bundler, no linter, and no test runner. All HTML/CSS/JS lives inline in `moco-xml-tool.html`.

## Commands

- **Run the tool**: open `moco-xml-tool.html` directly in a browser (no server needed).
- **Run the self-test suite**: open the file, expand "Selbsttests" and click "Selbsttests ausführen" — or run `runMocoSelfTests()` in the browser devtools console (it's exposed on `window`). There is no CLI test runner; verifying changes to the processing logic means running this suite in an actual browser. (A one-off way to check headlessly: inject a `window.onload` handler that calls `runMocoSelfTests()` and writes the result to `document.title`, then load the file in `chrome.exe --headless=new --dump-dom` and read the title from the dumped DOM.)
- When changing the core transform logic, add a corresponding `testN` function (see the existing `test1`...`test52`) and register it in the `tests` array inside `runMocoSelfTests()` — this is the only regression safety net in the project.
- Tests that inspect the Dossier/DocumentDossier structure in `serializedXml` (`test17`-`test19`) must temporarily set `CONFIG.removeDossierTaskFromOutput = false` (restored in a `finally`) — by default that whole Task is stripped before serialization (see pipeline step 4b below), so those elements no longer survive into the output.
- `processXmlString(xmlText, options)` requires `options.generalLedgerDate` (format `YYYY-MM-DD`, validated by regex) — every test call site must pass it (see `TEST_GENERAL_LEDGER_DATE` in the self-tests). A missing/malformed value is a fatal result (`ok: false`), not a per-invoice error, since it's chosen once per run in the UI (`generalLedgerDateInput`, pre-filled with the last day of the previous month).

## Architecture

### Input format

The expected input is an `AbaConnectContainer` XML export containing one or more `Task` elements, each with one or more `Transaction` children. Three task shapes matter:

- `Task > Transaction > Document` — one invoice ("Abrechnung") per transaction.
- `Task > Transaction > DocumentDossier > Dossier` — one file/dossier entry per transaction, optional (a whole export can have zero of these).
- `Task > Transaction > Customer > AddressData` — one customer master-data entry per transaction.

All DOM traversal is done by `localName`, never by prefixed tag name (`getDirectChildrenByLocalName` / `getDirectChildTextByLocalName`), because the namespace prefix varies between MOCO exports. Don't reintroduce namespace-prefix-based lookups.

### CONFIG-driven transform

`CONFIG` (top of the `<script>`) is the single source of truth for element names, the fields to enforce, and the DocumentName→Number rename. Business-rule changes (new required field, new value, different rename) should go through `CONFIG` and the generic helpers (`ensureField`, `renameElement`), not one-off code paths.

### Processing pipeline (`processXmlString`)

Runs in this order, and the order matters because later steps depend on values captured before earlier steps mutate them:

1. Parse and validate the root element (and `options.generalLedgerDate`, fatal if missing/malformed).
2. Find all `Document` invoices; for each, apply `CONFIG.fields` (currently `Division`, `PaymentOrderProcedure`) via `ensureField` — existing correct values are left in place, existing wrong values are corrected, missing fields are inserted after a configured anchor sibling. The invoice's **original** `Number` value is captured here (`originalNumber`) before anything can change it. In the same pass, `KeyAmount` (a pure duplicate of `Amount`, mirroring the `TaxAmount`→`KeyTaxAmount` pattern already present in every `LineItem`), `GeneralLedgerDate` (the single, run-wide Fibu-Datum from the UI — MOCO has no per-invoice service-date field), and `PaymentTerm` (a fixed `Number`/`CopyFromTable`/`Type` triple, hardcoded to ÜbergabeNr. 5 = "30 Tage netto" in `CONFIG.paymentTerm`; created fresh as the invoice's last child if missing, otherwise only those three sub-fields are corrected — `CONFIG.paymentTerm.removeFields` (currently `DeadlineAsDate`) are actively stripped if present, any other unlisted sub-field is left alone) are ensured the same way. All duplicate-structure checks for these fields run upfront in `validateInvoiceStructureOrThrow`, before any of them mutate the invoice, so a structurally broken invoice is left fully untouched rather than partially transformed.
3. Find all dossier entries; rename `DocumentName` → `Number` inside `Dossier` (`CONFIG.rename`). Entries that fail (duplicate/missing field) are excluded from the next step.
4. Transfer the MOCO document number from each successfully-renamed dossier entry into the matching invoice's `Document > Number`, joined via the *pre-transform* invoice `Number` against the outer `DocumentDossier > Number` (the "Beleg-Nummer"). A dossier entry with no matching invoice is not an error (dossier data is optional); two dossier entries claiming the same Beleg-Nummer is an error and fails only that invoice. Immediately after (same try/catch, same invoice), `PaymentReferenceLine` (the Swiss QR-bill reference) is recomputed from `Document > Reference` — see `buildPaymentReferenceLine`/`CONFIG.paymentReferenceLine`: a fixed 5-digit creditor prefix + `Reference` zero-padded to 26 digits + a recursive Modulo-10 check digit (identical algorithm to the classic Swiss ESR slip). Deliberately `Reference`, not `Number`: `Number` can get overwritten by the Beleg-Nummer transfer above, but `Reference` stays the stable ABACUS-native document number throughout. Only applies when `PaymentReferenceLineType` is `"QR"`; otherwise `"notApplicable"`, not an error.
4b. If `CONFIG.removeDossierTaskFromOutput` is true (the default), `removeDossierTasks` strips every `Task` that contains a `DocumentDossier` from the DOM entirely — PDF invoices are now delivered to ABACUS through a separate channel, so the interface no longer carries `Dossier`/filename references — and `updateTaskCount` recomputes `<TaskCount>` to match. This runs **after** step 4's number transfer and after the XLSX lookups in step 6 are built from the still-intact DOM, so the Abrechnungsliste keeps showing dossier/filename info even though the XML no longer does. The transfer logic itself (steps 3-4) is deliberately left in place rather than removed, in case the "no Dossier task" requirement is reversed later — flip the CONFIG flag back to `false`.
5. Find all `Customer` entries (independent of invoices/dossier entries — customers live in their own `Task`, e.g. `Id=Kunden`); for each, resolve its intercompany code via `resolveIntercompanyCode` — MOCO sometimes delivers compound names (`"Multicolor Print | Galledia Print AG"`), so the `AddressData > Name` is split on `CONFIG.nameSegmentSeparator` ("|") and each trimmed segment is checked against `CONFIG.intercompanyByCustomerName`, not just the name as a whole. On a match, ensure an `Intercompany` field with the mapped value exists (same `ensureField` mechanics as step 2, inserted after `DefaultCurrency`). No segment matching is not an error — most customers aren't intercompany customers.
6. Build supplementary lookups (customer master data by `CustomerNumber`, dossier/file info by the pre-transform Beleg-Nummer — **not** the now-overwritten `Number`) and merge them into a flattened per-invoice row. This only feeds the `.xlsx` export; it never touches the XML output.
7. Serialize back to XML, re-attaching the original `<?xml ...?>` declaration (after step 4b has already removed the Dossier task(s), if configured to).

Failures are per-item, not fatal: a structurally broken invoice or dossier entry is skipped with a recorded error message, and the rest of the batch still processes. `result` aggregates counts (`processedCount`, `errorCount`, `dossierRenamedCount`, `documentNumberStats`, …) plus per-error label/message lists, which the UI renders as-is in the result overview.

### XLSX export

`buildAbrechnungslisteXlsx` hand-rolls a minimal OOXML package (Content_Types, workbook, one worksheet) and zips it with a from-scratch, uncompressed ("stored" method) ZIP writer (`crc32`, `buildZip`). There is no external library involved — don't assume one is available or add one; the whole point was zero dependencies.

### Deployment / embedding constraint

The tool is embedded in Confluence Cloud via its native "Iframe" macro. That macro sandboxes the iframe without `allow-downloads`, which silently breaks the JS-triggered file download (`Blob` + `<a download>` click) inside Confluence — this is a Confluence-imposed browser restriction, not something fixable from inside the page alone, and the native macro exposes no sandbox/permission settings. No workaround is implemented yet; if asked to fix Confluence downloads, the viable path is an in-page "open in a real tab" escape hatch, not a sandbox-attribute change.

## Repository layout notes

- `example/` contains real-world-shaped MOCO XML samples used for manual verification against the actual `Task`/`Transaction`/`Document`/`DocumentDossier` structure — useful for understanding the schema, not part of the shipped tool.
