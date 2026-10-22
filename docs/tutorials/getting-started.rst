Tutorial: getting started
============================

In this tutorial you install ``tmdl-preflight`` 1.2.1, check two copies
of a small example model, repair the broken copy with ``fix``, and read
what the tool leaves for you to decide. It takes about ten minutes.

You need Git and Python 3.10 or later. Every step runs on Windows,
macOS and Linux, because the tool reads and writes plain TMDL and JSON
files. Where a command differs between shells, the page shows it for
PowerShell and for bash. Power BI Desktop is optional: the steps that
mention it are for readers on Windows who want to compare the files with
what Desktop does.

.. note::

   The output on this page was captured with bash on macOS. On Windows
   the text report prints file paths with backslashes. The PowerShell
   commands were not run, and no Windows output was captured, when this
   page was written.

1. Install
------------

Clone the repository at the ``v1.2.1`` tag, create a virtual
environment, and install the package from the clone. The ``[test]``
extra adds pytest, which the how-to guides use, and PyYAML, which lets
``manifest`` and ``refs`` read a targets file written in YAML.

PowerShell:

.. code-block:: powershell

    git clone --branch v1.2.1 https://github.com/C-Kapsalis/tmdl-preflight.git
    cd tmdl-preflight
    python -m venv .venv
    .venv\Scripts\Activate.ps1
    pip install -e ".[test]"

bash:

.. code-block:: bash

    git clone --branch v1.2.1 https://github.com/C-Kapsalis/tmdl-preflight.git
    cd tmdl-preflight
    python3 -m venv .venv
    source .venv/bin/activate
    pip install -e ".[test]"

Git reports a "detached HEAD" after cloning a tag; that is expected. On
Windows, if ``python`` is not on the path, the Python launcher ``py``
runs the same command (``py -m venv .venv``). If PowerShell refuses to
run the activation script, see the :mod:`venv` documentation for the
execution-policy setting it needs.
Every command after this one runs from the ``tmdl-preflight`` folder,
with the virtual environment active.

Confirm the console command is on the path:

.. code-block:: text

    tmdl-preflight --version

.. code-block:: text

    tmdl-preflight 1.2.1

2. Meet the example models
------------------------------

The repository ships two copies of a small fictional PBIP project, the
Pedal and Sprocket Bike Co. model, under ``examples/``:
``bike-shop-clean`` (the model as it should be) and
``bike-shop-broken`` (the same model, seeded with defects).

The tool's release notes record what Power BI Desktop (Windows,
installer build) did with copies of this model that carried one defect
each:

- a duplicate ``lineageTag``: Desktop refused to open the model ("Failed
  to add a deserialized Table object into the model ... An object with
  lineage-tag '...' already exists in the collection.");
- a table file with no ``ref table`` line: Desktop opened the model and
  loaded the table;
- an orphan comma in a field parameter: Desktop opened the model, but
  every visual that uses the field parameter failed ("Error fetching
  data for this visual.").

If you have Power BI Desktop, you can open
``examples/bike-shop-broken/BikeShop.pbip`` and compare. A duplicate tag
is easy to miss in review, because nothing in the diff looks wrong. The
tool names the file, the line, the rule and the repair.

3. Check the clean model
----------------------------

``check`` is read-only. Point it at a project root, a
``*.SemanticModel`` folder, a ``definition`` folder or a ``*.Report``
folder:

.. code-block:: text

    tmdl-preflight check examples/bike-shop-clean

This is the whole output:

.. code-block:: text

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:26  M017 info: Line 26: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    DAX Validation Summary:
       Total expressions: 6
       Valid: 6 (100.0%)
       Invalid: 0 (0.0%)
    R001: BikeShop.SemanticModel
    Validated 4 relationships
       Errors: 0
       Warnings (inactive): 1
       Success rate: 100.0%
    Inactive: Sales -> __Calendar
    R002: BikeShop.SemanticModel
    Cardinality Distribution:
       many_to_one            4
    R004: BikeShop.SemanticModel
    Cross-filtering Analysis:
       Unidirectional: 4
       Bidirectional: 0
    Bidirectional count is acceptable (<=3)
    R005: BikeShop.SemanticModel
    Total tables: 6
    With relationships: 4
    Orphans: 2

    Orphan Categories:
       - Plot Specific tables: 0
       - Selector tables: 1
       - Measure tables: 1
       - Other: 0

    Expected orphans (Plot Specific/Selector/Measure): 2
    Unexpected orphans: 0
    R009: BikeShop.SemanticModel
    Found 4 relationships (3 active, 1 inactive)
    R010: BikeShop.SemanticModel
    Circular Dependency Analysis:
       Active relationships checked: 3
       Tables in graph: 1
       Cycles detected: 0
    R012: BikeShop.SemanticModel
    Calendar Relationship Cardinality:
    Total calendar relationships: 2
    Valid (many-to-one): 2
    Invalid cardinality: 0
    R013: BikeShop.SemanticModel
    Calendar Table Completeness:
    Total columns: 5
    Standard columns present: 1/6

    Present standard columns:
      - Date

    Advisory - Missing standard columns:
      - Year
      - Quarter
      - Month
      - Week
      - Day

    These columns are recommended for complete date hierarchies.

    Additional columns (4):
      - Calendar Month
      - Calendar Month Name
      - Calendar Quarter
      - Calendar Year
    R014: BikeShop.SemanticModel
    Total dimension tables (_prefix): 1
    With key columns: 1 (100.0%)
    Without apparent keys: 0

    All dimension tables have key columns matching the canonical patterns.
    No .bookmark.json files found under examples/bike-shop-clean
    tmdl-preflight: 0 error(s), 0 warning(s), 2 info(s)

The report has five parts, in this order:

- the findings, one per line: file, line where there is one, rule id,
  severity and message, then the object in brackets;
- the ``scanned:`` line, which says how much of the project the run
  looked at, so a run that silently found nothing to check is obvious;
- the console report: blocks some rules print for themselves (the DAX
  validation summary, then the R001 to R014 blocks);
- notes such as the one about bookmark files;
- the summary line.

The two findings are info, and they are the two the example is meant
to carry: M017, a measure whose multi-line DAX is not fenced with
triple backticks, and R013, a calendar without the usual
year, quarter and month columns. Info findings never fail a run, so the
command exits ``0``. To see the exit code, run ``$LASTEXITCODE`` in
PowerShell or ``echo $?`` in bash right after the command.

4. Check the broken model
-----------------------------

.. code-block:: text

    tmdl-preflight check examples/bike-shop-broken

Seven findings come back: the four seeded defects and three notes. The
console report is the same as in step 3 and is cut here (the ``...``
line):

.. code-block:: text

    BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl:45  F001 error: 1 orphan comma(s). Power BI Desktop still opens the model, but every visual that uses this table fails ('Error fetching data for this visual'). Delete the orphan comma(s) (the auto-fix does). [Metric Selector (field-parameter file)] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/Stores.tmdl  M006 warning: table 'Stores' is defined in tables/ but has no 'ref table' line in model.tmdl. Power BI Desktop still loads the table, but model.tmdl is out of step with tables/. Add: ref table Stores [Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Stores.tmdl:3  M003 error: Duplicate lineageTag 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' found in files: Products.tmdl, Stores.tmdl. The tag on line 3 (table Products) is the one that keeps it; the copy on line 3 (table Stores) is regenerated by the auto-fix. [table Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl:4  M004 warning: LineageTag 'calendar-tag-TODO' is not a canonically-formatted UUID (expected 8-4-4-4-12 hex). Power BI rejects a tag it cannot read; the auto-fix writes a fresh one. [table __Calendar] (auto-fixable)

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    ...
    No .bookmark.json files found under examples/bike-shop-broken
    tmdl-preflight: 2 error(s), 2 warning(s), 3 info(s)

The exit code is ``1``, because there are errors. Where the tool can
repair a finding, the line ends with ``(auto-fixable)``:

- **M003** (error): ``Stores`` carries the ``lineageTag`` that
  ``Products`` already has. This is the defect that stopped Desktop from
  opening the model on the bench.
- **F001** (error): a deleted field-parameter row left an orphan comma
  behind, so the calculated partition source no longer parses.
- **M006** (warning): ``Stores`` is defined under ``tables/`` but has no
  ``ref table`` line, so ``model.tmdl`` is out of step with ``tables/``.
  Desktop still loaded the table on the bench; the fixer adds the line
  anyway.
- **M004** (warning): the ``lineageTag`` of ``__Calendar`` is the
  placeholder ``calendar-tag-TODO``, not a UUID. The release notes
  record that Desktop opened a copy whose only defect was this
  placeholder; the fixer replaces it with a fresh UUID.
- **S001** (info): the visible measure ``Units Sold`` has no
  ``formatString``.
- **M017** and **R013** (info): the same two notes as in the clean
  model.

Four findings are auto-fixable: F001, M006, M003 and M004. The other
three are not. A defect gets a fixer only when the failure itself
determines the repair; how ``Units Sold`` should be formatted is your
decision.

5. Fix a copy
----------------

``fix`` edits files in place. Copy the broken model first, so the
original stays as it is (if ``../bike-shop`` is left over from an
earlier run, delete it first):

PowerShell:

.. code-block:: powershell

    Copy-Item -Recurse examples/bike-shop-broken ../bike-shop

bash:

.. code-block:: bash

    cp -r examples/bike-shop-broken ../bike-shop

Then run the fixers on the copy:

.. code-block:: text

    tmdl-preflight fix ../bike-shop

Four fixers run, then the re-check reports what remains (the console
report is cut again):

.. code-block:: text

    fixed  M006: model.tmdl: + ref table Stores
    fixed  M003: Stores.tmdl:3 (table Stores): 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' -> 'cad85256-6367-4144-9e87-10018d08f1ce'
    fixed  M004: __Calendar.tmdl:4 (table __Calendar): 'calendar-tag-TODO' -> '06374f8f-6015-4152-a527-e555a357cc6f'
    fixed  F001: Metric Selector.tmdl: removed 1 orphan comma(s)

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    ...
    No .bookmark.json files found under ../bike-shop
    tmdl-preflight: 0 error(s), 0 warning(s), 3 info(s), 4 fix(es) applied

    Removed 1 orphan comma(s) across 1 file(s). Commit the changes and re-sync the affected workspace(s).

The regenerated UUIDs are random, so yours will differ. Three things
happened, in order: every rule ran (the check); the fixers of the rules
that reported findings ran and edited the files on disk; then every
rule ran again, from disk, and the report above is that re-check. The
exit code is ``0``: the re-check finds no errors and no warnings. The
last line is the closing advice the F001 fixer prints after the summary.

Four fixers are marked unsafe (D004, F016, P001 and R007) and never run
in a plain ``fix``. When the check reports a finding that one of them
repairs, the run says so on a ``not fixed  <RULE>:`` line and names the
command that applies it. This model has no such finding, so no such
line appears.

See exactly what changed. Git compares two folders outside a
repository with ``--no-index``; the command is the same in PowerShell
and bash:

.. code-block:: text

    git diff --no-index examples/bike-shop-broken ../bike-shop

.. code-block:: text

    diff --git a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/model.tmdl b/../bike-shop/BikeShop.SemanticModel/definition/model.tmdl
    index 836c6a7..57ae811 100644
    --- a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/model.tmdl
    +++ b/../bike-shop/BikeShop.SemanticModel/definition/model.tmdl
    @@ -19,3 +19,4 @@ ref table Sales
     ref table 'Sales Measures'
     ref table 'Metric Selector'

    +ref table Stores
    diff --git a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl b/../bike-shop/BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl
    index d92805a..2f7bbac 100644
    --- a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl
    +++ b/../bike-shop/BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl
    @@ -43,7 +43,7 @@ table 'Metric Selector'
     				{
     				    ( "Revenue", NAMEOF ( [Revenue] ), 0 ),
     				    ( "Orders #", NAMEOF ( [Orders #] ), 1 ),
    -				    ,
    +				    
     				    ( "Units Sold", NAMEOF ( [Units Sold] ), 3 )
     				}

    diff --git a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/Stores.tmdl b/../bike-shop/BikeShop.SemanticModel/definition/tables/Stores.tmdl
    index 24cfb68..9e9419e 100644
    --- a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/Stores.tmdl
    +++ b/../bike-shop/BikeShop.SemanticModel/definition/tables/Stores.tmdl
    @@ -1,6 +1,6 @@
     /// One row per retail store.
     table Stores
    -	lineageTag: e689596a-59ea-4b2c-a14d-48596a7b8c9d
    +	lineageTag: cad85256-6367-4144-9e87-10018d08f1ce

     	column store_id
     		dataType: int64
    diff --git a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/__Calendar.tmdl b/../bike-shop/BikeShop.SemanticModel/definition/tables/__Calendar.tmdl
    index 87ed11b..1c2dba7 100644
    --- a/examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/__Calendar.tmdl
    +++ b/../bike-shop/BikeShop.SemanticModel/definition/tables/__Calendar.tmdl
    @@ -1,7 +1,7 @@
     /// Central date dimension. Self-contained calculated table so it always
     /// populates without an external source.
     table __Calendar
    -	lineageTag: calendar-tag-TODO
    +	lineageTag: 06374f8f-6015-4152-a527-e555a357cc6f
     	dataCategory: Time

     	column Date

Every change is one line: the new ``ref table`` line, the two new
lineage tags, and the orphan comma, deleted (the emptied line keeps its
indentation).

6. Open the repaired copy (optional)
----------------------------------------

If you have Power BI Desktop, open ``../bike-shop/BikeShop.pbip``.
Opening the ``.pbip`` in Desktop is a manual step that the tool does
not do, and this page does not report its result. What the tool proves
is that the four defects it repaired are gone from the files and that
its re-check passes.

7. Read what remains
------------------------

.. code-block:: text

    tmdl-preflight check ../bike-shop

Three findings remain, and all three are info (console report cut):

.. code-block:: text

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:21  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:25  M017 info: Line 25: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    ...
    No .bookmark.json files found under ../bike-shop
    tmdl-preflight: 0 error(s), 0 warning(s), 3 info(s)

The S001 finding stays on purpose: it is an ``info`` nudge, and how to
format ``Units Sold`` is your choice. Give the measure a
``formatString`` if you format measures on the model itself, or run
with ``--ignore S001`` if your formatting lives elsewhere (a
calculation group, a theme).

8. Explore further
----------------------

The full rule catalog, one rule on its own, machine-readable output, a
stricter exit code, a Markdown report and a project inventory:

.. code-block:: text

    tmdl-preflight rules
    tmdl-preflight check examples/bike-shop-broken --select M004
    tmdl-preflight check examples/bike-shop-broken --json
    tmdl-preflight check examples/bike-shop-broken --strict
    tmdl-preflight check examples/bike-shop-broken --format markdown
    tmdl-preflight stats examples/bike-shop-clean

With ``--strict`` a warning fails the run too. Of the four ``check``
commands, the three that run every rule exit ``1``. The one that
selects M004 alone exits ``0``, because M004 is a warning.

Where to go next
--------------------

- Every command, flag and exit code: :doc:`../reference/usage`
- The rule catalog: :doc:`../reference/rules`
- Configuration files, exemptions, baselines and output formats:
  :doc:`../reference/configuration`
- Run it in CI and pre-commit: :doc:`../how-to/run-in-ci`
- Add a rule of your own: :doc:`../how-to/modify-the-rules-list`
- Add or change an auto-fix: :doc:`../how-to/modify-the-auto-fixes`
- The pytest fixture, the Python API, and the design behind the
  check, fix, re-check loop: :doc:`../reference/customization`
