Usage reference
===================

Everything the ``tmdl-preflight`` command accepts, plus the usage
patterns that come up most: CI, scoping a run, and consuming
machine-readable output. The rule catalog has its own page,
:doc:`rules`. This page describes version 1.2.1.

Commands on this page are written once when they are the same in
PowerShell and bash. Where they differ, both forms are shown. The
PowerShell forms were not run when this page was written.

Installing
-------------

``tmdl-preflight`` needs Python 3.10 or later and has no required
runtime dependency. Install it from the release tag with pip, into a
virtual environment:

.. code-block:: text

    pip install "tmdl-preflight @ git+https://github.com/C-Kapsalis/tmdl-preflight@v1.2.1"
    tmdl-preflight --version

.. code-block:: text

    tmdl-preflight 1.2.1

Or run it without installing, with `uv <https://docs.astral.sh/uv/>`_:

.. code-block:: text

    uvx --from git+https://github.com/C-Kapsalis/tmdl-preflight@v1.2.1 tmdl-preflight --version

To work from a clone of the repository (the examples and the test
suite live there), see :doc:`../tutorials/getting-started`.

**PyYAML for YAML targets files.** ``manifest`` and ``refs`` read a
targets file with ``--targets``. A ``.json`` targets file always
works. A YAML targets file needs PyYAML, which is not installed with
the package (the ``[test]`` extra of a clone install includes it).
Install it next to the tool:

.. code-block:: text

    pip install PyYAML

or, with uv:

.. code-block:: text

    uvx --with PyYAML --from git+https://github.com/C-Kapsalis/tmdl-preflight@v1.2.1 tmdl-preflight manifest <model> --targets targets.yml

Without PyYAML, a YAML targets file is ignored and the tool prints no
message about it: the run behaves as if ``--targets`` was not given.
The same happens when the targets file does not exist or does not
parse. So use a ``.json`` targets file, or install PyYAML, and check
the path you pass.

Command syntax
-----------------

.. code-block:: text

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

Every command prints its own help with ``--help``, for example
``tmdl-preflight fix --help``.

``tmdl-preflight check <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule and report violations. Strictly read-only;
never modifies a file.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--select IDS``
     - Comma-separated rule ids or names to run (default: every rule
       that is not opt-in, or the configuration's ``select``).
       Case-insensitive.
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
       print what was captured. Two rules record a floor this way: R008
       ``fact-conformed-dimension-join`` and R016
       ``dead-anchor-calendar``. A selection with neither of them is a
       usage error (exit ``2``). The O007 floor comes from
       ``dead-facts --capture`` or ``baseline --capture-floors O007``.
   * - ``--config FILE``
     - Read the configuration from ``FILE`` instead of searching for
       one.
   * - ``--profile NAME``
     - Merge the configuration's ``profiles.NAME`` table. A name that
       does not exist, or ``--profile`` with no configuration file, is
       an error (exit ``2``).
   * - ``--no-config``
     - Ignore any configuration file.

When ``--select`` names M003, M011, M013 or M017, the run also prints
that rule's own console report before the findings. For M003 and M011
under ``check``, that report reads as if it had repaired the files
("Mode: LIVE", "Fixed ..."), but ``check`` writes nothing. Use
``fix --dry-run`` to see what a repair would change.

``tmdl-preflight fix <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run every selected rule; for each fixable rule that reported
violations, apply its auto-fix; then re-run every selected rule from
disk and report what remains. ``fix`` edits files in place: run it on a
clean Git working tree, or on a copy, so the change can be reviewed and
undone.

The exit code reflects the post-fix state: ``0`` means the re-check
came back clean, ``1`` means violations survived the repair, or were
never fixable in the first place, and ``2`` means a fixer refused a
repair it could not prove (the file is left unchanged and listed under
``refused_fixes``; the other files are still repaired).

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
       fixers would act on (capped at 255). A blanket ``fix --check``
       does not count the objects of the unsafe fixers.
   * - ``--only {missing,invalid}``
     - Narrow the refresh-policy fixers: only add missing policies, or
       only realign invalid ones.
   * - ``--no-prune``
     - The refresh-policy fixer keeps the backlog entries of the tables
       it repaired.

**Unsafe fixers.** Four fixers are marked unsafe, because their repair
changes data or regenerates content: D004, F016, P001 and R007. A
blanket ``fix`` (one that does not name the rule in ``--select``)
never runs them. When the check reported findings that a skipped
unsafe fixer repairs, the run says so on a line like this one, and the
JSON payload lists the rule under ``skipped_fixers``:

.. code-block:: text

    not fixed  R007: its fixer can change data, so a blanket fix skips it; run 'fix --select R007' to apply it

P001 is named for P001 findings and also for P002 findings, because
its fixer adds the refresh policies P002 reports missing. A run with no
such finding prints no ``not fixed`` line. (In 1.2.0, every blanket fix
printed the R007 line, whether or not R007 had found anything.)

To run an unsafe fixer, select its rule:

.. code-block:: text

    tmdl-preflight fix <path> --select R007

R007 can also be opted into a blanket fix with its option
``fix_autodetected_relationships = true`` (see :doc:`configuration`).

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
     - The file to write (default:
       ``<path>/.tmdl-preflight-baseline.json``; with
       ``--capture-floors``, ``<path>/<rule>_floors.json``).
   * - ``--update``
     - Shrink an existing baseline to what still fails; never adds
       entries.
   * - ``--accept-new``
     - With ``--update``, also add today's new violations.
   * - ``--capture-floors RULE``
     - Record the floor file of a ratcheted rule (O007, R008 or R016)
       from today's findings and write it out. Without ``--output``
       the file is ``<path>/o007_floors.json``, ``r008_floors.json`` or
       ``r016_floors.json``.

A floor file written by ``baseline --capture-floors`` or by
``dead-facts --capture`` is JSON with sorted keys and LF line endings,
on Windows too (1.2.1; 1.2.0 wrote CRLF on Windows).

``tmdl-preflight stats <path>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print an inventory of the project: tables, columns, measures,
relationships, partitions by kind, calculation groups and items, field
parameters, descriptions, display folders, report pages, visuals and
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

List the fact tables no measure reads, one ``<model>: <table>`` line
each; a model with none prints nothing. A fact table that only filter
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
   * - ``--config FILE`` / ``--profile NAME`` / ``--no-config``
     - As for ``check``.

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
       drops the all-zero entries. Needs ``--floors``; when the file
       does not exist yet, it is seeded from the current scan.
   * - ``--config FILE`` / ``--profile NAME`` / ``--no-config``
     - As for ``check``.

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
The same evaluator backs the S005 rule. When the model has no
calculation-group table of that name, the command prints nothing and
exits ``0``.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--name NAME``
     - Required. The measure name (``SELECTEDMEASURENAME``).
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
     - Check the guard; exit 1 on a gap. Needs ``--guard-table``:
       without it the command prints ``--check needs --guard-table``
       and exits ``1``.
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
The command edits the files in place.

PowerShell (the backtick continues the line):

.. code-block:: powershell

    tmdl-preflight fieldparam retag "y-Selector 2.tmdl" "y-Selector 3.tmdl" `
        --names "Revenue" --from g_a --to g_b

bash:

.. code-block:: bash

    tmdl-preflight fieldparam retag "y-Selector 2.tmdl" "y-Selector 3.tmdl" \
        --names "Revenue" --from g_a --to g_b

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--names NAMES``
     - Required. Row display name; comma-separated, or repeat the
       option for a name that holds a comma.
   * - ``--from OLD``
     - Required. The current category.
   * - ``--to NEW``
     - Required. The new category.
   * - ``--position POSITION``
     - 1-based row position of the category (default: 4).

``tmdl-preflight manifest <model>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Write a byte-stable JSON manifest of a model: the table roles, the
columns with their warehouse origin through the Power Query renames,
the measures, the warehouse table each partition reads, the
relationships and the domains. ``<model>`` is a ``*.SemanticModel``
folder, or an entry key of the targets file. The warehouse project and
dataset the model declares are compared with the targets file when one
is given; a mismatch exits 1 and writes nothing.

The manifest lists the tables in the same order on every OS (1.2.1):
1.2.0 sorted them differently on Windows, so a manifest committed from
one OS read as stale under ``--check`` on the other. Regenerate a
manifest written by 1.2.0 on Windows once.

.. list-table::
   :header-rows: 1

   * - Option
     - Effect
   * - ``--out OUT``
     - Output file (default:
       ``<model>/generated/<name>-manifest.json``).
   * - ``--check``
     - Write nothing; exit 1 when the file is missing (``MISSING:``)
       or stale (``DRIFT:``), 0 with ``OK: ... is up to date.``
   * - ``--targets TARGETS``
     - The target contract file, JSON or YAML (YAML needs PyYAML; see
       `Installing`_).
   * - ``--config FILE`` / ``--profile NAME`` / ``--no-config``
     - As for ``check``.

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
radius. ``<name>`` is exact and case-sensitive; the paths to scan
default to the current folder.

.. code-block:: text

    tmdl-preflight refs Revenue examples/bike-shop-clean --files

.. code-block:: text


    References to "Revenue"  (matched as `[Revenue]`, `"Revenue"` or `"Table.Revenue"`)

      LIVE      model     4   report     0   total     4   files 2
      non-live  model     0   report     0   total     0   files 0

      Live blast radius: 4 reference(s).

         2  live     model  BikeShop.SemanticModel/definition/tables/Metric Selector.tmdl
         2  live     model  BikeShop.SemanticModel/definition/tables/Sales Measures.tmdl

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
       paths. JSON or YAML (YAML needs PyYAML; see `Installing`_).

``tmdl-preflight live <model>``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The only command that touches the network. It checks the published
dataset over the Power BI REST API, and is opt-in: nothing else in the
tool needs a credential. ``<model>`` is the local ``*.SemanticModel``
folder.

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
   * - ``--config FILE`` / ``--profile NAME`` / ``--no-config``
     - As for ``check``.

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
``--expect-name`` or ``--no-coupling-check`` is given. With no
workspace or dataset id (neither the option nor the environment
variable), the command exits ``2``.

Set the environment variables for one session like this.

PowerShell:

.. code-block:: powershell

    $env:FABRIC_WORKSPACE_ID = "<workspace id>"
    $env:FABRIC_MODEL_ID = "<dataset id>"

bash:

.. code-block:: bash

    export FABRIC_WORKSPACE_ID="<workspace id>"
    export FABRIC_MODEL_ID="<dataset id>"

``tmdl-preflight rules``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Print the rule catalog: id, name, severity, ``auto-fixable`` or
``check-only``, the description, and ``(opt-in)`` on the rules that
run only when named. ``--json`` emits it as a JSON array, including
``baselinable``, ``safe``, ``opt_in`` and the rule's ``options``. The
top-level flag ``tmdl-preflight --rules`` is an alias.

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
     - Write the catalog to ``FILE`` (``--format markdown`` only).
   * - ``--check``
     - With ``--out``: fail (exit ``1``) if ``FILE`` is missing or
       stale; write nothing.

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
run. Archived copies under the root are skipped by default; see
:doc:`configuration` (``exclude``).

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
       or malformed, or ``baseline`` refused to write. Also ``fix``
       when a fixer refused a repair it could not prove.
   * - ``3``
     - ``schema-check`` only: an entry was held in the SQL exposure
       mode

``fix --check`` is the exception: it exits with the number of objects
the fixers would act on.

Text output format
----------------------

.. code-block:: text

    <file>:<line>  <RULE> <severity>: <message> [<object>] (auto-fixable)

Files are shown relative to the checked path when possible, with the
path separator of the OS (backslashes on Windows). A run prints, in
this order:

1. on a ``fix``, the notes the fixers printed while they worked (for
   example R007's ``Found 1 autodetected relationships:``), then a
   blank line;
2. on a ``fix``, the ``fixed  <RULE>: <description>`` line of every
   applied repair, a ``would fix  <RULE>:`` line per rule for a dry
   run, a ``refused  ...`` line for every repair a fixer could not
   prove, and a ``not fixed  <RULE>:`` line for every unsafe fixer the
   run skipped while its findings were reported; then a blank line;
3. the findings, one per line (a rule may print a message that runs
   over more than one line);
4. a blank line, then the ``scanned:`` line: how many models, tables,
   columns, measures, relationships, DAX expressions, reports and
   bookmarks the run looked at;
5. the console report and the notes: the blocks some rules print for
   themselves (``DAX Validation Summary:``, ``R001:``, ``R002:`` ...
   ``R014:``), then the notes on skipped rules, exclusions, exemptions
   and the baseline;
6. the final ``tmdl-preflight:`` summary line;
7. on a ``fix``, the closing advice a fixer printed, after the summary
   line.

JSON output format
-----------------------

This is the real output of
``tmdl-preflight check examples/bike-shop-broken --select M003 --json``,
run in a clone of the tool's repository:

.. code-block:: json

    {
      "summary": {
        "errors": 1,
        "warnings": 0,
        "infos": 0,
        "fixes_applied": 0,
        "excluded": 0,
        "excluded_files": 0,
        "files_scanned": 15,
        "exempted": 0,
        "baselined": 0
      },
      "fixes": [],
      "skipped": [],
      "skip_reasons": {},
      "skipped_fixers": [],
      "would_fix": [],
      "refused_fixes": [],
      "notes": [],
      "fix_notes": [],
      "blocks": [],
      "violations": [
        {
          "rule": "M003",
          "severity": "error",
          "message": "Duplicate lineageTag 'e689596a-59ea-4b2c-a14d-48596a7b8c9d' found in files: Products.tmdl, Stores.tmdl. The tag on line 3 (table Products) is the one that keeps it; the copy on line 3 (table Stores) is regenerated by the auto-fix.",
          "file": "examples/bike-shop-broken/BikeShop.SemanticModel/definition/tables/Stores.tmdl",
          "line": 3,
          "object": "table Stores",
          "fixable": true,
          "path": "BikeShop.SemanticModel/definition/tables/Stores.tmdl"
        }
      ],
      "scanned": {
        "files": 15,
        "models": 1,
        "tables": 6,
        "columns": 22,
        "measures": 6,
        "relationships": 4,
        "dax_blocks": 8,
        "calculation_groups": 0,
        "expressions": 0,
        "tmdl_files": 9,
        "reports": 1,
        "pages": 1,
        "visuals": 0,
        "bookmarks": 0
      }
    }

.. list-table::
   :header-rows: 1

   * - Key
     - Meaning
   * - ``summary``
     - The counts. ``errors``, ``warnings`` and ``infos`` are the
       findings by severity; ``fixes_applied`` the repairs; ``excluded``
       the excluded violations plus the excluded files;
       ``excluded_files`` the files the ``exclude`` globs (or the
       default exclusions) dropped; ``files_scanned`` the files the run
       looked at; ``exempted`` and ``baselined`` the findings an
       exemption or a baseline accepted.
   * - ``fixes``
     - One string per applied repair.
   * - ``skipped``
     - The rules a policy option turned off for the run.
   * - ``skip_reasons``
     - Why each skipped rule was skipped, keyed by the rule's name.
   * - ``skipped_fixers``
     - The ids of the unsafe fixers a blanket ``fix`` did not run while
       their findings were reported.
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
     - The closing advice the fixers printed, in the order it was
       produced.
   * - ``violations``
     - The findings of the final state of the run (post-fix for
       ``fix``).
   * - ``scanned``
     - What the run looked at.

``file`` is the path as the run saw it; ``path`` is project-relative
(it starts at the ``*.SemanticModel`` or ``*.Report`` folder, with
forward slashes on every OS), the same identity baselines and
exemptions use. When they apply, the payload also has an
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

- Check one model in a multi-model repository: pass the specific
  folder, ``tmdl-preflight check src/Finance.SemanticModel``.
- Silence an advisory rule that does not apply to your architecture,
  for example S001 in a model that formats via calculation groups:
  ``tmdl-preflight check . --ignore S001``, or ``ignore = ["S001"]`` in
  the configuration file to make it permanent.
- Gate on a subset while cleaning up legacy debt:
  ``tmdl-preflight check . --select M002,M003,D001,R001``, or record a
  baseline and fail only on new violations (see :doc:`configuration`).
- Apply every safe repair in one pass, before cutting a release branch:
  ``tmdl-preflight fix MyProject/``. A blanket fix runs the twenty-one
  safe fixers and skips the four unsafe ones (D004, F016, P001 and
  R007). Preview first with ``--dry-run``; every line marked
  ``(auto-fixable)`` under ``check`` belongs to a rule with a fixer,
  and the ``would fix`` lines say what a real run would repair. Commit
  the fixes as their own commit, "apply tmdl-preflight auto-fixes," so
  reviewers can wave it through quickly.
- See what a repair would do before it does it:
  ``tmdl-preflight fix MyProject/ --dry-run --verbose``.

An unknown id or name in ``--select``/``--ignore`` is a usage error
(exit 2), not a silently smaller selection; run ``tmdl-preflight
rules`` to see the valid ids and names.

Consuming JSON output
~~~~~~~~~~~~~~~~~~~~~~~~~~

For annotations or dashboards, use ``--json`` and parse the payload:

.. code-block:: text

    tmdl-preflight check . --json > preflight.json

In Windows PowerShell 5.1, ``>`` writes UTF-16; to get a UTF-8 file
there, pipe to ``Out-File -Encoding utf8`` instead. PowerShell 7 writes
UTF-8 with ``>``.

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
