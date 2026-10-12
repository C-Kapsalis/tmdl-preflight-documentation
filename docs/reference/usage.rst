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
     - Comma-separated rule ids or names to run (default: all, or the
       configuration's ``select``). Case-insensitive.
   * - ``--ignore IDS``
     - Comma-separated rule ids or names to skip, in addition to the
       configuration's ``ignore``.
   * - ``--strict``
     - Exit nonzero on warnings as well as errors.
   * - ``--format FMT``
     - ``text`` (default), ``json``, ``github``, ``sarif`` or
       ``markdown``; see :doc:`configuration`.
   * - ``--json``
     - Same as ``--format json``.
   * - ``--baseline FILE``
     - Fail only on violations not recorded in ``FILE``.
   * - ``--config FILE``
     - Read the configuration from ``FILE`` instead of searching for
       one.
   * - ``--profile NAME``
     - Merge the configuration's ``profiles.NAME`` table.
   * - ``--no-config``
     - Ignore any configuration file.

``tmdl-preflight fix <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule; for each fixable rule that reported
violations, apply its auto-fix; then re-run every selected rule from
disk and report what remains. Takes the same options as ``check``.

The exit code reflects the post-fix state: ``0`` means the re-check
came back clean, ``1`` means violations survived the repair, or were
never fixable in the first place.

``tmdl-preflight baseline <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Record today's violations in ``<path>/.tmdl-preflight-baseline.json``
(or ``--output FILE``) so ``check --baseline`` fails only on new ones.
``--update`` shrinks an existing baseline to what still fails and never
adds entries; ``--update --accept-new`` also adds today's new
violations. Takes ``--config``, ``--profile`` and ``--no-config``; it
refuses ``--select``/``--ignore``. Details: :doc:`configuration`.

``tmdl-preflight stats <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print an inventory of the project: tables, columns, measures,
relationships (active, inactive, bidirectional), partitions by kind,
calculation groups and items, field parameters, parameters,
descriptions, display folders, lineage tags, report pages, visuals and
bookmarks. Counts only, no thresholds. ``--json`` emits a JSON object.

``tmdl-preflight rules``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print the rule catalog: id, name, severity, fixability, description.
``--json`` emits it as a JSON array, including ``baselinable`` and the
rule's ``options``. The top-level flag ``tmdl-preflight --rules`` is an
alias.

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
       ``--select``/``--ignore``, the selection leaves zero rules to
       run, the configuration is invalid, the baseline file is missing
       or malformed, or ``baseline`` refused to write

Text output format
-----------------------

::

    <file>:<line>  <RULE> <severity>: <message> [<object>] (auto-fixable)

Files are shown relative to the checked path when possible. ``fix``
runs prefix the report with one ``fixed  <RULE>: <description>`` line
per applied repair. The report ends with a ``scanned:`` line (how many
models, tables, columns, measures, relationships and DAX expressions
the run looked at), any notes about excluded paths, exemptions and the
baseline, and the final ``tmdl-preflight:`` summary line.

JSON output format
-----------------------

::

    {
      "summary": {
        "errors": 0,
        "warnings": 0,
        "infos": 0,
        "fixes_applied": 1,
        "excluded": 0,
        "exempted": 0,
        "baselined": 0
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
          "fixable": true,
          "path": "BikeShop.SemanticModel/definition/tables/Stores.tmdl"
        }
      ],
      "scanned": {"models": 1, "tables": 6, "columns": 22, "measures": 6,
                  "relationships": 4, "dax_blocks": 8, "...": "..."}
    }

``violations`` reflects the final state of the run (post-fix for
``fix``). ``path`` is project-relative (it starts at the
``*.SemanticModel`` or ``*.Report`` folder), the same identity
baselines and exemptions use. When they apply, the payload also has an
``exemptions`` object (applied, expired, unused) and a ``baseline``
object (suppressed, fixed, stale). All of these fields were added in
1.1.0 without changing the earlier ones.

Rule catalog
----------------

Severity semantics: **error**, will break an import or deploy, or
silently corrupt results; **warning**, suspicious, deserves a look
before shipping; **info**, advisory, never blocks, not even with
``--strict``. Select rules by id or name, case-insensitively:
``--select M003`` and ``--select lineage-duplicates`` are equivalent.
A configuration file can re-grade any rule (see
:doc:`configuration`).

Thirty-seven rules ship today; ten carry a fixer (M003, M004, M006,
M011, M013, F001, F002, F003, B001, B003). Six can never be baselined
because they stop a model from opening or deploying: M003, M006, M007,
M010, M015, R006. Adding a rule of your own is
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
       declare a table. A declaration the parser cannot attach (for
       example a mis-indented measure) is reported as "parser dropped
       N object(s)".
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
   * - M010
     - metadata-in-expression
     - error
     - no
     - A TMDL property line (``lineageTag:``, ``isHidden``, ``annotation`` ...) must not sit inside an expression body.
   * - M011
     - property-after-child-block
     - warning
     - **yes**
     - Simple properties must come before an object's ``annotation``, ``changedProperty`` and ``extendedProperty`` blocks.
   * - M012
     - duplicate-object-names
     - error
     - no
     - No two tables whose names differ only by case, no duplicate column or measure in one table, no measure named like a column of its table.
   * - M013
     - dangling-description
     - error (warning at end of file)
     - **yes**
     - A ``///`` description must sit directly on top of a declaration, not before a blank line.
   * - M014
     - calc-group-implicit-measures
     - error
     - no
     - A model with a calculation group must set ``discourageImplicitMeasures``.
   * - M015
     - ref-table-target-missing
     - error
     - no
     - Every ``ref table`` line in ``model.tmdl`` must name a table that exists.
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
     - Measure names must be unique across the whole model,
       case-insensitively; duplicates inside one table are M012's.
   * - D004
     - dax-reference-resolution
     - error (warning for bare ``[X]``)
     - no
     - Every table and qualified column a DAX expression names must exist.
   * - D005
     - duplicate-var-in-scope
     - error
     - no
     - The same ``VAR`` may not be declared twice in one scope.
   * - R001
     - relationship-endpoints
     - error
     - no
     - A relationship's from/to tables and columns must exist (names
       compare case-insensitively), and both endpoints must be present.
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
   * - R006
     - ambiguous-active-pair
     - error
     - no
     - At most one active relationship between the same two tables.
   * - R007
     - autodetected-relationships
     - warning
     - no
     - Relationships Desktop autodetected (``AutoDetected_*`` ids) should be reviewed.
   * - F001
     - fieldparam-comma-runs
     - error
     - **yes**
     - Two or more commas separated only by whitespace in a calculated
       partition source, the classic leftover of deleting a row without
       its separator, make the source unparseable.
   * - F002
     - fieldparam-missing-separator
     - error
     - **yes**
     - Rows of a ``{ (...), (...) }`` constructor must be separated by commas.
   * - F003
     - fieldparam-trailing-comma
     - error
     - **yes**
     - No comma after the last row, before the closing ``}``.
   * - F004
     - fieldparam-ragged-rows
     - warning
     - no
     - Every row of one constructor should have the same number of values.
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
   * - B003
     - bookmark-options-schema
     - error
     - **yes**
     - A bookmark's ``options`` may only use the keys the report schema accepts.
   * - B004
     - report-definition-integrity
     - error
     - no
     - ``definition.pbir`` must point at an existing model, ``pages.json`` entries must exist, page and visual JSON must parse.
   * - S001
     - format-strings
     - info
     - no
     - Visible measures should declare a ``formatString``. A measure
       with its own ``formatStringDefinition`` is exempt; a model whose
       calculation-group items define format strings is exempt as a
       whole.
   * - S002
     - measure-descriptions
     - info
     - no
     - Visible measures should carry a real ``///`` description (configurable minimum length and placeholder words).

Rules added in 1.1.0
~~~~~~~~~~~~~~~~~~~~~~~~

Each entry says what the rule checks, why it matters, and shows a
minimal example of the defect.

**M010 metadata-in-expression** (error, no fixer). A property line has
landed inside an expression, usually because a script inserted it at
the wrong indentation or inside a ````` fence. The property becomes
part of the DAX text and Power BI rejects the file ("... is neither a
property nor an object in the current context"). Move the line out of
the expression, one level under the declaration. Never baselinable.

::

    measure Revenue =
            SUM ( Sales[Amount] )
            formatString: #,0        <- read as DAX

**M011 property-after-child-block** (warning, fixer). Desktop always
writes an object's simple properties (``isHidden``, ``displayFolder``,
``formatString``, ``lineageTag`` ...) before its ``annotation``,
``changedProperty`` and ``extendedProperty`` blocks. A property
appended after them is at best ignored and at worst rejected. The fixer
moves the line up into the property section.

::

    column Region
        dataType: string
        annotation SummarizationSetBy = Automatic
        isHidden                 <- after a child block

**M012 duplicate-object-names** (error, no fixer). The engine compares
names case-insensitively. Two table files declaring ``Sales`` and
``SALES``, two ``Revenue`` measures in one table, or a measure named
like a column of its own table all stop the model from loading.

**M013 dangling-description** (error, warning at the end of a file;
fixer). A ``///`` run followed by a blank line describes nothing, and
Power BI rejects the file once a declaration follows ("Unexpected line
type: Empty"). At the end of a file it is harmless until something is
appended. The fixer deletes the run and one adjacent blank line,
keeping the file's newline style.

::

    /// Net sales after discounts.
                                 <- blank line orphans the description
    measure Revenue = SUM ( Sales[Amount] )

**M014 calc-group-implicit-measures** (error, no fixer). Calculation
groups apply only to explicit measures, so a model with a calculation
group must set ``discourageImplicitMeasures`` under ``model`` in
``model.tmdl``.

**M015 ref-table-target-missing** (error, no fixer). A ``ref table``
line whose table file was deleted or renamed stops Desktop from
opening the project. Restore the file or delete the line. Never
baselinable.

**D004 dax-reference-resolution** (error; warning for bare
references; no fixer). TMDL import does not resolve DAX names, so a
renamed table imports cleanly and then fails on every visual
("Cannot find table"). ``'T'[c]``, ``T[c]`` and a bare ``'T'`` must
resolve, case-insensitively, to a table and a column or measure on it.
An unqualified ``[X]`` that matches no measure, no column and no name
the expression itself introduces (``ADDCOLUMNS``, ``SELECTCOLUMNS``,
``SUMMARIZECOLUMNS`` and similar name columns with string literals) is
a warning. Strings and comments are never scanned, and the auto
date/time tables (``DateTableTemplate_*``, ``LocalDateTable_*``) are
always legal. Option: ``check_bare_references`` (default ``true``).

::

    measure Revenue = SUM ( Orders[Amount] )   -- table Orders was renamed to Sales

**D005 duplicate-var-in-scope** (error, no fixer). Declaring the same
``VAR`` twice at the outer level of an expression, a common merge
accident, fails on evaluation. Reusing a name inside separate nested
sub-expressions is legal and not reported.

::

    VAR total = SUM ( Sales[Amount] )
    VAR total = 0                 <- same scope
    RETURN total

**R006 ambiguous-active-pair** (error, no fixer). Two active
relationships between the same two tables make the filter path
ambiguous and the model cannot load. Mark all but one ``isActive:
false`` and use ``USERELATIONSHIP()``. Never baselinable.

**R007 autodetected-relationships** (warning, no fixer). Ids starting
with ``AutoDetected_`` were created by Desktop's relationship
autodetection and often join columns that merely share a name. Review
each one, then keep it (ignore or baseline the rule) or delete it.

**F002 fieldparam-missing-separator** (error, fixer). A row appended
to a ``{ (...), (...) }`` constructor without a comma after the
previous row makes the source unparseable and the table loads empty.
Rows may span lines or share one; the fixer inserts the comma.

::

    ( "Revenue", NAMEOF ( [Revenue] ), 0 )     <- comma missing
    ( "Orders", NAMEOF ( [Orders] ), 1 )

**F003 fieldparam-trailing-comma** (error, fixer). A comma after the
last row, left behind when that row was deleted, makes the source
unparseable. The fixer deletes it.

**F004 fieldparam-ragged-rows** (warning, no fixer). Rows with
different numbers of values usually mean a missing or extra value
inside a row. Only a human knows which value is wrong.

**B003 bookmark-options-schema** (error, fixer). A bookmark's
``options`` may only contain ``targetVisualNames``,
``applyOnlyToTargetVisuals``, ``suppressData``,
``suppressActiveSection`` and ``suppressDisplay``; any other key fails
schema validation. The fixer cuts only the illegal members out of the
file text and proves the edit by parsing both versions.

::

    "options": { "targetVisualNames": ["a1b2"], "suppressDatta": true }

**B004 report-definition-integrity** (error, no fixer).
``definition.pbir``'s ``byPath`` must point at an existing
``*.SemanticModel`` folder, every page named in ``pages/pages.json``
(``pageOrder``, ``activePageName``) needs its folder, and every
``page.json`` and ``visual.json`` must be valid JSON.

**S002 measure-descriptions** (info, no fixer). Visible measures
should carry a real ``///`` description: it is the tooltip report
authors see. Descriptions shorter than ``min_length`` (default 10) or
containing a word from ``placeholder_words`` (default ``todo``,
``tbd``, ``tbc``, ``fixme``, ``xxx``, ``placeholder``, ``lorem``) do
not count.

Usage patterns
------------------

Running in CI
~~~~~~~~~~~~~~~~~~

The one rule of CI usage: **CI checks; developers fix.** Run
``tmdl-preflight check`` (read-only) in the pipeline; auto-fixes belong
on a developer's machine, where the diff gets reviewed and committed
like any other change. :doc:`../how-to/run-in-ci` covers pre-commit,
GitHub Actions with pull-request annotations and SARIF upload, Azure
DevOps, and gating on a baseline.

Scoping a run
~~~~~~~~~~~~~~~~~

- Check one model in a multi-model repo: pass the specific folder,
  ``tmdl-preflight check src/Finance.SemanticModel``.
- Silence an advisory rule that does not apply to your architecture,
  for example S001 in a model that formats via calculation groups:
  ``tmdl-preflight check . --ignore S001``, or ``ignore = ["S001"]`` in
  the configuration file to make it permanent.
- Gate on a subset while cleaning up legacy debt:
  ``tmdl-preflight check . --select M002,M003,D001,R001``, or record a
  baseline and fail only on new violations (see :doc:`configuration`).
- Apply every safe repair in one pass, before cutting a release branch,
  without being distracted by advisory findings:
  ``tmdl-preflight fix MyProject/ --select M003,M004,M006,M011,M013,F001,F002,F003,B001,B003``.
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

The CLI itself reads no environment variables; project settings come
from the configuration file (:doc:`configuration`). The pytest fixture
honors ``TMDL_PREFLIGHT_AUTOFIX=1`` as a default for its ``autofix``
parameter; see :doc:`customization`.
