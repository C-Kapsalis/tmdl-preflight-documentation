How to modify the rules list
================================

Goal: add a rule of your own, enforcing a convention the built-in
119 rules do not cover, such as "every visible table must carry a
description," on the same engine the built-in rules use. There is no
plugin or entry-point discovery mechanism for this; a custom rule is
added in code, in one of two places:

- **a wrapper script**, which runs the rule through the Python API
  (steps 1 and 2 below). The ``tmdl-preflight`` command and the
  ``tmdl_preflight`` pytest fixture do not see a rule defined this way;
- **your own fork** of the tool, where you register the rule in
  ``ALL_RULES`` (step 4). The command, the configuration file and the
  pytest fixture then treat it like a built-in rule.

You will need working familiarity with Python and, for the pytest
route, with the pytest library itself, plus enough familiarity with
Power BI data models saved as PBIP/TMDL to know what a rule should be
looking at. This page assumes both; it does not teach either. The
concepts here, ``Context``, ``Rule``, ``RuleSet``, and how a rule is
evaluated, are covered in full in :doc:`../reference/customization`,
worth reading first if any of this feels underspecified.

1. Subclass ``Rule``
------------------------

A rule is a class with a few class attributes and a ``check()`` method:

.. code-block:: python

    from tmdl_preflight import Rule, Severity, Context, Violation


    class TableDescriptionRule(Rule):
        id = "Y001"                      # unique; a prefix outside M/D/R/F/B/S/A/C/L/N/O/P/X
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
and hand it to the engine. ``Context`` takes a path (a string or a
``pathlib.Path``) in any of the shapes the command accepts:

.. code-block:: python

    from tmdl_preflight import Context, check, default_ruleset

    ruleset = default_ruleset()
    ruleset.rules.append(TableDescriptionRule())

    report = check(Context("MyProject/"), ruleset)
    for v in report.violations:
        print(v.rule_id, v.message)

``default_ruleset()`` holds the rules a plain run executes: the 119
built-in rules minus the 32 opt-in ones, 87 rules. To work with the
whole catalog, opt-in rules included, use ``full_ruleset()`` from
``tmdl_preflight.rules`` instead.

To run only some rules, narrow the set with ``select()``. It compares
lower-case ids and names, so pass them in lower case:
``ruleset.select(select={"y001", "m003"})``. An upper-case ``"M003"``
matches nothing.

The same ``ruleset`` works with ``fix()`` if your rule also has a
fixer (see :doc:`modify-the-auto-fixes`), and inside a pytest test:

.. code-block:: python

    def test_custom_convention():
        report = check(Context("MyProject/"), ruleset)
        assert not report.errors

3. Test it like the built-ins
----------------------------------

Follow the pattern of the tool's own test suite (``tests/`` in the
repository): copy a clean fixture model into a temporary directory,
mutate it to trigger exactly your failure mode, and assert the
violation is detected. If you
also wrote a fixer, assert that fix, then re-check, comes back clean,
and that running the fixer twice changes nothing further.

4. Register it in a fork
---------------------------

In a fork, import the class in ``src/tmdl_preflight/rules/__init__.py``
and add it to the ``ALL_RULES`` list there (the list holds classes, not
instances). ``tmdl-preflight rules`` then lists it, ``--select Y001``
and the configuration file accept its id and name, and the pytest
fixture runs it. :doc:`../contributing` covers the rest of the steps
for a rule meant for the tool itself.

5. Remove or disable a built-in rule
-----------------------------------------

There is no ``--disable`` flag for permanently retiring a built-in
rule; the registry is the source of truth. To drop one from your fork,
remove its class from ``ALL_RULES`` in
``src/tmdl_preflight/rules/__init__.py``. To skip one for a particular
run without forking anything, use ``--ignore``; to skip it for a whole
repository, list it under ``ignore`` in the configuration file. See
:doc:`../reference/usage` and :doc:`../reference/configuration`.
