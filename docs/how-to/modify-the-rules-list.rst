How to modify the rules list
================================

Goal: add a rule of your own, enforcing a convention the built-in
twenty-one do not cover, such as "every visible table must carry a
description," using the same engine, CLI selection, and pytest fixture
the built-in rules use. There is no plugin or entry-point discovery
mechanism for this; a custom rule is added in code, in your own fork or
a wrapper script.

1. Subclass ``Rule``
------------------------

A rule is a class with four class attributes and a ``check()`` method:

::

    from tmdl_preflight import Rule, Severity, Context, Violation


    class TableDescriptionRule(Rule):
        id = "X001"                      # unique; pick a prefix outside M/D/R/F/B/S
        name = "table-descriptions"
        severity = Severity.WARNING
        description = "Visible tables should carry a /// description comment."

        def check(self, ctx: Context) -> list[Violation]:
            out = []
            for model in ctx.models:                    # parsed SemanticModel(s)
                for table in model.tables.values():
                    if table.is_hidden:
                        continue
                    text = table.file.read_text(encoding="utf-8")
                    if not text.lstrip().startswith("///"):
                        out.append(self.violation(
                            "visible table has no /// description",
                            file=table.file,
                            line=table.line,
                            obj=table.name,
                        ))
            return out

What ``ctx`` gives you:

- ``ctx.models``, a list of parsed ``SemanticModel`` objects (tables,
  columns, measures, relationships, lineage tags, ``dax_blocks()``).
- ``ctx.report_dirs``, every ``*.Report`` folder found, for report-side
  rules.
- ``ctx.reload()``, drops the parse cache; the engine calls it after
  fixes, so you rarely need to call it yourself.

2. Run it
------------

Build a ``RuleSet`` that includes your rule alongside the built-ins,
and hand it to the engine:

::

    from tmdl_preflight import Context, check, default_ruleset

    ruleset = default_ruleset()
    ruleset.rules.append(TableDescriptionRule())

    report = check(Context("MyProject/"), ruleset)
    for v in report.violations:
        print(v.rule_id, v.message)

The same ``ruleset`` works with ``fix()`` if your rule also has a
fixer (see :doc:`modify-the-auto-fixes`), and inside a pytest test:

::

    def test_custom_convention():
        report = check(Context("MyProject/"), ruleset)
        assert not report.errors

3. Test it like the built-ins
----------------------------------

Follow the pattern in the project's own ``tests/test_rules_*.py``: copy
a clean fixture model into a temporary directory, mutate it to trigger
exactly your failure mode, and assert the violation is detected. If you
also wrote a fixer, assert that fix, then re-check, comes back clean,
and that running the fixer twice changes nothing further.

4. Remove or disable a built-in rule
-----------------------------------------

There is no ``--disable`` flag for permanently retiring a built-in
rule; the registry is the source of truth. To drop one from your fork,
remove its instance from ``ALL_RULES`` in
``src/tmdl_preflight/rules/__init__.py``. To skip one only for a
particular run or repository without forking anything, use
``--ignore`` instead; see :doc:`../reference/usage`.
