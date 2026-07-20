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
``bike-shop-clean`` (every rule passes) and ``bike-shop-broken`` (the
same model, seeded with defects). Open
``examples/bike-shop-broken/BikeShop.pbip`` in Power BI Desktop.

It does not open. One of the seeded defects, a table missing its ``ref
table`` line in ``model.tmdl``, is exactly the kind of thing that sails
past a code review, since nothing in a diff looks wrong, but stops
Power BI Desktop cold. The error Desktop shows is generic, and in some
cases there is no error at all: the file just hangs on load and the
report never opens. Either way, nothing names the file, the table, or
the fix. That is the gap ``tmdl-preflight`` closes.

2. Install
-------------

From a clone of this repository (the ``[test]`` extra pulls in pytest,
which the how-to guides also use), install the package in editable
mode:

::

    $ pip install -e ".[test]"

Confirm the console command is on the path:

::

    $ tmdl-preflight --version

::

    tmdl-preflight 0.1.0

3. Check the clean model
----------------------------

``check`` is strictly read-only; point it at a project root, a
``*.SemanticModel`` folder, a ``definition`` folder, or a
``*.Report`` folder:

::

    $ tmdl-preflight check examples/bike-shop-clean

Every rule passes:

::

    tmdl-preflight: 0 error(s), 0 warning(s), 0 info(s)

Exit code 0: every rule passes, and the model opens in Power BI
Desktop without incident.

4. Check the broken model
-----------------------------

::

    $ tmdl-preflight check examples/bike-shop-broken

Five findings come back:

::

    BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl:45  F001 error: stray comma run (1 orphan comma(s)) in calculated partition source [Metric Selector (partition source)] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:17  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]
    BikeShop.SemanticModel/definition/tables/Stores.tmdl  M006 error: table 'Stores' is defined in tables/ but has no 'ref table' line in model.tmdl, so it is not part of the model and Power BI will not open the project. Add: ref table Stores [Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/Stores.tmdl:3  M003 error: lineageTag 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' duplicates the one on table Products (Products.tmdl:3) [table Stores] (auto-fixable)
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl:4  M004 warning: lineageTag 'calendar-tag-TODO' is not a canonical UUID [table __Calendar] (auto-fixable)

    tmdl-preflight: 3 error(s), 1 warning(s), 1 info(s)

Exit code 1. Every line names the file, the line where one exists, the
rule id, the severity, and what is wrong; where the tool can repair it,
an ``(auto-fixable)`` marker:

- **M006**, the exact defect that stopped Desktop from opening the
  project: ``Stores`` is defined under ``tables/`` but has no ``ref
  table`` line, so it is not attached to the model.
- **M003**: ``Stores`` copied ``Products``' ``lineageTag``. A duplicate
  like this is often the one that survives review and Desktop, then
  fails a deployment days later; here it also compounds the open
  failure.
- **M004**: ``__Calendar``'s ``lineageTag`` is a ``TODO`` placeholder,
  not a UUID.
- **F001**: a deleted field-parameter row left an orphan comma behind,
  so the calculated partition source no longer parses.
- **S001**: the visible measure ``Units Sold`` has no ``formatString``.
  An advisory nudge, not a blocker; it does not stop the model from
  opening.

Four findings carry the ``(auto-fixable)`` marker; S001 does not. That
split is deliberate: a defect earns an auto-fix only when the failure
itself fully determines the repair.

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
    fixed  M003: Stores.tmdl:3 (table Stores): 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' -> '14946e6f-ad2f-4520-97f8-ade96bc6adeb'
    fixed  M004: __Calendar.tmdl:4 (table __Calendar): 'calendar-tag-TODO' -> 'a3461075-352b-454c-9469-c584419c2b3e'
    fixed  F001: Metric Selector.tmdl: removed 1 orphan comma(s) from 'Metric Selector' partition source

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:17  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]

    tmdl-preflight: 0 error(s), 0 warning(s), 1 info(s), 4 fix(es) applied

(The regenerated UUIDs are random; yours will differ.) Three things
happened, in order: every rule ran (``check``); the fixers of the
rules that failed ran, editing files on disk; then every rule ran
**again**, from disk, and the report above is that re-check. Exit code
0 because the re-check came back clean of blockers, not because fixers
ran. Diff the copy against ``examples/bike-shop-broken`` before moving
on: every auto-fix is a one-line, or one-value, change.

6. Open the repaired copy
-----------------------------

Open ``/tmp/bike-shop/BikeShop.pbip`` in Power BI Desktop. It opens
clean: ``Stores`` is present, the calendar and duplicate lineage tags
are resolved, and the ``Metric Selector`` field parameter loads without
a parse error. Nothing was guessed on your behalf; every change above
is one you can read in a two-line diff.

7. Read what remains
------------------------

::

    $ tmdl-preflight check /tmp/bike-shop

One finding remains:

::

    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:17  S001 info: visible measure has no formatString, so it renders with the engine's default formatting. Add a formatString, or run with --ignore S001 if your formatting strategy lives elsewhere. [Sales Measures[Units Sold]]

    tmdl-preflight: 0 error(s), 0 warning(s), 1 info(s)

Exit code 0. S001 stays on purpose: it is an ``info``-level nudge, not
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
    $ tmdl-preflight check examples/bike-shop-broken --select M006
    $ tmdl-preflight check examples/bike-shop-broken --json
    $ tmdl-preflight check examples/bike-shop-broken --strict

Where to go next
--------------------

- Add a rule of your own: :doc:`../how-to/modify-the-rules-list`
- Add or change an auto-fix: :doc:`../how-to/modify-the-auto-fixes`
- Every CLI flag, exit code, and output format:
  :doc:`../reference/usage`
- The pytest fixture, the Python API, and the design behind the
  check → fix → re-check loop: :doc:`../reference/customization`
