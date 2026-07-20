How to modify the auto-fixes
================================

Goal: give a rule of your own an auto-fix, or change what one of the
five built-in fixers does. Only five of the twenty-one rules ship a
fixer, and that is deliberate; read the safety contract below before
writing one.

You will need working familiarity with Python and, for the pytest
route, with the pytest library itself, plus enough familiarity with
Power BI data models saved as PBIP/TMDL to judge what a fixer is
allowed to touch. This page assumes both; it does not teach either.
The imposition pattern and the full reasoning behind the safety
contract below are covered in :doc:`../reference/customization`, worth
reading first if any of this feels underspecified.

Where the built-in fixers live
-----------------------------------

.. list-table::
   :header-rows: 1

   * - Rule
     - Class
     - File
   * - M003 lineage-duplicates
     - ``LineageTagUniquenessRule.fix()``
     - ``src/tmdl_preflight/rules/structural.py``
   * - M004 lineage-format
     - ``LineageTagFormatRule.fix()``
     - ``src/tmdl_preflight/rules/structural.py``
   * - M006 model-table-refs
     - ``ModelTableReferencesRule.fix()``
     - ``src/tmdl_preflight/rules/structural.py``
   * - F001 fieldparam-comma-runs
     - ``FieldParameterCommaRunsRule.fix()``
     - ``src/tmdl_preflight/rules/field_parameters.py``
   * - B001 bookmark-int-types
     - ``BookmarkIntegerTypesRule.fix()``
     - ``src/tmdl_preflight/rules/report.py``

To change one's behavior, for example if you want M004's regenerated
tag to come from a seeded generator instead of a random one, edit that
method directly in your fork; there is no configuration hook for
fixer internals.

Add a fixer to a rule of your own
---------------------------------------

Starting from a rule built as in :doc:`modify-the-rules-list`, set
``fixable = True`` and implement ``fix()``:

::

    class TableDescriptionRule(Rule):
        id = "X001"
        name = "table-descriptions"
        severity = Severity.WARNING
        description = "Visible tables should carry a /// description comment."
        fixable = True

        def check(self, ctx: Context) -> list[Violation]:
            ...  # as before

        def fix(self, ctx: Context) -> list[str]:
            applied = []
            for model in ctx.models:
                for table in model.tables.values():
                    if table.is_hidden:
                        continue
                    text = table.file.read_text(encoding="utf-8")
                    if not text.lstrip().startswith("///"):
                        table.file.write_text(
                            f"/// {table.name}\n{text}", encoding="utf-8"
                        )
                        applied.append(f"{table.file.name}: added description stub")
            return applied

Return one human-readable string per change; the engine prints them
and re-checks from disk afterwards. If a fix did not actually resolve
the violation, the re-check says so; you do not need to verify that
yourself, and the run's exit code will still reflect the leftover
problem.

The safety contract
------------------------

A repair earns a fixer only if it satisfies all of the following. Only
five of the twenty-one built-in rules do:

1. **The failure mode fully determines the repair.** No intent needs
   to be inferred. A duplicate tag needs a fresh unique value; an
   orphan comma needs deletion; a string ``"2"`` needs to become the
   integer ``2``.
2. **It never overwrites human-authored semantics.** A fixer touches
   identity metadata, separators, and JSON types, never a DAX
   expression, a name, or a relationship.
3. **It is idempotent.** Running ``fix`` twice produces the same tree
   as running it once.
4. **It is minimal.** Line-level, targeted edits, so the resulting
   diff is reviewable at a glance.

Two of the built-in rules make the boundary concrete by failing it: an
unbalanced DAX expression (D001) has infinitely many "balanced"
completions, so only the author knows which one is meant, and there is
no fixer. A dangling bookmark reference (B002) could mean restore the
visual, retarget the bookmark, or delete the bookmark, three different
intents, so there is no fixer there either.

Test it like the built-ins
-------------------------------

Follow the pattern in the project's own ``tests/test_rules_*.py``:
copy a clean fixture model, break exactly one thing, assert the
violation, then assert that fix, followed by a re-check, comes back
clean, and that running the fixer a second time changes nothing
further.
