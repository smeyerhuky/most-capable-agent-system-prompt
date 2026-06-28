# GitHub Copilot Code Review Instructions: OKF Standards

## 🚨 ABSOLUTE RULE 🚨
**Every single issue, critique, or comment you make MUST include a specific, actionable proposed fix or code suggestion.** Do not merely point out an error; you must provide the exact corrected JSON, schema, or code snippet to resolve it.

## 1. OKF (Frictionless Data) Validation Rules
When reviewing Pull Requests containing OKF formatted documents (like `datapackage.json`, schema definitions, or tabular data files), strictly validate the following:
* **Data Package JSON Structure:** Ensure compliance with the [Frictionless Data Package specification](https://specs.frictionlessdata.io/data-package/). Check for required fields (`name`, `resources`).
* **Table Schema Compliance:** Validate field definitions, data types (string, integer, date, etc.), and constraints (required, unique, minimum, maximum).
* **Resource Validation:** Verify that resource `path`, `format`, `encoding`, and `mediatype` are correctly specified and logically sound.
* **Foreign Key Relationships:** Ensure foreign keys correctly map local fields to reference fields in other resources.
* **CSV Dialect Specifications:** Validate any specified `dialect` properties (delimiter, doubleQuote, lineTerminator) against the standard.

## 2. GitHub Cross-Reference Validation
* **Validate URLs & Paths:** Cross-reference paths and URLs in OKF documents to ensure they point to valid locations. Validate against the actual repository structure where possible.
* **Metadata References:** Check that any referenced repositories, contributors, or external documentation links are accessible, accurate, and formatted correctly.
* **Resource Links:** Verify that data resource links point to valid data files within the repository or valid external URIs.

## 3. Best Practices
* **Metadata Completeness:** Require rich metadata (e.g., ensuring `description`, `licenses`, `sources`, and `contributors` are fully populated).
* **Schema Design Patterns:** Ensure schemas are modular, reusable, and intuitively named.
* **Documentation Standards:** Advise on inline descriptions or accompanying `README.md` updates when schemas or datasets are modified.
