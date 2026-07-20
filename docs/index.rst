tmdl-preflight
==============

Preflight checks and auto-fixes for Power BI semantic models saved in
TMDL format (PBIP folders). For most of Power BI's life the semantic
model was a binary ``.pbix`` blob: un-diffable, un-reviewable,
un-testable. PBIP and TMDL changed that; the model is now a folder of
plain-text files, which means it can finally be tested like code. But
nothing ships that testing layer on its own: a duplicate
``lineageTag`` left behind by a copy-pasted table file sails through
Power BI Desktop and every code review, then kills the deployment days
later with "an object with that lineage tag already exists."
``tmdl-preflight`` catches that class of defect on your machine, and
repairs the mechanical ones itself.

Twenty-one rules cover the failure modes that reject imports or
silently corrupt results, five of them auto-fixable, reachable from a
CLI, a pytest fixture, or a plain Python API.

.. toctree::
   :maxdepth: 1

   tutorials/getting-started
   how-to/run-in-ci
   how-to/run-in-pytest
   how-to/run-only-autofixable-rules
   how-to/select-and-ignore-rules
   how-to/consume-json-output
   how-to/add-a-custom-rule
   reference/cli
   reference/rules
   reference/python-api
   explanation/imposition-pattern
   explanation/layered-testing
   explanation/will-it-open
   explanation/tmdl-well-formedness
   explanation/duplicate-lineage-tags
   explanation/column-data-types
   explanation/dax-delimiter-balance
   explanation/nameof-reference-integrity
   explanation/duplicate-measure-names
   explanation/relationship-integrity
   explanation/bidirectional-and-orphan-tables
   explanation/field-parameter-comma-runs
   explanation/bookmark-integer-types
   explanation/bookmark-visual-references
   explanation/format-strings
