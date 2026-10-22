Tutorial: getting started
============================

In this tutorial you open a broken semantic model, watch it fail in
Power BI Desktop, then use ``tmdl-preflight`` to find and repair every
blocker until it opens clean. It takes about ten minutes and assumes a
working Python 3.10 or later installation. Opening the model yourself,
steps 1 and 6, needs Power BI Desktop, which needs Windows; every
other step runs on whatever platform runs Python, since the CLI reads
and writes plain TMDL text files directly.

1. Open the broken model, and watch it fail
-------------------------------------------------

The repository ships two copies of a small fictional PBIP project, the
Pedal and Sprocket Bike Co. model, under ``examples/``:
``bike-shop-clean`` (the model as it should be) and
``bike-shop-broken`` (the same model, seeded with defects). Open
``examples/bike-shop-broken/BikeShop.pbip`` in Power BI Desktop.

It does not open. The seeded defect that stops it is a duplicate
``lineageTag``: ``Stores`` carries the tag ``Products`` already has, and
Desktop refuses the project outright ("Failed to add a deserialized
Table object into the model ... An object with lineage-tag '...' already
exists in the collection"). That is the kind of thing that sails past a
code review, since nothing in a diff looks wrong, but stops Power BI
Desktop cold. The error Desktop shows is generic, and in some cases
there is no error at all: the file just hangs on load and the report
never opens. Either way, nothing names the file, the table, or the
fix. That is the gap ``tmdl-preflight`` closes.

The other seeded defects behave differently, and the tool's own release
notes record what Desktop did with each of them on a Windows bench: a
missing ``ref table`` line (M006) still loads, and a field-parameter
orphan comma (F001) opens the model but breaks every visual that uses
the field parameter. Only the duplicate tag stops the project opening.

2. Install
-------------

Clone the repository, then install the package in editable mode from
inside it (the ``[test]`` extra pulls in pytest, which the how-to
guides also use):

::

    $ git clone https://github.com/c-kapsalis/tmdl-preflight.git
    $ cd tmdl-preflight
    $ pip install -e ".[test]"

Confirm the console command is on the path:

::

    $ tmdl-preflight --version

::

    tmdl-preflight 1.2.0

3. Check the clean model
----------------------------

``check`` is strictly read-only; point it at a project root, a
``*.SemanticModel`` folder, a ``definition`` folder, or a
``*.Report`` folder:

::

    $ tmdl-preflight check examples/bike-shop-clean

The model and the report both pass every rule. The run reports two info
findings:

::

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:26  M017 info: Line 26: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    (the run then prints its console report: the DAX validation summary
    and the R001, R002, R004, R005, R009, R010, R012, R013 and R014 blocks)
    No .bookmark.json files found under examples/bike-shop-clean
    tmdl-preflight: 0 error(s), 0 warning(s), 2 info(s)

The ``scanned:`` line says how much of the project the run looked at,
so a run that silently found nothing to check is obvious. The run exits
``0``.

The two findings are info, and they are the two the example is meant to
carry: M017, a measure whose multi-line DAX is not fenced, and R013, a
calendar without the usual year/quarter/month levels. Neither stops
Desktop opening the model.

4. Check the broken model
-----------------------------

::

    $ tmdl-preflight check examples/bike-shop-broken

Seven findings come back — the four seeded defects and three notes:

::

    BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl:45  F001 error: 1 orphan comma(s). Power BI Desktop still opens the model, but every visual that uses this table fails ('Error fetching data for this visual'). Delete the orphan comma(s) (the auto-fix does). [Metric Selector (field-parameter file)] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/Stores.tmdl  M006 warning: table 'Stores' is defined in tables/ but has no 'ref table' line in model.tmdl. Power BI Desktop still loads the table, but model.tmdl is out of step with tables/. Add: ref table Stores [Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Stores.tmdl:3  M003 error: Duplicate lineageTag 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' found in files: Products.tmdl, Stores.tmdl. The tag on line 3 (table Products) is the one that keeps it; the copy on line 3 (table Stores) is regenerated by the auto-fix. [table Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl:4  M004 warning: LineageTag 'calendar-tag-TODO' is not a canonically-formatted UUID (expected 8-4-4-4-12 hex). Power BI rejects a tag it cannot read; the auto-fix writes a fresh one. [table __Calendar] (auto-fixable)

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    (the run then prints its console report: the DAX validation summary
    and the R001, R002, R004, R005, R009, R010, R012, R013 and R014 blocks)
    No .bookmark.json files found under examples/bike-shop-broken
    tmdl-preflight: 2 error(s), 2 warning(s), 3 info(s)

Exit code 1. Every line names the file, the line where one exists, the
rule id, the severity, and what is wrong; where the tool can repair it,
an ``(auto-fixable)`` marker:

- **M003**: ``Stores`` copied ``Products``' ``lineageTag``. This is the
  exact defect that stopped Desktop from opening the project, and the
  one that survives review because nothing in the diff looks wrong.
- **F001**: a deleted field-parameter row left an orphan comma behind,
  so the calculated partition source no longer parses. Desktop still
  opens the model, but every visual that uses the field parameter fails.
- **M006**: ``Stores`` is defined under ``tables/`` but has no ``ref
  table`` line, so ``model.tmdl`` is out of step with ``tables/``. This
  is a **warning**, not a blocker: Desktop still loads the table, and
  the fixer adds the missing line anyway.
- **M004**: ``__Calendar``'s ``lineageTag`` is a ``TODO`` placeholder,
  not a UUID.
- **S001**: the visible measure ``Units Sold`` has no ``formatString``.
  An advisory nudge, not a blocker; it does not stop the model from
  opening.
- **M017** and **R013**: two info findings about a multi-line DAX body
  and a calendar without the usual levels.

Four findings carry the ``(auto-fixable)`` marker: F001, M006, M003 and
M004. The other three do not. That split is deliberate: a defect earns
an auto-fix only when the failure itself fully determines the repair.

5. Fix a copy
----------------

``fix`` edits files in place, so work on a copy and keep the broken
original for rereading this tutorial:

::

    $ cp -r examples/bike-shop-broken /tmp/bike-shop
    $ tmdl-preflight fix /tmp/bike-shop

Four fixers run, then the re-check reports what remains:

::

    fixed  M006: model.tmdl: + ref table Stores
    fixed  M003: Stores.tmdl:3 (table Stores): 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' -> '91232640-8fc3-4547-9f7a-b32beb0b4d0f'
    fixed  M004: __Calendar.tmdl:4 (table __Calendar): 'calendar-tag-TODO' -> '4ddb8fa0-44ca-444c-9d59-b1744dad3fd2'
    fixed  F001: Metric Selector.tmdl: removed 1 orphan comma(s)
    not fixed  R007: its fixer can change data, so a blanket fix skips it; run 'fix --select R007' to apply it

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    (the run then prints its console report: the DAX validation summary
    and the R001, R002, R004, R005, R009, R010, R012, R013 and R014 blocks)
    No .bookmark.json files found under /tmp/bike-shop
    tmdl-preflight: 0 error(s), 0 warning(s), 3 info(s), 4 fix(es) applied

    Removed 1 orphan comma(s) across 1 file(s). Commit the changes and re-sync the affected workspace(s).

(The regenerated UUIDs are random; yours will differ.) Three things
happened, in order: every rule ran (``check``); the fixers of the
rules that failed ran, editing files on disk; then every rule ran
**again**, from disk, and the report above is that re-check. The exit
code is ``0``: the re-check finds no errors and no warnings, and
``fix`` reports the state of the run rather than hiding it. The
``not fixed  R007:`` line is the other half of that honesty: R007's
fixer deletes relationship blocks, so it is unsafe and never runs in a
blanket fix. Diff the copy against ``examples/bike-shop-broken``
before moving on: every auto-fix is a one-line, or one-value, change.

6. Open the repaired copy
-----------------------------

Open ``/tmp/bike-shop/BikeShop.pbip`` in Power BI Desktop. Nothing was
guessed on your behalf; every change above is one you can read in a
two-line diff: ``Stores`` is attached to the model again, the calendar
and duplicate lineage tags are resolved, and the ``Metric Selector``
field parameter loads without a parse error. Opening the ``.pbip`` in
Power BI Desktop is a manual step that has not been done by the tool:
what the tool proves is that the four defects it repaired are gone
from the files.

7. Read what remains
------------------------

::

    $ tmdl-preflight check /tmp/bike-shop

Three findings remain, and all three are info:

::

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    (the run then prints its console report: the DAX validation summary
    and the R001, R002, R004, R005, R009, R010, R012, R013 and R014 blocks)
    No .bookmark.json files found under /tmp/bike-shop
    tmdl-preflight: 0 error(s), 0 warning(s), 3 info(s)

The S001 finding stays on purpose: it is an ``info``-level nudge, not
a defect that stops the model from opening, and the choice of how to
format ``Units Sold`` is yours. Give it a ``formatString`` if you
format measures on the model itself, or run with ``--ignore S001`` if
your formatting strategy lives elsewhere (a calculation group, a
theme).

8. Explore further
----------------------

The full rule catalog, one rule in isolation, machine-readable output,
and a stricter exit code:

::

    $ tmdl-preflight rules
    $ tmdl-preflight check examples/bike-shop-broken --select M003
    $ tmdl-preflight check examples/bike-shop-broken --json
    $ tmdl-preflight check examples/bike-shop-broken --strict
    $ tmdl-preflight check examples/bike-shop-broken --format markdown
    $ tmdl-preflight stats examples/bike-shop-clean

Where to go next
--------------------

- Add a rule of your own: :doc:`../how-to/modify-the-rules-list`
- Add or change an auto-fix: :doc:`../how-to/modify-the-auto-fixes`
- Every CLI flag, exit code, and command:
  :doc:`../reference/usage`
- The rule catalog, family by family:
  :doc:`../reference/rules`
- Configuration files, exemptions, baselines and output formats:
  :doc:`../reference/configuration`
- Run it in CI and pre-commit: :doc:`../how-to/run-in-ci`
- The pytest fixture, the Python API, and the design behind the
  check → fix → re-check loop: :doc:`../reference/customization`
