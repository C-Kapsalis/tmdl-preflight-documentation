Usage reference
===================

Everything the ``tmdl-preflight`` command accepts, plus the usage
patterns that come up most: CI, scoping a run, and consuming
machine-readable output. The rule catalog has its own page,
:doc:`rules`.

Command syntax
-----------------

::

    tmdl-preflight [--version] [--rules] <command> [options]

The command has sixteen subcommands.

.. list-table::
   :header-rows: 1

   * - Command
     - What it does
   * - ``check``
     - Report violations; never modifies files.
   * - ``fix``
     - Apply auto-fixes, then re-check (the imposition pattern).
   * - ``baseline``
     - Record today's violations so ``check --baseline`` fails only on
       new ones.
   * - ``stats``
     - Print an inventory of the project (counts only).
   * - ``dead-facts``
     - Report the fact tables no measure reads, and record them as a
       floor.
   * - ``comment-refs``
     - Count ticket references (``#123``) in model comments.
   * - ``report``
     - Read-only reports: ``complexity``, ``nameof`` or
       ``relationships``.
   * - ``format-resolve``
     - Resolve the format a measure name gets from a format calculation
       group.
   * - ``grain``
     - Derive which dimensions can filter a cost fact and its backend
       facts; check a grain guard against it.
   * - ``info``
     - Derived model facts: the analysis categories.
   * - ``fieldparam``
     - Edit field-parameter rows (``fieldparam retag``).
   * - ``manifest``
     - Write a JSON manifest of a model: tables, roles, sources,
       relationships.
   * - ``schema-check``
     - Compare a manifest with a warehouse schema snapshot, or prove
       view columns from SQL files.
   * - ``refs``
     - Count the places that name an object (model and report), before
       a rename or delete.
   * - ``live``
     - Opt-in checks against the published dataset (Power BI REST;
       ``--replay`` answers from recordings).
   * - ``rules``
     - List the rule catalog.

``tmdl-preflight check <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
   * - ``--capture-floor FILE``
     - Write the ratchet floor of the selected rules to ``FILE`` and
       print what was captured. The ratcheted rules are O007
       ``dead-fact-tables``, R008 ``fact-conformed-dimension-join`` and
       R016 ``dead-anchor-calendar``.
   * - ``--config FILE``
     - Read the configuration from ``FILE`` instead of searching for
       one.
   * - ``--profile NAME``
     - Merge the configuration's ``profiles.NAME`` table.
   * - ``--no-config``
     - Ignore any configuration file.

``tmdl-preflight fix <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule; for each fixable rule that reported
violations, apply its auto-fix; then re-run every selected rule from
disk and report what remains.

The exit code reflects the post-fix state: ``0`` means the re-check
came back clean, ``1`` means violations survived the repair, or were
never fixable in the first place.

``fix`` takes every option ``check`` takes, and these:

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--dry-run``
     - Report what the fix would change; write nothing.
   * - ``--verbose``
     - List every file a dry run would change.
   * - ``--clear-cache``
     - After lineage tags change, delete the model's ``.pbi/cache.abf``
       so Desktop does not reuse the old tags.
   * - ``--keep-cache``
     - After lineage tags change, keep the model's ``.pbi/cache.abf``
       even on a single-model run.
   * - ``--check``
     - Write nothing; exit with the number of objects the selected
       fixers would act on (capped at 255).
   * - ``--only {missing,invalid}``
     - Narrow the refresh-policy fixers: only add missing policies, or
       only realign invalid ones.
   * - ``--no-prune``
     - The refresh-policy fixer keeps the backlog entries of the tables
       it repaired.

A fixer that is marked **unsafe** (D004, F016, P001 and R007) never
runs in a blanket fix; the run says so on a ``not fixed  <RULE>:`` line
and names the command that would apply it. Select the rule explicitly
to run it.

``tmdl-preflight baseline <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Record today's violations in ``<path>/.tmdl-preflight-baseline.json``
(or ``--output FILE``) so ``check --baseline`` fails only on new ones.
``--update`` shrinks an existing baseline to what still fails and never
adds entries; ``--update --accept-new`` also adds today's new
violations. Takes ``--config``, ``--profile`` and ``--no-config``; it
refuses ``--select``/``--ignore``. Details: :doc:`configuration`.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--output FILE``
     - The baseline file to write (default:
       ``<path>/.tmdl-preflight-baseline.json``).
   * - ``--update``
     - Shrink an existing baseline to what still fails; never adds
       entries.
   * - ``--accept-new``
     - With ``--update``, also add today's new violations.
   * - ``--capture-floors RULE``
     - Record the floor file of a ratcheted rule (O007, R008 or R016)
       from today's findings and write it out.

``tmdl-preflight stats <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print an inventory of the project: tables, columns, measures,
relationships (active, inactive, bidirectional), partitions by kind,
calculation groups and items, field parameters, parameters,
descriptions, display folders, lineage tags, report pages, visuals and
bookmarks. Counts only, no thresholds.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--json``
     - Emit a JSON object.
   * - ``--measures``
     - Add the measure-organisation summary: name classes, helper kinds,
       field-parameter surfacing, top tables, described helpers.
   * - ``--field-parameter-tables NAMES``
     - Comma-separated table names or globs whose ``NAMEOF`` rows count
       as surfaced (default: the detected field parameters).

``tmdl-preflight dead-facts <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

List the fact tables no measure reads. A fact table that only filter
sensors mention (``ISFILTERED``) or that only comments mention still
counts as dead.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--capture FILE``
     - Write the current dead fact tables to ``FILE`` as the floor: a
       JSON object of model key to table list, the models with none
       left out.
   * - ``--json``
     - Machine-readable output.

``tmdl-preflight comment-refs <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Count the ticket references in model comments (``#365``,
``org/repo#12``). Colours such as ``#FF00AA`` and names such as
``#Report Measures`` are not references.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--list [PREFIX]``
     - List every reference, optionally only the models whose key
       starts with ``PREFIX``.
   * - ``--json``
     - Machine-readable output.
   * - ``--floors FILE``
     - The ratchet file to compare with. With a floor file the command
       gates: a count above the floor fails the run, a fall is reported
       for tightening.
   * - ``--tighten``
     - Lower the floors in ``FILE`` to today's counts. Never raises, and
       drops the all-zero entries.

``tmdl-preflight report <kind> <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Read-only reports. The kind is one of:

.. list-table::
   :header-rows: 1

   * - Kind
     - What it reports
   * - ``complexity``
     - The length distribution of every measure and a threshold
       suggestion.
   * - ``nameof``
     - The ``NAMEOF`` inventory: how many references, how many are
       valid, and where the invalid ones are.
   * - ``relationships``
     - The relationship topology: the cardinality distribution, the
       cross-filter directions, the orphans and the cycles.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--json``
     - Machine-readable output.
   * - ``--top TOP``
     - ``complexity``: the longest N measures.
   * - ``--group-prefix PREFIX``
     - ``nameof``: the table-name prefix of the x family (default:
       ``x-Selector`` and ``x-Plot Specific``).

``tmdl-preflight format-resolve <path> --name NAME``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Resolve the format a measure name gets from a format calculation group.
The same evaluator backs the S005 rule.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--name NAME``
     - The measure name (``SELECTEDMEASURENAME``).
   * - ``--table TABLE``
     - The calculation-group table (default: ``z-Selector Format``).
   * - ``--axis AXIS``
     - A field-parameter table in scope (default: every axis the group
       branches on, plus none).
   * - ``--label LABEL``
     - The label the field parameter shows (default: the name).
   * - ``--compare NAME2``
     - Resolve a second name on the same axes.
   * - ``--json``
     - Machine-readable output.

``tmdl-preflight grain <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Derive the conformed and blocked dimensions from the relationship graph
(a ``USERELATIONSHIP`` to a calendar counts), and check a grain guard
against them. One row per dimension, then, with ``--check``, the
coverage gaps or the clean line.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--cost-fact COST_FACT``
     - The fact the cost side reads (default: ``Campaigns``).
   * - ``--backend-fact TABLE``
     - A fact of the population side (repeatable; default: ``_Users``
       and ``Transactions``; naming one replaces the default list).
   * - ``--permit TABLE``
     - Treat this dimension as conformed (repeatable).
   * - ``--axis-prefix PREFIX``
     - Field-parameter tables whose ``NAMEOF`` rows expose columns
       (default: ``y-Selector`` and ``y-Plot Specific``).
   * - ``--check``
     - Check the guard; exit 1 on a gap.
   * - ``--guard-table GUARD_TABLE``
     - With ``--check``: the table holding the guard.
   * - ``--guard-measure GUARD_MEASURE``
     - With ``--check``: the guard measure (default: ``-helper Cost
       Grain Valid``; a name starting with ``-`` needs the
       ``--guard-measure=...`` form).
   * - ``--emit-config``
     - Print the blocked/permitted lists as
       ``grain-guard-coverage`` configuration.
   * - ``--json``
     - Machine-readable output.

``tmdl-preflight info --categories <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Decode the encoded category-mapping table. ``groups`` maps each group
id to its categories (many-to-many, with the axis spacers and the
standard calendar groups excluded); ``fact_tables`` maps every fact
table to the categories its metric-axis measures reach.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--categories``
     - Required. The report to produce.
   * - ``--mapping-table MAPPING_TABLE``
     - The mapping table (default: the first table named ``z-Map*``).
   * - ``--json``
     - Machine-readable output.

``tmdl-preflight fieldparam retag <files>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Move named rows from one category to another, across every file given.
The move is all or nothing: a name that is not found live under the old
category in any one of the files stops the whole run and writes nothing.

::

    tmdl-preflight fieldparam retag "y-Selector 2.tmdl" "y-Selector 3.tmdl" \
        --names "Revenue" --from g_a --to g_b

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--names NAMES``
     - Row display name; comma-separated, or repeat the option for a
       name that holds a comma.
   * - ``--from OLD``
     - The current category.
   * - ``--to NEW``
     - The new category.
   * - ``--position POSITION``
     - 1-based row position of the category (default: 4).

``tmdl-preflight manifest <model>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Write a byte-stable JSON manifest of a model: the table roles, the
columns with their warehouse origin through the Power Query renames,
the measures, the warehouse table each partition reads, the
relationships and the domains. The warehouse project and dataset the
model declares are compared with an optional targets file; a mismatch
exits 1 and writes nothing.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--out OUT``
     - Output file (default:
       ``<model>/generated/<name>-manifest.json``).
   * - ``--check``
     - Write nothing; exit 1 when the file is missing or stale.
   * - ``--targets TARGETS``
     - The target contract file (YAML or JSON).

``tmdl-preflight schema-check``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Compare a manifest with a warehouse schema snapshot, or prove view
columns from SQL files. The snapshot is a JSON object of
``{table: [{name, type}, ...]}``.

With ``--manifest`` and ``--schema``, the command reconciles the two:
``missing_in_bq`` fails, ``needs_review`` flags programmatically renamed
tables, and ``not_surfaced_in_pbi`` and suspicious type pairs are
informational. The result is written next to the manifest as
``<name>.schema.json``.

With ``--sql-dir`` and ``--entries``, it proves view-column exposure
from SQL files and exits **3** when an entry is held.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--manifest MANIFEST``
     - A manifest written by ``manifest``.
   * - ``--schema SCHEMA``
     - The schema snapshot to compare with.
   * - ``--out OUT``
     - Output file (default: ``<manifest stem>.schema.json``).
   * - ``--ignore-columns IGNORE_COLUMNS``
     - JSON ``{columns: [...], exceptions: [[table, col]]}``.
   * - ``--strict``
     - Exit 1 on missing columns, needs-review, unreachable tables or a
       coverage gap.
   * - ``--project PROJECT`` / ``--dataset DATASET``
     - The warehouse project and dataset to check against (default: the
       manifest's own target).
   * - ``--sql-dir SQL_DIR``
     - The SQL project root: the view-column exposure mode.
   * - ``--entries ENTRIES``
     - With ``--sql-dir``: a JSON file of
       ``{client, table, view, columns}`` entries.
   * - ``--holdbacks HOLDBACKS``
     - With ``--sql-dir``: the holdbacks file path.
   * - ``--write-holdbacks``
     - With ``--sql-dir``: write the holdbacks file.
   * - ``--clear-ready``
     - With ``--sql-dir``: remove the entries of the clients whose every
       entry is ``READY``.
   * - ``--owner-folder-pattern OWNER_FOLDER_PATTERN``
     - With ``--sql-dir``: a regex whose first group turns an owner id
       into the SQL folder segment that holds its views.

``tmdl-preflight refs <name> [paths ...]``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Count the anchored references to a name, model side and report side,
live and archived. Run it before a rename or a delete, to see the blast
radius.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--files``
     - List the matching files.
   * - ``--include-nonlive`` / ``--include-history``
     - With ``--files``: list the non-live files too.
   * - ``--history GLOB``
     - A glob of archived paths (repeatable).
   * - ``--loose``
     - Count ``NAME]`` and ``NAME"`` (a closing delimiter only;
       over-counts).
   * - ``--targets TARGETS``
     - The target contract file: the live set is the registered model
       paths.

``tmdl-preflight live <model>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The only command that touches the network. It checks the published
dataset over the Power BI REST API, and is opt-in: nothing else in the
tool needs a credential.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--workspace WORKSPACE``
     - Workspace (group) id (default: the ``FABRIC_WORKSPACE_ID``
       environment variable).
   * - ``--dataset DATASET``
     - Dataset id (default: the ``FABRIC_MODEL_ID`` environment
       variable).
   * - ``--checks CHECKS``
     - Comma-separated live check ids (default: all).
   * - ``--json``
     - Machine-readable output.
   * - ``--strict``
     - Exit 1 on warnings too.
   * - ``--record DIR``
     - Write every response to ``DIR``.
   * - ``--replay DIR``
     - Answer from responses recorded in ``DIR`` (no network).
   * - ``--no-coupling-check``
     - Run even when the dataset name does not match the local model.
   * - ``--expect-name EXPECT_NAME``
     - Accept the dataset when this text is part of the workspace name.
   * - ``--non-interactive``
     - Never open a sign-in prompt (service principal only).
   * - ``--cache-ttl CACHE_TTL``
     - Seconds to cache dataset metadata (default: 300).

The check ids are ``live-refresh-status``, ``live-freshness``,
``live-table-sample``, ``live-dimension-rows``, ``live-measure-sample``,
``live-parameter-table-measures``, ``live-uniform-row-limit``,
``live-calendar-rows``, ``live-dimension-multirow``,
``live-calendar-scan``, ``live-calendar-vs-parameters``,
``live-parameter-audit``, ``live-identity``, ``live-filter-compare``,
``live-metadata``, ``live-filter-diagnostic``, ``live-smoke`` and
``live-bpa``. An outcome is pass, warn, fail, skip or error; the
command exits 1 on a fail or an error, and on a warn with ``--strict``.
The dataset name must match the local model folder unless
``--expect-name`` or ``--no-coupling-check`` is given.

``tmdl-preflight rules``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print the rule catalog: id, name, severity, fixability, description.
``--json`` emits it as a JSON array, including ``baselinable``,
``safe``, ``opt_in`` and the rule's ``options``. The top-level flag
``tmdl-preflight --rules`` is an alias.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--json``
     - The catalog as a JSON array.
   * - ``--format {text,json,markdown}``
     - The catalog format. ``markdown`` renders the generated catalog
       page.
   * - ``--out FILE``
     - Write the catalog to ``FILE``.
   * - ``--check``
     - With ``--out``: fail if ``FILE`` is missing or stale; write
       nothing.

The catalog page the tool generates is the reference for every rule:
:doc:`rules`.

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
   * - a ``*.bookmark.json`` file
     - itself (bookmark rules only)
   * - a folder that holds ``*.bookmark.json`` files and names no
       report
     - the bookmark files it holds

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
   * - ``3``
     - ``schema-check`` only: an entry was held in the SQL exposure
       mode

Text output format
----------------------

::

    <file>:<line>  <RULE> <severity>: <message> [<object>] (auto-fixable)

Files are shown relative to the checked path when possible. A run
prints, in this order:

1. the findings, one per line;
2. the ``fixed  <RULE>: <description>`` line of every applied repair
   (``fix`` only), and a ``not fixed  <RULE>: <reason>`` line for every
   fixer a blanket ``fix`` skipped because it is unsafe;
3. a blank line, then the ``scanned:`` line: how many models, tables,
   columns, measures, relationships and DAX expressions the run looked
   at;
4. the console report: the notes a rule printed for itself, as blocks
   headed by the rule id (``DAX Validation Summary:``, ``R001:``,
   ``R002:`` ... ``R014:``);
5. the final ``tmdl-preflight:`` summary line;
6. on a ``fix``, the note the fixer printed for itself, after the
   summary line.

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
        "excluded_files": 0,
        "files_scanned": 12,
        "exempted": 0,
        "baselined": 0
      },
      "fixes": ["M003: Stores.tmdl:3 (table Stores): '...' -> '...'"],
      "skipped": [],
      "skip_reasons": {},
      "skipped_fixers": [],
      "would_fix": [],
      "refused_fixes": [],
      "notes": ["DAX Validation Summary: ..."],
      "fix_notes": [],
      "blocks": [],
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
      "scanned": {"files": 12, "models": 1, "tables": 6, "columns": 22,
                  "measures": 6, "relationships": 4, "dax_blocks": 8,
                  "...": "..."}
    }

.. list-table::
   :header-rows: 1

   * - Key
     - Meaning
   * - ``summary``
     - The counts. ``errors``, ``warnings`` and ``infos`` are the
       findings by severity; ``fixes_applied`` the repairs; ``excluded``
       the excluded violations; ``excluded_files`` the files the
       ``exclude`` globs dropped; ``files_scanned`` the files the run
       looked at; ``exempted`` and ``baselined`` the findings an
       exemption or a baseline accepted.
   * - ``fixes``
     - One string per applied repair.
   * - ``skipped``
     - The rules a policy option turned off for the run.
   * - ``skip_reasons``
     - Why each skipped rule was skipped, keyed by the rule's name.
   * - ``skipped_fixers``
     - The fixers a blanket ``fix`` did not run, because they are
       unsafe. The ids of the rules.
   * - ``would_fix``
     - What a dry run would change.
   * - ``refused_fixes``
     - The repairs that were refused, and why.
   * - ``notes``
     - The console report: the blocks a rule printed for itself, one
       string per block.
   * - ``fix_notes``
     - The notes the fixers printed.
   * - ``blocks``
     - The console blocks a rule appended during the run, in the order
       they were produced.
   * - ``violations``
     - The findings of the final state of the run (post-fix for
       ``fix``).
   * - ``scanned``
     - What the run looked at.

``violations`` reflects the final state of the run (post-fix for
``fix``). ``path`` is project-relative (it starts at the
``*.SemanticModel`` or ``*.Report`` folder), the same identity
baselines and exemptions use. When they apply, the payload also has an
``exemptions`` object (applied, expired, unused) and a ``baseline``
object (suppressed, fixed, stale). The payload grew its top-level keys
in 1.1.0 and again in 1.2.0, without changing the earlier ones.

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
  ``tmdl-preflight fix MyProject/``. A blanket fix runs the twenty-one
  safe fixers and skips the four unsafe ones (D004, F016, P001 and
  R007), naming each on a ``not fixed  <RULE>:`` line. To repair one of
  those too, select it: ``tmdl-preflight fix MyProject/ --select
  R007``. Preview first with ``--dry-run``; every line marked
  ``(auto-fixable)`` under ``check`` is one the subsequent ``fix`` will
  touch. Commit the fixes as their own commit, "apply tmdl-preflight
  auto-fixes," so reviewers can wave it through quickly.
- See what a repair would do before it does it:
  ``tmdl-preflight fix MyProject/ --dry-run --verbose``.

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

Everything the CLI reads about a project comes from the configuration
file (:doc:`configuration`); no environment variable changes a run's
configuration. Two exceptions:

- the pytest fixture honors ``TMDL_PREFLIGHT_AUTOFIX=1`` as a default
  for its ``autofix`` parameter (see :doc:`customization`);
- ``live`` reads ``FABRIC_WORKSPACE_ID`` and ``FABRIC_MODEL_ID`` for
  the workspace and the dataset, and the service-principal credentials
  ``PBI_TENANT_ID``, ``PBI_CLIENT_ID`` and ``PBI_CLIENT_SECRET`` (or
  their ``FABRIC_`` equivalents). It is the only command that needs
  any of them.
