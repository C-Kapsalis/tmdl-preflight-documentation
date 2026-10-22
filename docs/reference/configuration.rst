Configuration reference
===========================

Everything that changes a run without changing the command line: the
configuration file, exemptions, baselines, and the output formats.

The configuration file
--------------------------

``tmdl-preflight`` reads the first of these it finds in the checked
path or one of its parent folders. The search stops at the repository
root, the first folder that contains ``.git``.

1. ``.tmdl-preflight.toml``
2. ``.tmdl-preflight.json``
3. ``pyproject.toml`` with a ``[tool.tmdl-preflight]`` table

``--config FILE`` names a file explicitly; ``--no-config`` ignores any
file. TOML needs Python 3.11 or newer (the standard library's
``tomllib``). On Python 3.10, use ``.tmdl-preflight.json`` with the
same keys; a TOML file found on 3.10 is an error rather than being
skipped.

Every key is validated. An unknown key, an unknown rule id or name, or
an option of the wrong type stops the run with exit code ``2`` and a
message naming the problem; nothing is silently ignored.

An example ``pyproject.toml`` table (it was loaded and validated by
``tmdl-preflight check`` 1.2.1 when this page was written):

.. code-block:: toml

    [tool.tmdl-preflight]
    select = ["M001", "M002", "M003"]      # optional; default: every rule
    ignore = ["S002"]
    strict = false
    exclude = ["**/tables/Legacy*.tmdl"]

    [tool.tmdl-preflight.severity_overrides]
    R007 = "error"

    [tool.tmdl-preflight.rules.S002]
    min_length = 20
    placeholder_words = ["todo", "tbd", "fixme"]

    [tool.tmdl-preflight.rules.D004]
    check_bare_references = true

    [[tool.tmdl-preflight.exemptions]]
    rule = "R004"
    object = "relationship 1f6f2b3a"
    reason = "bidirectional on purpose: product slicers filter stores"
    until = "2026-12-31"

    [tool.tmdl-preflight.profiles.ci]
    strict = true

In ``.tmdl-preflight.toml`` the same keys sit at the top level (or
under ``[tool.tmdl-preflight]``); in ``.tmdl-preflight.json`` they are
the keys of the top-level object.

.. list-table::
   :header-rows: 1

   * - Key
     - Meaning
   * - ``select``
     - Rule ids or names to run. ``--select`` on the command line
       replaces it. The opt-in rules (:doc:`rules`) run only when they
       are named here or on the command line.
   * - ``ignore``
     - Rule ids or names to skip. ``--ignore`` on the command line adds
       to it.
   * - ``strict``
     - ``true`` to fail on warnings as well as errors, like
       ``--strict``.
   * - ``severity_overrides``
     - A table of rule id to ``error``, ``warning`` or ``info``. The
       rule's violations are reported, and gate the exit code, at that
       severity.
   * - ``exclude``
     - Globs of files whose violations are not reported. Files are
       still parsed, so references into them still resolve. A
       configured list replaces the default exclusions (below);
       ``exclude = []`` turns them off.
   * - ``rules.<ID>``
     - Per-rule options. Each rule declares the options it accepts
       (``tmdl-preflight rules --json`` lists them, with their
       defaults); 70 of the 119 rules declare options. Examples: S002
       ``min_length`` and ``placeholder_words``, D004
       ``check_bare_references``, D001 ``min_success_rate``, D002
       ``excluded_tables``, M013 ``fix_mode``, M016 ``required_files``,
       R007 ``fix_autodetected_relationships`` (``true`` lets a blanket
       ``fix`` run R007's unsafe fixer), R011
       ``min_fact_coverage_percent``, every contract rule from C001 to
       C012, and every O rule except O008, O011, O012 and O014.
   * - ``exemptions``
     - A list of documented exceptions, below.
   * - ``profiles.<name>``
     - A table with any of the keys above, deep-merged over the base
       settings when the run passes ``--profile <name>``. Tables
       merge key by key; lists and values replace. ``--profile`` with a
       name that does not exist, or with no configuration file, is an
       error (exit ``2``), never a silent no-op.
   * - ``contracts``, ``refresh_policy``, ``live``, ``advisory``, ``table_roles``, ``measure_classes``, ``bookmarks``, ``manifest``
     - Project sections that a group of rules reads as a whole: the
       declared model contracts, the incremental-refresh settings, the
       live-check options, the advisory list the pytest plugin reads,
       and the shared table-role, measure-class, bookmark and manifest
       settings. Each section is validated by the rules that use it, so
       a mistake in one is reported by the rule that reads it, not by
       the configuration loader.

Paths and globs
~~~~~~~~~~~~~~~~~~~

Excludes, exemptions and baselines all use the same project-relative
path: it starts at the ``*.SemanticModel`` or ``*.Report`` folder, for
example ``BikeShop.SemanticModel/definition/tables/Sales.tmdl``. It
never contains the folder the project is checked out in, so a pattern
cannot accidentally match a parent directory, and the same
configuration works on every machine. Absolute patterns and patterns
containing ``..`` are rejected.

In a glob, ``*`` matches within one folder, ``**`` matches across
folders, and ``?`` matches one character. A pattern without ``/``
matches the file name alone: ``Legacy*.tmdl`` and
``**/tables/Legacy*.tmdl`` are equivalent for table files.

Default exclusions
~~~~~~~~~~~~~~~~~~~~~~

With no ``exclude`` key in the configuration, the tool skips archived
copies: every file whose path below the checked folder has a folder
name containing ``legacy``, ``_backup`` or ``_b2`` (a substring match,
so ``old_legacy_x/`` and ``spinout-05_db_backup/`` are skipped). Only
folders below the checked path count: a project checked out under
``C:\legacy\`` is still scanned, and so is a ``legacy`` folder that you
pass as the path itself.

The text report does not mention skipped files. The JSON report counts
them in ``summary.excluded_files``. To scan everything, set
``exclude = []``; to choose your own globs, set ``exclude`` to them,
which replaces the default.

Exemptions
--------------

An exemption accepts the violations of one rule, optionally narrowed to
a path glob and an exact object, for a stated reason:

.. list-table::
   :header-rows: 1

   * - Key
     - Meaning
   * - ``rule``
     - Required. The rule id or name.
   * - ``reason``
     - Required and non-empty. Why the exception is acceptable, for the
       next reviewer.
   * - ``path``
     - Optional glob over the project-relative path.
   * - ``object``
     - Optional exact object, as printed in brackets in the text
       report (for example ``Sales Measures[Units Sold]``).
   * - ``until``
     - Optional date, ``YYYY-MM-DD``. After that day the exemption no
       longer applies: its violations are reported again, and the
       report lists the expired exemption.

The text report notes how many violations were exempted, every
exemption that has expired, and every exemption that matched nothing
(a sign the problem was fixed and the entry can go).

Baselines
-------------

A baseline lets a team adopt the tool on a model that is not clean yet:
record what fails today, then fail only on what is new.

.. code-block:: text

    tmdl-preflight baseline .
    tmdl-preflight check . --baseline .tmdl-preflight-baseline.json

Run on a copy of the tool's ``examples/bike-shop-broken`` model, the
first command prints:

.. code-block:: text

    tmdl-preflight: baseline .tmdl-preflight-baseline.json: 6 violation(s) recorded
    tmdl-preflight: 1 violation(s) of rules that can never be baselined were not recorded (M003); they will still fail every check

The file records violation identities: rule id, project-relative file,
object, and how many violations share that identity. It has no line
numbers, so editing an unrelated part of a file does not turn a known
violation into a new one. Commit it next to the project.

``check --baseline`` reports only violations beyond the recorded
counts, and lists **fixed** entries (fewer violations than recorded)
and **stale** entries (a rule that no longer exists, or one that can
never be baselined). Shrink the file as violations get fixed:

.. code-block:: text

    tmdl-preflight baseline . --update
    tmdl-preflight baseline . --update --accept-new

``--update`` only ever removes entries or lowers counts. Growing the
baseline takes the explicit ``--accept-new``. The ``baseline`` command
refuses to run:

- with ``--select`` or ``--ignore``, because a partial run would record
  (and, on ``--update``, forget) only some violations;
- while any file fails to parse (M002), because a baseline recorded
  from a partly-read model would hide real violations later;
- when the file already exists and ``--update`` was not given.

Rules whose defects stop a model from opening or deploying are never
recorded and always fail the check: M003, M007, M010, M015, R006 and
R007 (``baselinable = False``). R007 is in that list because a
relationship Desktop invented on its own can silently change results,
so a team has to deal with it rather than accept it.

Output formats
------------------

``--format`` chooses the renderer for ``check`` and ``fix``; the exit
code is the same for every format.

.. list-table::
   :header-rows: 1

   * - Format
     - Use
   * - ``text``
     - The default, for people. One line per violation, then the
       ``scanned:`` line, any notes, and the summary line (the full
       order is in :doc:`usage`).
   * - ``json``
     - For scripts. Described in :doc:`usage`. ``--json`` is an alias.
   * - ``github``
     - GitHub Actions workflow commands
       (``::error file=...,line=...,title=...::message``), which show up
       as annotations on the pull request diff. Info findings become
       notices. File paths are relative to the working directory, so
       run the command from the repository root.
   * - ``sarif``
     - SARIF 2.1.0, for GitHub code scanning and other SARIF viewers.
       One run, with every rule the run used in ``tool.driver.rules``
       and one result per violation; errors, warnings and infos map to
       the SARIF levels ``error``, ``warning`` and ``note``. File URIs
       are relative to the working directory, so run the command from
       the repository root.
   * - ``markdown``
     - A summary, a table of violations and the console notes as a
       list, for pull request comments or a job summary
       (``>> "$GITHUB_STEP_SUMMARY"``).
