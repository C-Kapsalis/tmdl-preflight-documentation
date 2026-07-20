Rule catalog
============

.. AUDIT GAP, fix before porting: tmdl-preflight-docs/reference/rules.md
   documents only 17 of the 21 rules the code actually implements. Add the
   four missing entries below before this page is considered ported, not
   just copied. Authoritative source: ``src/tmdl_preflight/rules/__init__.py``
   ``ALL_RULES``, cross-checked against ``rules/structural.py``.

.. Severity legend: error (breaks an import/deploy, or silently corrupts
   results), warning (suspicious, deserves a look), info (advisory, never
   blocks, not even with --strict).

Structural rules (M0xx)
------------------------

.. Existing, already drafted: M001 model-structure, M002 tmdl-well-formed,
   M003 lineage-duplicates (auto-fixable), M004 lineage-format
   (auto-fixable), M005 column-data-types.

.. MISSING, add these four:
   M006 model-table-refs (error, auto-fixable), a table with no matching
   ``ref table`` line in model.tmdl; Power BI Desktop never opens the
   project. Pairs with the new will-it-open explanation page.
   M007 table-partitions (error, not auto-fixable), a table with no
   partition at all; Desktop crashes on open ("Sequence contains no
   elements").
   M008 entity-query-source (error, not auto-fixable), an inline
   ``#table(type table ...)`` M source forces a composite model; Desktop
   refuses to open it.
   M009 reserved-table-name (error, not auto-fixable), a table literally
   named "Measures" collides with the implicit measures container.

DAX rules (D0xx)
-----------------

.. Existing, already drafted: D001 dax-delimiters, D002 nameof-resolution,
   D003 duplicate-measure-names.

Relationship rules (R0xx)
---------------------------

.. Existing, already drafted: R001 through R005.

Field-parameter rules (F0xx)
------------------------------

.. Existing, already drafted: F001 fieldparam-comma-runs (auto-fixable).

Report rules (B0xx)
---------------------

.. Existing, already drafted: B001 bookmark-int-types (auto-fixable), B002
   bookmark-visual-refs.

Style rules (S0xx)
--------------------

.. Existing, already drafted: S001 format-strings (advisory only).

Before publishing this page
-----------------------------

.. Run every message in this table against the error-message checklist in
   ``_kit/error-message-checklist.md``, several already meet it (M007,
   R002), which is a good model for the four you are adding.
