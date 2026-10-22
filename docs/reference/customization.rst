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

A model that is not ready yet can be listed in the configuration's
``advisory`` section, and the plugin then reports a failure on it as an
expected failure instead of a failure. A model that passes stays a
pass, and a setup failure stays a failure.

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

The classes below appear in the order a run actually evaluates them: a
``Context`` is built from a path, a ``RuleSet`` of ``Rule`` instances
(each carrying a ``Severity``) is chosen, ``check()`` or ``fix()``
runs it against the context, and the result comes back as a
``PreflightReport`` full of ``Violation`` objects.

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

.. list-table::
   :header-rows: 1

   * - Member
     - Meaning
   * - ``models``
     - A list of parsed ``SemanticModel`` objects (tables, columns,
       measures, relationships, lineage tags, ``dax_blocks()``);
       parsed on first access, then cached.
   * - ``report_dirs``
     - Every ``*.Report`` folder found under the given path, for
       report-side rules.
   * - ``reload()``
     - Drops the parse cache. Call this after a fixer has rewritten
       files so the next check reads what is actually on disk; the
       engine already does this between the fix and re-check phases.
   * - ``is_empty()``
     - ``True`` if no semantic model or report folder was found under
       the given path.

``Severity``
~~~~~~~~~~~~~~~~

::

    from tmdl_preflight import Severity

An enum with three values, in descending order of how much they block
a run:

.. list-table::
   :header-rows: 1

   * - Value
     - Meaning
   * - ``Severity.ERROR``
     - Will break an import or deploy, or silently corrupt results.
       Always blocks (nonzero exit, failed assertion).
   * - ``Severity.WARNING``
     - Suspicious; deserves a look before shipping. Blocks only under
       ``--strict`` (CLI) or ``strict=True`` (pytest fixture).
   * - ``Severity.INFO``
     - Advisory; encodes a judgment call that belongs to the modeler.
       Never blocks, not even under ``--strict``.

Every ``Rule`` declares one of these as its ``severity`` class
attribute, and every ``Violation`` it produces carries that same
value.

``Rule``
~~~~~~~~~~~~

::

    from tmdl_preflight import Rule

The base class every rule, built-in or custom, subclasses. A handful of
class attributes and one method make a rule:

.. list-table::
   :header-rows: 1

   * - Attribute or method
     - Meaning
   * - ``id``
     - A short unique code, for example ``"M003"``. Built-in ids use a
       letter prefix by layer (M/D/R/F/B/S, and A/C/L/N/O/P/X); pick a
       prefix outside that set for a rule of your own.
   * - ``name``
     - A lowercase, hyphenated name, for example
       ``"lineage-duplicates"``. Accepted anywhere ``id`` is, in
       ``--select``/``--ignore`` and the pytest fixture's
       ``select``/``ignore`` arguments.
   * - ``severity``
     - The ``Severity`` this rule's violations carry by default.
   * - ``description``
     - A one-paragraph, human-readable statement of what the rule
       checks; this is what ``tmdl-preflight rules`` prints.
   * - ``fixable``
     - ``False`` by default. Set ``True`` only alongside a real
       ``fix()`` implementation; see
       :doc:`../how-to/modify-the-auto-fixes`.
   * - ``check(self, ctx)``
     - Required. Reads the ``Context`` and returns a list of
       ``Violation`` objects; never modifies anything.
   * - ``fix(self, ctx)``
     - Optional, only called when ``fixable`` is ``True``. Edits files
       on disk and returns one human-readable description per change.
   * - ``baselinable``
     - ``True`` by default. ``False`` for defects that stop a model
       from opening or deploying; such violations are never written to
       a baseline (see :doc:`configuration`).
   * - ``safe``
     - ``True`` by default. Set ``False`` on a rule whose fixer changes
       data or regenerates content: an unsafe fixer never runs in a
       blanket ``fix``, only when its rule is selected explicitly. Four
       rules ship it: D004, F016, P001 and R007.
   * - ``opt_in``
     - ``False`` by default. Set ``True`` on a rule that encodes one
       team's conventions: it runs only when it is named, with
       ``--select`` or in the configuration's ``select``. Thirty-two
       rules ship it; see :doc:`rules`.
   * - ``option_defaults``
     - A dict of per-rule options and their defaults, for example
       ``{"min_length": 10}``. The configuration file may override
       them under ``rules.<ID>``; ``self.options`` holds the merged
       values.
   * - ``self.violation(...)``
     - A convenience method every rule inherits, building a
       ``Violation`` that is already stamped with this rule's ``id``,
       ``severity``, and ``fixable`` value.

Adding a rule of your own is :doc:`../how-to/modify-the-rules-list`;
adding a fixer to one is :doc:`../how-to/modify-the-auto-fixes`.

``RuleSet``
~~~~~~~~~~~~~~~

::

    from tmdl_preflight import RuleSet

A thin container: one field, ``rules``, a list of ``Rule`` instances,
and one method:

::

    narrowed = ruleset.select(select={"m003", "m004"}, ignore=None)

``select(select=None, ignore=None)`` returns a new ``RuleSet``
narrowed to the given rule ids or names, case-insensitively; the
semantics match the CLI's ``--select``/``--ignore`` exactly, including
that a name matching nothing in the set simply drops out rather than
raising (the CLI's own "unknown rule id" error is a check it performs
itself, before calling ``select()``).

When ``select`` is ``None`` -- that is, when no rule is named -- every
``opt_in`` rule is dropped. A convention rule therefore never runs
unless it is asked for by name. The same happens in the CLI, which is
why the catalog marks the opt-in rules: a rule you have never heard of
may or may not be running.

``default_ruleset()``
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    from tmdl_preflight import default_ruleset

    ruleset = default_ruleset()

Returns the ``RuleSet`` a plain run executes: every one of the 119
built-in rules **except** the 32 opt-in ones, in the order
``tmdl-preflight rules`` prints them. Narrow it with
``RuleSet.select()``, above, or extend it by appending a rule of your
own to ``ruleset.rules``.

For the whole catalog, opt-in rules included, use ``full_ruleset()``:

::

    from tmdl_preflight.rules import ALL_RULES, full_ruleset

    every_rule = full_ruleset()          # 119 rules
    assert len(ALL_RULES) == 119

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
   * - ``scanned``
     - Counts of what the run looked at (tables, measures,
       relationships, DAX blocks, and so on).
   * - ``to_dict()``
     - The same structure the CLI's ``--json`` output serializes: a
       ``summary``, the ``fixes``, ``skipped``, ``skip_reasons``,
       ``skipped_fixers``, ``would_fix``, ``refused_fixes``, ``notes``,
       ``fix_notes`` and ``blocks`` lists, the ``violations`` list, and
       ``scanned``.

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

What the package exports
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``tmdl_preflight`` exports the names above, and these:

.. list-table::
   :header-rows: 1

   * - Name
     - Meaning
   * - ``ALL_RULES``
     - The list of every registered rule class, opt-in ones included
       (119).
   * - ``full_ruleset()``
     - A ``RuleSet`` of all 119 rules.
   * - ``parse_model(path)``
     - Parse one model folder into a ``SemanticModel``.
   * - ``apply_policy(...)``
     - The policy a configuration file applies to a run: the selection,
       the excludes, the exemptions, the baseline.

Two helper modules are part of the API as well, and the rules use them
too:

.. list-table::
   :header-rows: 1

   * - Name
     - Meaning
   * - ``tmdl_preflight.measure_classes.classify_measure_name(name)``
     - Classify a measure name: is it for report authors, or plumbing?
       Returns the class and the helper subcategory.
   * - ``tmdl_preflight.roles.fact_tables(model)``
     - The shared table-role law: which tables are facts, dimensions,
       selectors or measure tables.

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

Twenty-five of the 119 rules ship a fixer, and that is deliberate: a
repair earns one only if the failure mode fully determines it, it never
overwrites human-authored semantics, it is idempotent, and it is a
minimal, line-level edit. Four of the twenty-five are marked **unsafe**
(D004, F016, P001 and R007): their repair changes data or regenerates
content, so they never run in a blanket ``fix`` and only run when their
rule is selected explicitly. Every fixer also proves its edit before
writing it (``edits.verified_write``), so a repair that cannot be shown
to change only what it claims is never written. The full contract,
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
     - M001-M023
     - the files
   * - Expression
     - Is the DAX lexically sound, and are its references real?
     - D001-D012, X001
     - a parsed model
   * - Model integration
     - Do cross-object structures cohere: relationships, the calendar,
       field parameters, the grain guards?
     - R001-R016, F001-F017, C001-C012
     - a parsed model
   * - Report
     - Does the report layer agree with itself, and is it complete
       enough for Fabric to import?
     - B001-B008
     - the ``*.Report`` folder
   * - Security
     - Do the roles and their row filters do what they claim?
     - L001-L006
     - the ``*.SemanticModel`` folder
   * - Governance
     - Is the model following the conventions, and is its refresh
       policy sound?
     - S001-S005, O001-O015, N020, A001-A005, P001-P002
     - a parsed model, and a configuration for the opt-in ones
   * - Advisory
     - Is the model following good practice?
     - the ``info`` rules of every layer (S001, S002, R004, R005,
       A002, L006, and the inventory rules)
     - a parsed model

The ordering is diagnostic, not just taxonomic. A defect at one layer
routinely causes noise at a higher layer: a table file that does not
parse (M002) makes every relationship into it look broken (R001), and
a stray comma in a field-parameter source (F001) makes every
``NAMEOF`` inside it unresolvable (D002). When a run reports
violations at several layers, fix the lowest layer first and re-run;
much of the rest usually evaporates.

All these layers share one property: they run locally, against files,
in seconds, with no credentials. That is the niche this tool occupies,
the checks affordable on every save and required before every push.
Deliberately out of scope, because they need a live service or a
human: deployment state (did the model actually load and refresh),
data quality (did the pipeline fill the tables correctly), and visual
rendering (does the report look right). Passing preflight means a
model deserves the more expensive downstream verification; it does not
replace it. The one command that does reach a service, ``live``, is
opt-in and never runs by default.

Severity encodes the layer's authority. Structural and expression
defects fail hard (``error``) because they are facts: the import will
reject a duplicate lineage tag. Advisory rules (``info``) never block,
because they encode judgment, and judgment belongs to the modeler:
three bidirectional relationships might be exactly right for a given
model. ``warning`` sits in between, ``--strict`` promotes it to
blocking, covering things that are almost always wrong but have
legitimate exceptions.

A judgment call does not earn a fixer, but a determined defect does,
whatever layer it sits in. R004 ``relationship-bidirectional`` still has
no fixer, because an auto-fix that "repaired" a bidirectional filter
would be the tool overruling the modeler. The rules that do have
fixers repair what the failure itself determines: a duplicated lineage
tag (M003), an orphan comma (F001), a missing ``ref table`` line
(M006), a hidden helper measure (O009), a ragged field-parameter row
(F004). The four whose repair changes data or regenerates content
(D004, F016, P001, R007) are marked unsafe and run only when their
rule is named.

Using the layers in practice: the inner loop while editing can run
just ``--select M002,D001`` for parse and delimiter confidence in well
under a second; a pre-push check runs the full default set; CI runs
the full set with ``--strict``, read-only (see
:doc:`usage`).
