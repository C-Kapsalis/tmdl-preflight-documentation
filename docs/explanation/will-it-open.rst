Will Power BI Desktop actually open this
==========================================

.. NEW PAGE. The M006-M009 rule family (model-table-refs, table-partitions,
   entity-query-source, reserved-table-name) has no explanation page
   anywhere, even though each one maps to a specific opaque Power BI
   Desktop error string, exactly the "why this matters" content this
   quadrant exists for. Follow the shape of the existing
   ``duplicate-lineage-tags`` page: state the failure, the exact Desktop
   error text it produces, and why the rule catches it before that error
   ever appears.

.. One subsection per rule, each pairing the rule id with the Desktop
   error string it prevents:
   M006 -> the project never loads (missing ``ref table`` line)
   M007 -> "Sequence contains no elements" (a table has no partition)
   M008 -> "A composite model cannot be used with entity based query
   sources" (inline ``#table(type table ...)`` forces a composite model)
   M009 -> "Unsupported Table name ... found in data model schema" (a
   table literally named "Measures")
