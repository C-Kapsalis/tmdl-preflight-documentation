Python API reference
=====================

The plain Python API is a first-class entry point alongside the CLI
and the pytest fixture. It is the same ``check``/``fix`` engine the
other two surfaces call; nothing here has flags or exit codes, only
calls and return values.

``Context``
--------------

::

    from tmdl_preflight.rules.base import Context

    ctx = Context("MyProject/")

The constructor accepts any of four path shapes: a PBIP project root,
a ``*.SemanticModel`` folder, a ``definition`` folder, or a
``*.Report`` folder. Parsing is lazy; the first access to ``ctx.models``
parses every discovered ``definition/`` folder and caches the result.

- ``ctx.reload()``: drops the parse cache. Call this after fixers have
  rewritten files so the next check reads what is actually on disk.
- ``ctx.is_empty()``: ``True`` if no semantic model or report folder
  was found under the given path.

``default_ruleset()``
-------------------------

::

    from tmdl_preflight.rules import default_ruleset

    ruleset = default_ruleset()

Returns the full ``RuleSet`` of built-in rules. Narrow it with
``RuleSet.select(select=None, ignore=None)``, which takes
case-insensitive rule ids or names; the semantics match the CLI's
``--select``/``--ignore`` exactly:

::

    narrowed = ruleset.select(select={"m003", "m004"})

``check(ctx, ruleset)`` and ``fix(ctx, ruleset)``
------------------------------------------------------

Both return a ``PreflightReport``.

::

    from tmdl_preflight.engine import check

    report = check(ctx, ruleset)
    print(report.errors)

``check`` is a pure read-only pass: it runs every rule in the ruleset
and collects violations, mutating nothing.

::

    from tmdl_preflight.engine import fix

    report = fix(ctx, ruleset)
    print(report.fixes_applied)

``fix`` runs the imposition pattern: check, apply every fixable rule's
repair, reload the context from disk, then check again. The returned
report reflects the post-fix state; a violation still present there
survived the repair attempt.

``PreflightReport``
-----------------------

.. list-table::
   :header-rows: 1

   * - Member
     - Meaning
   * - ``violations``
     - The full list of ``Violation`` objects from this run.
   * - ``fixes_applied``
     - Human-readable descriptions of every repair ``fix()`` made.
   * - ``errors`` / ``warnings`` / ``infos``
     - ``violations`` filtered to one severity.
   * - ``exit_code(strict=False)``
     - ``1`` if any error exists (or any warning, when ``strict`` is
       ``True``); ``0`` otherwise. The same logic the CLI uses to pick
       its process exit code.
   * - ``to_dict()``
     - The same structure the CLI's ``--json`` output serializes:
       a ``summary``, the ``fixes`` list, and the ``violations`` list.

``Violation``
----------------

.. list-table::
   :header-rows: 1

   * - Field
     - Meaning
   * - ``rule_id``
     - The rule's id, for example ``"M003"``.
   * - ``severity``
     - A ``Severity`` enum value: ``ERROR``, ``WARNING``, or ``INFO``.
   * - ``message``
     - The human-readable description.
   * - ``file`` / ``line``
     - Where the violation was found, when known.
   * - ``obj``
     - The name of the affected object (a table, a measure, and so
       on), when known.
   * - ``fixable``
     - Whether this violation's rule ships an auto-fixer.

``.location()`` formats ``file`` and ``line`` as one string
(``"path/to/file.tmdl:12"``), the same form the CLI prints.
