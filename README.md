# CMPivot-Queries
Collection of cmpivot queries for endpoint engineering, audits etc.

## Scope

This repository contains CMPivot queries intended for:
- Endpoint inventory and validation
- Configuration visibility
- Remediation scoping and verification

All queries are detection-only and do not modify endpoint state.

## Notes on CMPivot

CMPivot provides a reduced query surface compared to full KQL or SCCM inventory.
Some attributes (e.g., MSI vs Click-to-Run distinctions) are not reliably exposed.
Where applicable, queries are documented with these limitations in mind.


## Inspiration

Some of the initial structure and ideas for this repository were inspired by:
https://github.com/svschmit/CMPivot-Queries

If you're looking for a broader set of general CMPivot examples, that repo is a great resource.
