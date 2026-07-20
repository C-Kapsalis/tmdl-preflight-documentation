Usage reference
===================

Everything the ``tmdl-preflight`` command accepts, plus the full rule
catalog and the usage patterns that come up most: CI, scoping a run,
and consuming machine-readable output.

Command syntax
------------------

::

    tmdl-preflight [--version] [--rules] <command> [options]

``tmdl-preflight check <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule and report violations. Strictly read-only;
never modifies a file.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--select IDS``
     - Comma-separated rule ids or names to run (default: all).
       Case-insensitive.
   * - ``--ignore IDS``
     - Comma-separated rule ids or names to skip.
   * - ``--json``
     - Emit the machine-readable report instead of text.
   * - ``--strict``
     - Exit nonzero on warnings as well as errors.

``tmdl-preflight fix <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule; for each fixable rule that reported
violations, apply its auto-fix; then re-run every selected rule from
disk and report what remains. Takes the same options as ``check``.

The exit code reflects the post-fix state: ``0`` means the re-check
came back clean, ``1`` means violations survived the repair, or were
never fixable in the first place.

``tmdl-preflight rules``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print the rule catalog: id, name, severity, fixability, description.
``--json`` emits it as a JSON array. The top-level flag
``tmdl-preflight --rules`` is an alias.

``<path>`` resolution
-------------------------

.. list-table::
   :header-rows: 1

   * - You pass
     - tmdl-preflight uses
   * - a PBIP project root
     - every ``*.SemanticModel/definition`` and every ``*.Report`` found
       beneath it
   * - a ``*.SemanticModel`` folder
     - its ``definition/``
   * - a ``definition`` folder (has ``tables/``)
     - itself
   * - a ``*.Report`` folder
     - itself (report-side rules only)

Multiple models or reports under one root are all checked in a single
run.

Exit codes
-------------

.. list-table::
   :header-rows: 1

   * - Code
     - Meaning
   * - ``0``
     - clean (after fixes, for ``fix``)
   * - ``1``
     - violations remain, errors always, warnings too with ``--strict``
   * - ``2``
     - usage error: the path does not exist, nothing to check was
       found there, an unknown rule id or name was given to
       ``--select``/``--ignore``, or the selection leaves zero rules
       to run

Text output format
-----------------------

::

    <file>:<line>  <RULE> <severity>: <message> [<object>] (auto-fixable)

Files are shown relative to the checked path when possible. ``fix``
runs prefix the report with one ``fixed  <RULE>: <description>`` line
per applied repair.

JSON output format
-----------------------

::

    {
      "summary": {
        "errors": 0,
        "warnings": 0,
        "infos": 0,
        "fixes_applied": 1
      },
      "fixes": ["M003: Stores.tmdl:3 (table Stores): '...' -> '...'"],
      "violations": [
        {
          "rule": "M003",
          "severity": "error",
          "message": "...",
          "file": "...",
          "line": 3,
          "object": "table Stores",
          "fixable": true
        }
      ]
    }

``violations`` reflects the final state of the run (post-fix for
``fix``).

Rule catalog
----------------

Severity semantics: **error**, will break an import or deploy, or
silently corrupt results; **warning**, suspicious, deserves a look
before shipping; **info**, advisory, never blocks, not even with
``--strict``. Select rules by id or name, case-insensitively:
``--select M003`` and ``--select lineage-duplicates`` are equivalent.

Twenty-one rules ship today; five carry a fixer (M003, M004, M006,
F001, B001). Adding a rule of your own is
:doc:`../how-to/modify-the-rules-list`; giving one a fixer is
:doc:`../how-to/modify-the-auto-fixes`.

.. list-table::
   :header-rows: 1

   * - ID
     - Name
     - Severity
     - Fixable
     - What it checks
   * - M001
     - model-structure
     - error
     - no
     - ``definition/`` must contain ``model.tmdl`` and a non-empty
       ``tables/`` folder.
   * - M002
     - tmdl-well-formed
     - error
     - no
     - Every ``.tmdl`` file must decode as UTF-8, contain no null
       bytes, have balanced expression fences, and, for table files,
       declare a table.
   * - M003
     - lineage-duplicates
     - error
     - **yes**
     - No two objects may share a ``lineageTag``; duplicates are
       rejected at import.
   * - M004
     - lineage-format
     - warning
     - **yes**
     - A ``lineageTag`` must be a canonical hyphenated UUID, not a
       placeholder or hand-typed value.
   * - M005
     - column-data-types
     - warning
     - no
     - Every declared column ``dataType`` must be one of ``string``,
       ``int64``, ``double``, ``decimal``, ``dateTime``, ``boolean``,
       ``binary``, ``variant``, ``currency``, ``rowNumber``.
   * - M006
     - model-table-refs
     - error
     - **yes**
     - Every ``tables/*.tmdl`` file needs a matching ``ref table``
       line in ``model.tmdl``; without one the table is not attached
       to the model and Power BI Desktop refuses to open the project.
   * - M007
     - table-partitions
     - error
     - no
     - Every table needs at least one partition, even a measures-only
       table; without one Power BI crashes on open resolving the
       table's query.
   * - M008
     - entity-query-source
     - error
     - no
     - An inline ``#table(type table [...])`` M source is entity-typed;
       combined with any calculated table, Power BI treats the model as
       composite and refuses to open it.
   * - M009
     - reserved-table-name
     - error
     - no
     - Power BI reserves certain table names and refuses to open a
       model that uses one, most notably a table literally named
       ``Measures``.
   * - D001
     - dax-delimiters
     - error
     - no
     - Parentheses, braces, brackets, and quoted strings must balance
       in every DAX expression.
   * - D002
     - nameof-resolution
     - error (warning if ambiguous)
     - no
     - ``NAMEOF('Table'[Member])`` must resolve to a real column or
       measure; an ambiguous bare-measure reference is downgraded to a
       warning, since D003 already reports the underlying duplicate.
   * - D003
     - duplicate-measure-names
     - error
     - no
     - Measure names must be unique across the whole model, not merely
       within one table.
   * - R001
     - relationship-endpoints
     - error
     - no
     - A relationship's from/to tables and columns must exist.
   * - R002
     - relationship-cardinality
     - error
     - no
     - ``fromCardinality``/``toCardinality``, where declared, must be
       ``one`` or ``many``.
   * - R003
     - relationship-duplicates
     - warning
     - no
     - Two relationships over the same column pair.
   * - R004
     - relationship-bidirectional
     - info
     - no
     - Surfaces every ``bothDirections`` cross-filter so the modeler
       confirms it is intentional.
   * - R005
     - orphan-tables
     - info
     - no
     - Visible-column data tables with no relationships; measure-only,
       hidden, measure-home, field-parameter, and calculated tables are
       exempt by design.
   * - F001
     - fieldparam-comma-runs
     - error
     - **yes**
     - Two or more commas separated only by whitespace in a calculated
       partition source, the classic leftover of deleting a row without
       its separator, make the source unparseable.
   * - B001
     - bookmark-int-types
     - error
     - **yes**
     - ``howCreated``, ``ComparisonKind``, and ``Version`` in a
       ``*.bookmark.json`` must be JSON integers, not strings.
   * - B002
     - bookmark-visual-refs
     - error
     - no
     - Every visual a bookmark references must exist under
       ``pages/*/visuals/``; the right resolution depends on intent, so
       there is no fixer.
   * - S001
     - format-strings
     - info
     - no
     - Visible measures should declare a ``formatString``; models using
       ``formatStringDefinition`` (dynamic formatting via calculation
       groups) are exempted automatically.

Usage patterns
------------------

Running in CI
~~~~~~~~~~~~~~~~~~

The one rule of CI usage: **CI checks; developers fix.** Run
``tmdl-preflight check`` (read-only) in the pipeline; auto-fixes belong
on a developer's machine, where the diff gets reviewed and committed
like any other change.

::

    name: tmdl-preflight
    on:
      pull_request:
        paths:
          - "**/*.SemanticModel/**"
          - "**/*.Report/**"

    jobs:
      preflight:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - uses: actions/setup-python@v5
            with:
              python-version: "3.12"
          - run: pip install tmdl-preflight
          - run: tmdl-preflight check . --strict

``--strict`` makes warnings fail the job as well as errors; drop it if
only hard failures should block. Azure DevOps is the same shape:

::

    steps:
      - task: UsePythonVersion@0
        inputs:
          versionSpec: "3.12"
      - script: pip install tmdl-preflight
      - script: tmdl-preflight check $(Build.SourcesDirectory) --strict
        displayName: TMDL preflight

Scoping a run
~~~~~~~~~~~~~~~~~

- Check one model in a multi-model repo: pass the specific folder,
  ``tmdl-preflight check src/Finance.SemanticModel``.
- Silence an advisory rule that does not apply to your architecture,
  for example S001 in a model that formats via calculation groups:
  ``tmdl-preflight check . --ignore S001``.
- Gate on a subset while cleaning up legacy debt:
  ``tmdl-preflight check . --select M002,M003,D001,R001``.
- Apply every safe repair in one pass, before cutting a release branch,
  without being distracted by advisory findings:
  ``tmdl-preflight fix MyProject/ --select M003,M004,M006,F001,B001``.
  Preview first with the same selection under ``check``; every line
  marked ``(auto-fixable)`` is one the subsequent ``fix`` will touch.
  Commit the fixes as their own commit, "apply tmdl-preflight
  auto-fixes," so reviewers can wave it through quickly.

An unknown id or name in ``--select``/``--ignore`` is a usage error
(exit 2), not a silently smaller selection; run ``tmdl-preflight
rules`` to see the valid ids and names.

Consuming JSON output
~~~~~~~~~~~~~~~~~~~~~~~~~~

For annotations or dashboards, use ``--json`` and parse the payload:

::

    $ tmdl-preflight check . --json > preflight.json

Exit codes are unchanged under ``--json``: ``0`` clean, ``1``
violations, ``2`` usage or path errors, so a missing model folder fails
the job loudly instead of passing vacuously.

Environment
---------------

The CLI itself reads no environment variables. The pytest fixture
honors ``TMDL_PREFLIGHT_AUTOFIX=1`` as a default for its ``autofix``
parameter; see :doc:`customization`.
