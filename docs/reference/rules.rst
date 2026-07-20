Rule catalog
============

Severity semantics: error, will break an import or deploy or silently
corrupt results; warning, suspicious, deserves a look before shipping;
info, advisory, never blocks (not even with ``--strict``).

Select rules by id or name, case-insensitively: ``--select M003`` and
``--select lineage-duplicates`` are equivalent.

Each rule links to an explanation guide covering the underlying
pitfall.

M001, model-structure
-------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - each ``definition/`` folder

The definition folder must contain ``model.tmdl`` and a ``tables/``
folder with at least one ``.tmdl`` file. Guards against truncated
exports and bad merges that drop whole folders.
See :doc:`../explanation/tmdl-well-formedness`.

M002, tmdl-well-formed
--------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every ``.tmdl`` file, recursively

Each file must decode as UTF-8, contain no null bytes, have an even
number of expression fences, and (for table files) declare a table.
See :doc:`../explanation/tmdl-well-formedness`.

M003, lineage-duplicates
-----------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - yes; keeps the first occurrence, regenerates the rest
   * - Scope
     - every ``lineageTag:`` across the definition folder

No two objects may share a ``lineageTag``. Duplicates are rejected at
import ("an object with that lineage tag already exists").
See :doc:`../explanation/duplicate-lineage-tags`.

M004, lineage-format
-------------------------

.. list-table::

   * - Severity
     - warning
   * - Auto-fix
     - yes; regenerates the tag as a fresh UUID
   * - Scope
     - every ``lineageTag:`` across the definition folder

Tags must be canonical hyphenated UUIDs (8-4-4-4-12 hex). Placeholder
or hand-typed tags defeat the identity mechanism lineage tags exist
for. See :doc:`../explanation/duplicate-lineage-tags`.

M005, column-data-types
----------------------------

.. list-table::

   * - Severity
     - warning
   * - Auto-fix
     - no
   * - Scope
     - every declared column ``dataType:``

Value must be one of ``string``, ``int64``, ``double``, ``decimal``,
``dateTime``, ``boolean``, ``binary``, ``variant``, ``currency``,
``rowNumber``. See :doc:`../explanation/column-data-types`.

M006, model-table-refs
---------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - yes; appends the missing ``ref table`` line to ``model.tmdl``
   * - Scope
     - every ``tables/*.tmdl`` versus the ``ref table`` lines in
       ``model.tmdl``

Every table must be linked to the model with a ``ref table <name>``
line in ``model.tmdl``. A table file with no matching ref is not
attached to the model, so Power BI Desktop refuses to open the
project. See :doc:`../explanation/will-it-open`.

M007, table-partitions
---------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every table

Every table must declare at least one partition; a measures-only
table still needs one (a calculated partition, or a dummy
entered-data partition). Without one, Power BI crashes on open while
resolving the table's query. See :doc:`../explanation/will-it-open`.

M008, entity-query-source
------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every M partition source

An inline ``#table(type table [...])`` source is entity-typed;
combined with any calculated table, Power BI treats the model as
composite and refuses to open it. Re-encode manual data as entered
data (``Table.FromRows(Json.Document(Binary.Decompress(...)))``) or
use a calculated table. See :doc:`../explanation/will-it-open`.

M009, reserved-table-name
------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every table name

Power BI reserves certain table names and refuses to open a model
that uses one, most notably a table literally named ``Measures`` (it
collides with the implicit measures container: "Unsupported Table
name ... found in data model schema"). Rename it, for example to
``Key Measures`` or ``<Domain> Measures``. See
:doc:`../explanation/will-it-open`.

D001, dax-delimiters
-------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every DAX expression: measures, calculated columns,
       calculated-partition sources, calculation items, dynamic
       format strings

Parentheses, braces, brackets, strings, and quoted identifiers must
balance. Comment contents and string contents are excluded from the
count. See :doc:`../explanation/dax-delimiter-balance`.

D002, nameof-resolution
----------------------------

.. list-table::

   * - Severity
     - error (warning for ambiguous references)
   * - Auto-fix
     - no
   * - Scope
     - every ``NAMEOF()`` call in every DAX expression

``NAMEOF('Table'[Member])`` must point at an existing column or
measure on that table; ``NAMEOF([Measure])`` at an existing measure.
Qualified references to measures are accepted; Power BI Desktop
writes that form itself when building field parameters. An ambiguous
bare-measure reference (the same measure name declared in several
tables) is downgraded to a warning, because rule D003 already
reports the underlying duplicate as an error. See
:doc:`../explanation/nameof-reference-integrity`.

D003, duplicate-measure-names
----------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - all measures, model-wide

The tabular engine requires measure names to be unique across the
whole model, not merely within a table. See
:doc:`../explanation/duplicate-measure-names`.

R001, relationship-endpoints
----------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every relationship in ``relationships.tmdl``

The from and to tables must exist and the referenced columns must
exist on them. See :doc:`../explanation/relationship-integrity`.

R002, relationship-cardinality
------------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no
   * - Scope
     - every relationship

``fromCardinality`` and ``toCardinality``, where declared, must be
``one`` or ``many``. See :doc:`../explanation/relationship-integrity`.

R003, relationship-duplicates
-----------------------------------

.. list-table::

   * - Severity
     - warning
   * - Auto-fix
     - no
   * - Scope
     - every relationship

Two relationships over the same column pair (keyed on endpoints, not
GUIDs). See :doc:`../explanation/relationship-integrity`.

R004, relationship-bidirectional
--------------------------------------

.. list-table::

   * - Severity
     - info
   * - Auto-fix
     - no
   * - Scope
     - every relationship

Surfaces each ``crossFilteringBehavior: bothDirections`` so the
modeler confirms it is intentional. See
:doc:`../explanation/bidirectional-and-orphan-tables`.

R005, orphan-tables
------------------------

.. list-table::

   * - Severity
     - info
   * - Auto-fix
     - no
   * - Scope
     - every table

Data tables that participate in no relationship. Tables that are
unrelated by design are exempt: measure-only tables, tables whose
columns are all hidden (the measure-home pattern), hidden tables,
field parameters, and calculated helper tables. Deliberate
disconnected slicer tables with visible columns still surface, as
info, so they never fail a run. See
:doc:`../explanation/bidirectional-and-orphan-tables`.

F001, fieldparam-comma-runs
---------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - yes; keeps each run's first comma, deletes the orphans
   * - Scope
     - every ``calculated`` partition source (field parameters,
       calculated tables)

Two or more commas separated only by whitespace, the classic leftover
of deleting a row without its separator, make the whole source
unparseable. Commas inside ``--``/``//`` comments are exempt. See
:doc:`../explanation/field-parameter-comma-runs`.

B001, bookmark-int-types
------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - yes; coerces the value to its integer form
   * - Scope
     - every ``*.bookmark.json`` under each ``*.Report`` folder

``howCreated``, ``ComparisonKind``, and ``Version`` must be JSON
integers, not strings. Known string spellings of ``howCreated``
(``"User"``, ``"System"``, and so on) are mapped; unknown strings are
left for a human. See :doc:`../explanation/bookmark-integer-types`.

B002, bookmark-visual-refs
--------------------------------

.. list-table::

   * - Severity
     - error
   * - Auto-fix
     - no; the right resolution depends on intent
   * - Scope
     - every ``*.bookmark.json`` under each ``*.Report`` folder

Every visual a bookmark references must exist as a visual folder
under ``pages/*/visuals/``. See
:doc:`../explanation/bookmark-visual-references`.

S001, format-strings
-------------------------

.. list-table::

   * - Severity
     - info
   * - Auto-fix
     - no
   * - Scope
     - every visible measure

Visible measures should declare a ``formatString``. Models that carry
``formatStringDefinition`` blocks (dynamic formatting via calculation
groups) are exempted automatically; disable explicitly with
``--ignore S001`` if your formatting strategy lives elsewhere. See
:doc:`../explanation/format-strings`.

Configuration
----------------

``tmdl-preflight`` has no config file in this release; scoping is
done per invocation with ``--select``/``--ignore`` (or the pytest
fixture's ``select=``/``ignore=`` arguments). Custom rules are added
in code; see :doc:`../how-to/add-a-custom-rule`.
