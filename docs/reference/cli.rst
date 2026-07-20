CLI reference
=============

.. Port from tmdl-preflight-docs/reference/cli.md. Per the CLI archetype
   notes: document flags exactly as printed (kebab-case, lowercase
   description, no trailing period), and keep the exit-code table in the
   same three-row shape used across all three CLI projects.

Commands
--------

.. ``check``, ``fix``, ``rules``, one subsection each, with the flags
   each accepts (``--select``, ``--ignore``, ``--json``, ``--strict``).

Exit codes
----------

.. Table: 0 clean, 1 violations remain, 2 usage or path error.

Accepted paths
--------------

.. The four path shapes ``Context`` resolves: a PBIP project root, a
   ``*.SemanticModel`` folder, a ``definition`` folder, a ``*.Report``
   folder.
