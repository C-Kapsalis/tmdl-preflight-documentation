Select and ignore specific rules
================================

Goal: run a narrower set of rules than the full default catalog, the
same way across the CLI, pytest, and the plain Python API.

All three surfaces share the same semantics: name rules by id or by
name, case-insensitively, and the two options are mutually
narrowing, not additive; ``--select`` picks the working set,
``--ignore`` removes from it.

On the CLI
-------------

::

    $ tmdl-preflight check MyProject/ --select M003,M004
    $ tmdl-preflight check MyProject/ --ignore S001

Both flags accept rule names as well as ids:

::

    $ tmdl-preflight check MyProject/ --select lineage-duplicates,lineage-format

In pytest
------------

The fixture's ``assert_clean`` and ``run`` take the same idea as
keyword arguments:

::

    def test_lineage_only(tmdl_preflight):
        tmdl_preflight.assert_clean("src/Shop.SemanticModel",
                                    select={"M003", "M004"})

    def test_everything_but_style(tmdl_preflight):
        tmdl_preflight.assert_clean("src/Shop.SemanticModel",
                                    ignore={"S001"})

In the plain Python API
----------------------------

``RuleSet.select(select=None, ignore=None)`` narrows a ruleset before
you hand it to ``check()`` or ``fix()``:

::

    from tmdl_preflight.rules import default_ruleset

    ruleset = default_ruleset().select(select={"m003", "m004"})

Selecting an unknown id or name is an error, not a silent no-op: the
CLI exits 2 and names the unrecognized value, so a typo in a pipeline
configuration fails loudly instead of quietly running every rule.
