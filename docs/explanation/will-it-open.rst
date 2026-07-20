Will Power BI Desktop actually open this
==========================================

Rules M006 through M009.

The pitfall
--------------

Every other structural rule in this catalog protects a model that
already opens: a duplicate lineage tag, a malformed one, a wrong
column type, all surface as an import rejection or a silent
correctness problem, but the project itself loads. This family is
different. Each of these four defects stops Power BI Desktop from
opening the project at all, and each one produces an opaque, generic
error message that names neither the file nor the actual cause. A
modeler who hits one of these has no lead at all; the project simply
refuses to open.

The four failures
---------------------

M006, model-table-refs
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A table file exists under ``tables/``, but ``model.tmdl`` has no
matching ``ref table <name>`` line. The table is defined but never
attached to the model. The project does not error with any specific
message naming the table; it simply never loads, because the model's
own manifest does not know the table file exists. This usually
happens when a table is added by hand, or a merge drops the ``ref``
line while keeping the table file itself.

M007, table-partitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A table has no partition at all, not even a calculated one. Power BI
Desktop crashes on open while resolving the table's query, with the
message "Sequence contains no elements." Nothing in the message names
the table; the modeler is left checking every table by hand.
Measures-only tables are a common trap here: they still need a
partition, typically a calculated one or a dummy entered-data
partition, even though they carry no real data.

M008, entity-query-source
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An M partition's source is an inline ``#table(type table [...])``
construct. Power BI classifies this as an entity-typed query source,
and combined with any calculated table elsewhere in the model, it
treats the whole model as composite, refusing to open it with "A
composite model cannot be used with entity based query sources." The
fix is to re-encode the manual data as entered data
(``Table.FromRows(Json.Document(Binary.Decompress(...)))``) or to use
a calculated table instead.

M009, reserved-table-name
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A table is literally named ``Measures``. Power BI reserves that name
for its own implicit measures container, and refuses to open the
project with "Unsupported Table name ... found in data model schema."
The fix is simply to rename the table, for example to ``Key Measures``
or ``<Domain> Measures``.

Why these four get no auto-fix, except one
-------------------------------------------------

M006 is auto-fixable: the missing ``ref table`` line is fully
determined by the table file that already exists, so appending it is
safe and mechanical. The other three are not, for the same reason
most of this catalog is not: the correct partition source, the
correct re-encoding of an entity-typed source, and the correct new
name for a reserved table all require intent the file does not
record. What every rule in this family contributes, fixable or not,
is turning a project that silently refuses to open into a message
that names the actual file and the actual cause, before anyone has to
guess.
