Run only the auto-fixable rules
================================

Goal: apply every safe repair in one pass, for example before a
release branch is cut, without being distracted by advisory findings.

Five rules ship a fixer:

.. list-table::
   :header-rows: 1

   * - ID
     - Name
     - Repairs
   * - M003
     - lineage-duplicates
     - regenerates duplicate ``lineageTag`` values
   * - M004
     - lineage-format
     - regenerates malformed ``lineageTag`` values
   * - M006
     - model-table-refs
     - appends the missing ``ref table`` line to ``model.tmdl``
   * - F001
     - fieldparam-comma-runs
     - removes orphan commas from calculated-table sources
   * - B001
     - bookmark-int-types
     - coerces string-typed bookmark fields to integers

One command
--------------

::

    $ tmdl-preflight fix MyProject/ --select M003,M004,M006,F001,B001

``--select`` accepts names too:

::

    $ tmdl-preflight fix MyProject/ --select lineage-duplicates,lineage-format,model-table-refs,fieldparam-comma-runs,bookmark-int-types

The run checks, fixes, then re-checks. Exit code 0 means the selected
rules are clean after the repair; a nonzero exit means something in
the selected set could not be safely repaired (for example a bookmark
``howCreated`` value with no known integer meaning) and needs a
human.

Preview before writing
--------------------------

``check`` with the same selection shows exactly what ``fix`` would
touch, without touching it:

::

    $ tmdl-preflight check MyProject/ --select M003,M004,M006,F001,B001

Every violation printed with ``(auto-fixable)`` is one the subsequent
``fix`` run will handle.

Review the result
---------------------

Fixers make surgical, line-level edits, so the diff is small and
readable:

::

    $ git diff --stat
     .../definition/tables/Stores.tmdl                  | 2 +-
     .../definition/bookmarks/spotlight.bookmark.json   | 2 +-

Commit the fixes as their own commit, "apply tmdl-preflight
auto-fixes," so reviewers can wave it through in seconds.
