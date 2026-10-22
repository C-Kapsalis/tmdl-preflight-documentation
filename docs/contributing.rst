Contributing
================

``tmdl-preflight`` is a preflight linter and auto-fixer for Power BI
semantic models saved in TMDL format (PBIP folders): it catches the
defects that stop Power BI Desktop from opening a project, duplicate
lineage tags, tables with no partition, inline ``#table`` entity
sources, malformed lineage tags, field-parameter comma runs, and the
report files Fabric needs, and mechanically repairs the ones whose fix
is fully determined by the failure.

Contributions are welcome: new rules, fixers, docs, and bug reports.
This page covers how to set up, where things live, and how to add a
rule.

Development setup
---------------------

The tool has zero runtime dependencies (standard library only, Python
3.10 or later); the only dev dependency is pytest.

::

    $ git clone https://github.com/C-Kapsalis/tmdl-preflight.git
    $ cd tmdl-preflight
    $ python -m venv .venv
    $ source .venv/bin/activate
    $ pip install -e ".[test]"
    $ pytest

(On Windows, activate with ``.venv\\Scripts\\activate`` instead.)
``pip install -e .`` puts the ``tmdl-preflight`` console command on
your PATH and registers the ``tmdl_preflight`` pytest plugin. Verify
with:

::

    $ tmdl-preflight --version

    tmdl-preflight 1.2.0

Project layout
------------------

::

    src/tmdl_preflight/
    |-- cli.py             # the tmdl-preflight command (16 subcommands)
    |-- engine.py          # check -> fix -> re-check loop, scanned counts, apply_policy()
    |-- config.py          # config discovery and validation, profiles, excludes, exemptions
    |-- baseline.py        # baseline files: record, apply, update
    |-- output.py          # text / json / github / sarif / markdown renderers
    |-- console.py         # the console reports the rules that print their own
    |-- catalog.py         # the generated rule catalog page (rules --format markdown)
    |-- categories.py      # analysis categories: the encoded mapping table
    |-- fieldparams.py     # which files are field parameters, and how to read their rows
    |-- format_resolver.py # what format a measure name resolves to
    |-- ifilters.py        # the applied-filters summary measure
    |-- live.py            # the opt-in checks against a published dataset
    |-- manifest.py        # the JSON manifest of a model
    |-- measure_classes.py # is a measure user-facing or plumbing?
    |-- nameof.py          # NAMEOF extraction and resolution
    |-- ratchet.py         # per-rule ratchet files ("floors")
    |-- reconcile.py       # a manifest against a warehouse schema snapshot
    |-- refresh.py         # incremental refresh: the refreshPolicy block
    |-- refs.py            # the reference census (refs)
    |-- reports.py         # the read-only reports (complexity, nameof, relationships)
    |-- roles.py           # table roles: fact, dimension, selector, measure table
    |-- sqlviews.py        # prove view-column exposure from SQL files
    |-- backlog.py         # editing the refresh-policy backlog in place
    |-- stats.py           # the stats inventory
    |-- model.py           # parsed SemanticModel: tables, columns, measures, ...
    |-- parser.py          # TMDL / PBIP folder parsing, path resolution
    |-- dax.py             # DAX region classifier and lexing helpers
    |-- recognizers.py     # line-level TMDL recognizers shared by checks and fixers
    |-- edits.py           # read_text_file() / verified_write() for every fixer
    |-- pytest_plugin.py   # the tmdl_preflight fixture
    `-- rules/
        |-- base.py             # Rule, Violation, Severity, Context, RuleSet
        |-- structural.py       # M001-M016
        |-- hygiene.py          # M017-M019, M022
        |-- cultures.py         # M021
        |-- validation_notes.py # M023
        |-- dax_rules.py        # D001-D005
        |-- dax_patterns.py     # D010, D011, X001
        |-- grain.py            # C009-C011, D012
        |-- relationships.py    # R001-R008
        |-- topology.py         # R009-R016
        |-- field_parameters.py # F002, F003, F005
        |-- fp_rows.py          # F001, F004, F006, F007
        |-- fp_axes.py          # M020, F008-F015
        |-- fp_layout.py        # F016, F017
        |-- report.py           # B001-B006
        |-- report_format.py    # B007, B008
        |-- rls.py              # L001-L006
        |-- mapping.py          # A001-A005
        |-- contracts.py        # C001-C004
        |-- contract_patterns.py # C005-C008
        |-- measure_hygiene.py  # O008-O015, C012
        |-- organization.py     # S003, O001-O007, N020
        |-- refresh_policy.py   # P001, P002
        |-- comments.py         # S004
        |-- formats.py          # S005
        |-- style.py            # S001, S002
        `-- __init__.py         # ALL_RULES registry + default_ruleset()

    tests/       # pytest suite; fixtures/ holds the clean example model
    examples/    # bike-shop-clean and bike-shop-broken PBIP projects, plus ci/

The full Diátaxis documentation (a tutorial, how-to guides, and
reference pages) lives in this separate ``tmdl-preflight-documentation``
repository, published on Read the Docs.

Each rule has an ``id`` (``M003``), a ``name`` (``lineage-duplicates``),
a ``Severity`` (``ERROR``/``WARNING``/``INFO``), and a one-paragraph
``description``. A rule's ``check()`` returns ``Violation``\s; a rule
that can repair its own violation class sets ``fixable = True`` and
implements ``fix()``. ``baselinable = False`` marks defects a baseline
may never hide, ``opt_in = True`` marks a rule that runs only when it
is named, ``safe = False`` marks a fixer that never runs in a blanket
``fix``, and ``option_defaults`` declares the per-rule options a
configuration file may set.

Design principles
---------------------

- **The detector and the fixer share one recognizer.** A fixable
  rule's ``check()`` and ``fix()`` call the same function, so they can
  never disagree about what counts as a defect.
- **Every edit is surgical and verified.** Fixers change file text in
  place and write through ``edits.verified_write()``, whose
  ``verify(old, new)`` callback proves the edit first. If the proof
  fails, nothing is written and the violation stays on the report.
- **Every rule test plants a defect.** Each rule has a clean-case test
  and at least one test that breaks a copy of the clean fixture in
  exactly one way and asserts the rule fires.
- **Rules that can fire on clean, Desktop-generated models are
  warning or info.** Errors are reserved for defects that stop a model
  from opening, deploying or evaluating.

Adding a new rule
---------------------

1. Subclass ``Rule`` in the appropriate ``rules/*.py`` module, or a new
   one. Give it a unique ``id``, a ``name``, a ``severity``, and a
   ``description``, then implement
   ``check(self, ctx) -> list[Violation]``:

   ::

       from .base import Context, Rule, Severity, Violation

       class TableDescriptionRule(Rule):
           id = "Y001"                 # unique; a prefix outside M/D/R/F/B/S/A/C/L/N/O/P/X
           name = "table-descriptions"
           severity = Severity.WARNING
           description = "Visible tables should carry a /// description comment."

           def check(self, ctx: Context) -> list[Violation]:
               out = []
               for model in ctx.models:
                   for table in model.tables.values():
                       if not table.is_hidden and needs_description(table):
                           out.append(self.violation(
                               "visible table has no /// description",
                               file=table.file, line=table.line, obj=table.name,
                           ))
               return out

   ``ctx.models`` are parsed ``SemanticModel``\\s; ``ctx.report_dirs``
   are the ``*.Report`` folders for report-side rules; ``ctx.reload()``
   drops the parse cache (the engine calls it after fixers run).

2. Optionally add a fixer. Set ``fixable = True`` and implement
   ``fix(self, ctx) -> list[str]``, returning one human-readable line
   per change. Honor the safety contract: only repair what the failure
   fully determines, be idempotent, make minimal line-level edits, and
   never overwrite human-authored semantics (names, DAX,
   relationships). Write the file through ``edits.verified_write()``.
   The engine re-checks from disk afterward, so a fixer never vouches
   for itself.

3. Register it in ``src/tmdl_preflight/rules/__init__.py``: import the
   class and add it to the ``ALL_RULES`` list.

4. Add a test in ``tests/`` (see Testing, below).

5. Document it: add an entry to the rule catalog in this
   documentation's :doc:`reference/usage`, and update
   :doc:`how-to/modify-the-rules-list` or
   :doc:`how-to/modify-the-auto-fixes` if the change is substantial
   enough to affect the recipe those pages teach.

Testing
-----------

Tests live in ``tests/``, one ``test_rules_*.py`` file per rule
module. The shared fixtures in ``tests/conftest.py`` give you a
pristine copy of the Pedal and Sprocket Bike Co. model:

- ``project``, a disposable copy of the clean model in ``tmp_path``,
  safe to mutate.
- ``definition``/``report_dir``, the model and report folders inside
  it.
- ``clean_project``, the pristine fixture; never mutate it.

Follow the built-in pattern: copy the clean fixture, break exactly one
thing to trigger your failure mode, assert the violation is detected,
and, for a fixer, assert that fix, then re-check, comes back clean and
that a second fix is a no-op (idempotent). For example:

::

    def test_duplicate_lineage_tag_is_flagged(project, definition):
        stores = definition / "tables" / "Stores.tmdl"
        # ...copy Products' lineageTag onto Stores, then:
        violations = LineageTagUniquenessRule().check(Context(project))
        assert any(v.rule_id == "M003" for v in violations)

Before you open a PR
-------------------------

Run both of these; both must pass:

::

    $ pytest

::

    $ tmdl-preflight check examples/bike-shop-clean

    BikeShop.Report/definition/pages/pages.json  B005 error: definition/pages/pages.json is missing; the report has no pages metadata. Save the report again from Power BI Desktop (PBIR format). [BikeShop.Report]
    BikeShop.Report/definition/report.json  B008 error: BikeShop.Report: definition/report.json is missing; Power BI Desktop cannot open a report folder without it.
    BikeShop.Report/definition/version.json  B008 error: BikeShop.Report: definition/version.json is missing; Power BI Desktop cannot open a report folder without it.
    BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl:26  M017 info: Line 26: Multiline DAX should use triple backticks
    BikeShop.SemanticModel/definition/tables/__Calendar.tmdl  R013 info: calendar __Calendar has no Year, Quarter, Month, Week, Day column(s). [__Calendar]

    scanned: 1 model, 6 tables, 22 columns, 6 measures, 4 relationships, 8 DAX expressions, 1 report
    (the run then prints its console report: the DAX validation summary
    and the R001, R002, R004, R005, R009, R010, R012, R013 and R014 blocks)
    tmdl-preflight: 3 error(s), 0 warning(s), 2 info(s)

The second command is the end-to-end smoke test. Its output is fixed:
the shipped example is what it is, and the findings above are the ones
it produces. The three errors all come from the report layer, not from
the model: the example's ``BikeShop.Report`` folder is in the older
PBIR layout (its ``report.json`` sits at the report root, and it has
no ``definition/version.json`` and no ``definition/pages/pages.json``),
and the B005 and B008 rules want the file set a report saved by Power
BI Desktop has. The model itself is clean: the run reports nothing but
two info findings (M017 and R013) against it. The command therefore
exits ``1``, and a change to that output is a change to the smoke
test.

If you touched a fixer, also confirm ``tmdl-preflight fix`` on a copy
of ``examples/bike-shop-broken`` repairs the four seeded defects and
leaves the report-layer errors and the info findings behind:

::

    $ cp -r examples/bike-shop-broken /tmp/bike-shop
    $ tmdl-preflight fix /tmp/bike-shop
    tmdl-preflight: 3 error(s), 0 warning(s), 3 info(s), 4 fix(es) applied

That summary line is the whole verdict: four fixers ran, and the
re-check still finds the three report-layer errors and three info
findings. The exit code is ``1``: ``fix`` reports the state of the run
rather than hiding it.

Commit and PR conventions
------------------------------

- Branch off ``master``; keep each PR focused on one rule, fix, or
  topic.
- Write imperative, present-tense commit subjects ("Add M009
  partition-mode rule," not "Added..."); keep the subject under about
  seventy-two characters and explain the why in the body when it is
  not obvious.
- Keep auto-fix changes to a model in their own commit so reviewers
  can wave them through.
- A PR should describe what it changes and why, note any new or
  changed rule ids, and confirm that ``pytest`` passes and that the
  clean-example check prints the report above unchanged.

Reporting issues
---------------------

Open an issue on the `GitHub tracker
<https://github.com/C-Kapsalis/tmdl-preflight/issues>`_ with:

- what you ran, the exact ``tmdl-preflight`` command and flags,
- what you expected versus what happened (paste the report output),
- your Python version and OS, and
- a minimal TMDL/PBIP snippet that reproduces it, if you can share
  one.

Security-sensitive reports should go to the maintainer privately
rather than a public issue.

License
-----------

By contributing you agree that your contributions are licensed under
the project's MIT License (copyright 2026 Christoforos Kapsalis).
