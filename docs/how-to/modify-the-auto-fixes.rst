How to modify the auto-fixes
================================

Goal: give a rule of your own an auto-fix, or change what one of the
twenty-five built-in fixers does. Only twenty-five of the 119 rules
ship a fixer, and that is deliberate; read the safety contract below
before writing one.

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
   * - M011 property-after-child-block
     - ``PropertyAfterChildBlockRule.fix()``
     - ``src/tmdl_preflight/rules/structural.py``
   * - M013 dangling-description
     - ``DanglingDescriptionRule.fix()``
     - ``src/tmdl_preflight/rules/structural.py``
   * - M019 undefined-query-group
     - ``UndefinedQueryGroupRule.fix()``
     - ``src/tmdl_preflight/rules/hygiene.py``
   * - M021 culture-orphan-bindings
     - ``CultureOrphanBindingsRule.fix()``
     - ``src/tmdl_preflight/rules/cultures.py``
   * - M022 diagram-layout-blank
     - ``DiagramLayoutBlankRule.fix()``
     - ``src/tmdl_preflight/rules/hygiene.py``
   * - D004 dax-reference-resolution
     - ``DaxReferenceResolutionRule.fix()``
     - ``src/tmdl_preflight/rules/dax_rules.py``
   * - R007 autodetected-relationships
     - ``AutodetectedRelationshipsRule.fix()``
     - ``src/tmdl_preflight/rules/relationships.py``
   * - F001 fieldparam-comma-runs
     - ``FieldParameterCommaRunsRule.fix()``
     - ``src/tmdl_preflight/rules/fp_rows.py``
   * - F002 fieldparam-missing-separator
     - ``MissingRowSeparatorRule.fix()``
     - ``src/tmdl_preflight/rules/field_parameters.py``
   * - F003 fieldparam-trailing-comma
     - ``TrailingRowCommaRule.fix()``
     - ``src/tmdl_preflight/rules/field_parameters.py``
   * - F004 fieldparam-ragged-rows
     - ``RaggedRowWidthRule.fix()``
     - ``src/tmdl_preflight/rules/fp_rows.py``
   * - F006 fieldparam-duplicate-rows
     - ``FieldParameterDuplicateRowsRule.fix()``
     - ``src/tmdl_preflight/rules/fp_rows.py``
   * - F007 fieldparam-family-consistency
     - ``FieldParameterFamilyConsistencyRule.fix()``
     - ``src/tmdl_preflight/rules/fp_rows.py``
   * - F014 selector-branch-resolution
     - ``SelectorBranchResolutionRule.fix()``
     - ``src/tmdl_preflight/rules/fp_axes.py``
   * - F016 fieldparam-double-slash-rows
     - ``FieldParameterDoubleSlashRowsRule.fix()``
     - ``src/tmdl_preflight/rules/fp_layout.py``
   * - F017 fieldparam-blank-lines
     - ``FieldParameterBlankLinesRule.fix()``
     - ``src/tmdl_preflight/rules/fp_layout.py``
   * - B001 bookmark-int-types
     - ``BookmarkIntegerTypesRule.fix()``
     - ``src/tmdl_preflight/rules/report.py``
   * - B003 bookmark-options-schema
     - ``BookmarkOptionsSchemaRule.fix()``
     - ``src/tmdl_preflight/rules/report.py``
   * - B007 report-format-schema
     - ``ReportFormatSchemaRule.fix()``
     - ``src/tmdl_preflight/rules/report_format.py``
   * - A003 mapping-category-required-keys
     - ``MappingCategoryRequiredKeysRule.fix()``
     - ``src/tmdl_preflight/rules/mapping.py``
   * - O009 helper-measures-hidden
     - ``HelperMeasuresHiddenRule.fix()``
     - ``src/tmdl_preflight/rules/measure_hygiene.py``
   * - P001 refresh-policy-repair
     - ``RefreshPolicyRepairRule.fix()``
     - ``src/tmdl_preflight/rules/refresh_policy.py``

Four of the twenty-five are **unsafe**: D004, F016, P001 and R007.
Their repair changes data or regenerates content, so they never run in
a blanket ``fix`` and run only when their rule is selected explicitly
(R007 also runs when its option ``fix_autodetected_relationships`` is
true). A rule marks itself unsafe with the ``safe = False`` class
attribute; ``tmdl-preflight rules --json`` reports ``"safe": false``
for it. When a blanket ``fix`` skips an unsafe fixer while the check
reported findings it repairs, the run names it on a ``not fixed
<RULE>:`` line; a fixer that also repairs another rule's findings says
so by overriding ``repaired_rule_ids()`` (P001 returns P001 and P002).

The built-in fixers share two habits. A rule's ``check()`` and
``fix()`` call the same recognizer (in ``recognizers.py``, ``dax.py``
or the rule's own module), so they agree on what a defect is. And
every fixer except M022, which replaces a whole ``diagramLayout.json``
with the blank layout, writes through
``edits.verified_write(original, new_text, verify)``: the
``verify`` callback must prove the edit (for JSON, by parsing both
versions and comparing) or raise ``EditRefused``, in which case
nothing is written. ``verified_write`` also refuses a file that mixes
its line endings, and a file that changed on disk since it was read.
``read_text_file`` notes the file's newline style and byte-order mark,
and ``verified_write`` writes them back unchanged.

A few fixers read options from the configuration file: M013
``fix_mode`` (``"delete"`` or ``"attach"``), R007
``fix_autodetected_relationships`` and D004 ``fix_enumeration_lines``.
Beyond those options, the configuration can select, ignore or re-grade
rules, but it has no hook for fixer internals. To change one's
behavior, for example if you want M004's regenerated tag to come from a
seeded generator instead of a random one, edit that method directly in
your fork.

Add a fixer to a rule of your own
---------------------------------------

Starting from a rule built as in :doc:`modify-the-rules-list`, set
``fixable = True`` and implement ``fix()``:

.. code-block:: python

    from tmdl_preflight import Context, Rule, Severity, Violation
    from tmdl_preflight.edits import EditRefused, read_text_file, verified_write


    class TableDescriptionRule(Rule):
        id = "Y001"
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
                    original = read_text_file(table.file)
                    if original.text.lstrip().startswith("///"):
                        continue
                    new_text = f"/// {table.name}{original.newline}{original.text}"

                    def verify(old, new):
                        if not new.endswith(old):
                            raise EditRefused("only a prefix may be added")

                    try:
                        if verified_write(original, new_text, verify):
                            applied.append(f"{table.file.name}: added description stub")
                    except EditRefused as exc:
                        ctx.refused_fixes.append(f"{self.id}: {table.file.name}: {exc}")
            return applied

Return one human-readable string per change; the engine prints them
and re-checks from disk afterwards. Catch ``EditRefused`` yourself:
the engine does not, so an uncaught one stops the whole run. Recording
it in ``ctx.refused_fixes``, as the built-in fixers do, makes the CLI
list it on a ``refused`` line and exit ``2``. If a fix did not actually resolve
the violation, the re-check says so; you do not need to verify that
yourself, and the run's exit code will still reflect the leftover
problem.

The safety contract
------------------------

A repair earns a fixer only if it satisfies all of the following. Only
twenty-five of the 119 built-in rules have one:

1. **The failure mode fully determines the repair.** No intent needs
   to be inferred. A duplicate tag needs a fresh unique value; an
   orphan comma needs deletion; a string ``"2"`` needs to become the
   integer ``2``.
2. **It never overwrites human-authored semantics.** Most fixers touch
   identity metadata, separators, layout and JSON types. The few safe
   fixers that remove or copy content do so only where the failure
   determines the result: F006 removes an exact duplicate row, F007
   copies a row the rest of its family already offers and annotates
   the edit for review, and F014 deletes a selector branch that calls
   a measure that no longer exists. A fixer never chooses a new name,
   expression or relationship for you.
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

A fifth condition applies to the four unsafe fixers: a repair that
changes data or regenerates content is not safe to run without being
asked for. D004 deletes whole generated ``ISFILTERED(...)`` lines, F016
deletes rows from a picker a user sees, P001 copies the partition
source into a drifted ``sourceExpression`` and adds missing refresh
policies (rewriting a loader call from a configured template, after
which a full refresh is required), and R007 deletes relationship
blocks. Each one is marked ``safe = False`` and
runs only when its rule is selected explicitly.

Test it like the built-ins
-------------------------------

Follow the pattern of the tool's own test suite (``tests/`` in the
repository): copy a clean fixture model, break exactly one thing,
assert the violation, then assert that fix, followed by a re-check,
comes back clean, and that running the fixer a second time changes
nothing further.
