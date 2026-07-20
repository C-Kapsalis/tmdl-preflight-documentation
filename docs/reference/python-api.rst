Python API reference
=====================

.. NEW PAGE, no Markdown draft exists yet for this one. The README
   advertises the plain Python API as a first-class third entry point
   alongside the CLI and the pytest fixture, but it has never had a
   reference page anywhere. Follow ``_kit/library-archetype-notes.md``, not
   the CLI archetype: signatures and return shapes, not flags and exit
   codes.

``Context``
-----------

.. Constructor accepts a PBIP root, a ``*.SemanticModel`` folder, a
   ``definition`` folder, or a ``*.Report`` folder. Note ``.reload()``
   (drops the parse cache) and ``.is_empty()``.

``default_ruleset()``
----------------------

.. Returns the full ``RuleSet``. Note ``RuleSet.select(select=, ignore=)``
  , case-insensitive rule ids or names, same semantics as the CLI's
   ``--select``/``--ignore`` but state that explicitly rather than assuming
   the reader infers it.

``check(ctx, ruleset)`` and ``fix(ctx, ruleset)``
---------------------------------------------------

.. Both return a ``PreflightReport``. One short runnable example per
   function, call then result, in that order.

``PreflightReport``
---------------------

.. Fields and methods to document: ``.errors``, ``.warnings``, ``.infos``
   (each a list of ``Violation``), ``.exit_code(strict=False)``,
   ``.to_dict()``.

``Violation``
--------------

.. Fields: ``rule_id``, ``severity``, ``message``, ``file``, ``line``,
   ``obj``, ``fixable``. Note ``.location()``.
