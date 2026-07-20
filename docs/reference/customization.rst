Customization reference
===========================

The pytest fixture and the plain Python API, the two entry points
alongside the CLI, and the design the whole rule set is built on: why
it works the way it does, and where the boundaries of automation sit.

The pytest fixture
-----------------------

``tmdl-preflight``'s rules grew out of a pytest suite, and running them
from pytest is still a first-class path. Installing the package
registers the plugin automatically; no ``conftest.py`` wiring is
needed.

::

    def test_model_is_deployable(tmdl_preflight):
        tmdl_preflight.assert_clean("src/Shop.SemanticModel")

On failure, the assertion message carries the full violation list:

::

    Failed: tmdl-preflight found 1 blocking violation(s) in src/Shop.SemanticModel:
      M003 error: lineageTag '...' duplicates the one on table Products (...)

``assert_clean`` fails on errors; pass ``strict=True`` to fail on
warnings too. ``info`` findings never fail a test. Scope it the same
way as the CLI:

::

    def test_lineage_only(tmdl_preflight):
        tmdl_preflight.assert_clean("src/Shop.SemanticModel",
                                    select={"M003", "M004"})

    def test_everything_but_style(tmdl_preflight):
        tmdl_preflight.assert_clean("src/Shop.SemanticModel",
                                    ignore={"S001"})

With ``autofix=True`` the runner repairs what it safely can and
asserts on the re-check, the repair becomes part of the test rather
than a separate chore:

::

    def test_model_is_deployable(tmdl_preflight):
        report = tmdl_preflight.assert_clean("src/Shop.SemanticModel",
                                             autofix=True)
        if report.fixes_applied:
            print("auto-fixed; review and commit:", *report.fixes_applied, sep="\n  ")

Or flip auto-fix from the environment without touching test code,
useful for keeping CI read-only while developers repair locally:

::

    $ TMDL_PREFLIGHT_AUTOFIX=1 pytest tests/test_model_health.py
    $ pytest tests/test_model_health.py

``run()`` returns the report without failing the test, for custom
assertions:

::

    def test_no_new_bidirectional_filters(tmdl_preflight):
        report = tmdl_preflight.run("src/Shop.SemanticModel", select={"R004"})
        assert len(report.infos) <= 2   # the two we have documented

The Python API
------------------

The plain Python API is a first-class entry point alongside the CLI
and the pytest fixture; it is the same ``check``/``fix`` engine both
of them call. Nothing here has flags or exit codes, only calls and
return values, and it is what building or running a custom
:doc:`../how-to/modify-the-rules-list` rests on.

``Context``
~~~~~~~~~~~~~~~

::

    from tmdl_preflight import Context

    ctx = Context("MyProject/")

The constructor accepts any of four path shapes: a PBIP project root,
a ``*.SemanticModel`` folder, a ``definition`` folder, or a
``*.Report`` folder. Parsing is lazy; the first access to
``ctx.models`` parses every discovered ``definition/`` folder and
caches the result.

- ``ctx.reload()``: drops the parse cache. Call this after fixers have
  rewritten files so the next check reads what is actually on disk.
- ``ctx.is_empty()``: ``True`` if no semantic model or report folder
  was found under the given path.

``default_ruleset()``
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    from tmdl_preflight import default_ruleset

    ruleset = default_ruleset()

Returns the full ``RuleSet`` of built-in rules. Narrow it with
``RuleSet.select(select=None, ignore=None)``, which takes
case-insensitive rule ids or names; the semantics match the CLI's
``--select``/``--ignore`` exactly:

::

    narrowed = ruleset.select(select={"m003", "m004"})

``check(ctx, ruleset)`` and ``fix(ctx, ruleset)``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Both return a ``PreflightReport``.

::

    from tmdl_preflight import check

    report = check(ctx, ruleset)
    print(report.errors)

``check`` is a pure read-only pass: it runs every rule in the ruleset
and collects violations, mutating nothing.

::

    from tmdl_preflight import fix

    report = fix(ctx, ruleset)
    print(report.fixes_applied)

``fix`` runs the imposition pattern (below): check, apply every
fixable rule's repair, reload the context from disk, then check again.
The returned report reflects the post-fix state; a violation still
present there survived the repair attempt.

``PreflightReport``
~~~~~~~~~~~~~~~~~~~~~~~~

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
     - The same structure the CLI's ``--json`` output serializes: a
       ``summary``, the ``fixes`` list, and the ``violations`` list.

``Violation``
~~~~~~~~~~~~~~~~~

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

Design: the imposition pattern
------------------------------------

Most linters stop at detection: they report what is wrong and leave
the repair to a human. For violations that are mechanical, where the
correct repair is fully determined by the failure itself,
``tmdl-preflight`` folds the repair into the run instead. ``fix``
executes three phases in strict order:

1. **Check.** Every selected rule runs read-only and reports
   violations.
2. **Fix.** Each fixable rule that reported a violation applies its
   repair, editing files on disk directly; fixers never mutate the
   in-memory model.
3. **Re-check.** The parse cache is dropped and every rule runs again,
   against what is actually on disk now.

The run is clean only if the re-check is clean. A fixer never gets to
vouch for itself: if a repair was incomplete or impossible, the
violation is still on the final report and the exit code is still
nonzero. Auto-repair can therefore never mask a defect; the worst it
can do is fail to remove one, exactly as if it did not exist.

Only five of the twenty-one rules ship a fixer, and that is
deliberate: a repair earns one only if the failure mode fully
determines it, it never overwrites human-authored semantics, it is
idempotent, and it is a minimal, line-level edit. The full contract,
with the two rules that fail it on purpose, is in
:doc:`../how-to/modify-the-auto-fixes`.

Design: the layered test model
------------------------------------

The rule set is organized the way mature BI test suites organize their
checks: in layers, ordered by how early a defect can be caught and how
much infrastructure the check needs.

.. list-table::
   :header-rows: 1

   * - Layer
     - Question
     - Rules
     - Needs
   * - Structural
     - Do the files even parse? Is identity metadata sound? Will
       Power BI open the project at all?
     - M001-M009
     - the files
   * - Expression
     - Is the DAX lexically sound, and are its references real?
     - D001-D003
     - a parsed model
   * - Model integration
     - Do cross-object structures cohere, relationships, field
       parameters?
     - R001-R005, F001
     - a parsed model
   * - Report
     - Does the report layer agree with itself?
     - B001, B002
     - the ``*.Report`` folder
   * - Advisory
     - Is the model following good practice?
     - S001 (and R004/R005 by severity)
     - a parsed model

The ordering is diagnostic, not just taxonomic. A defect at one layer
routinely causes noise at a higher layer: a table file that does not
parse (M002) makes every relationship into it look broken (R001), and
a stray comma in a field-parameter source (F001) makes every
``NAMEOF`` inside it unresolvable (D002). When a run reports
violations at several layers, fix the lowest layer first and re-run;
much of the rest usually evaporates.

All five layers share one property: they run locally, against files,
in seconds, with no credentials. That is the niche this tool occupies,
the checks affordable on every save and required before every push.
Deliberately out of scope, because they need a live service or a
human: deployment state (did the model actually load and refresh),
data quality (did the pipeline fill the tables correctly), and visual
rendering (does the report look right). Passing preflight means a
model deserves the more expensive downstream verification; it does not
replace it.

Severity encodes the layer's authority. Structural and expression
defects fail hard (``error``) because they are facts: the import will
reject a duplicate lineage tag. Advisory rules (``info``) never block,
because they encode judgment, and judgment belongs to the modeler:
three bidirectional relationships might be exactly right for a given
model. ``warning`` sits in between, ``--strict`` promotes it to
blocking, covering things that are almost always wrong but have
legitimate exceptions. This is also why advisory rules never grow
fixers: an auto-fix that "repaired" a bidirectional filter would be
the tool overruling the modeler on a judgment call, exactly what the
imposition pattern's safety contract forbids.

Using the layers in practice: the inner loop while editing can run
just ``--select M002,D001`` for parse and delimiter confidence in well
under a second; a pre-push check runs the full default set; CI runs
the full set with ``--strict``, read-only (see
:doc:`usage`).
