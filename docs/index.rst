tmdl-preflight
==============

Preflight checks and auto-fixes for Power BI semantic models saved in
TMDL format (PBIP folders). For most of Power BI's life the semantic
model was a binary ``.pbix`` blob: un-diffable, un-reviewable,
un-testable. PBIP and TMDL changed that; the model is now a folder of
plain-text files, which means it can finally be treated like code, and
today that also means it can be developed with AI or a custom script
just as easily as by hand. This shift has (almost) elevated Power BI
engineering into a proper software discipline, and a software
discipline is governed by the same standards as any other. The
central pillar of those standards is testing.

``tmdl-preflight`` is that testing layer: a CLI built on pytest that
tests a model's TMDL artifacts against a set of rules for breaking
changes and unaccepted practices. Some violations it repairs itself;
others it reports and leaves to a human. Twenty-one rules ship today,
five of them auto-fixable, reachable from the CLI or a pytest fixture.
Both the rule set and the auto-fixes are meant
to be extended: fork or clone the repository, add a rule of your own,
and give it a fixer if the repair is safe to automate.

Audience
-----------

This is written for BI developers already comfortable with a Power BI
model-as-code workflow, and specifically with TMDL. It assumes you
know what a ``lineageTag`` is, what a field parameter's partition
source looks like, and why a PBIP folder is structured the way it is.

Why this matters
--------------------

The `one-canvas philosophy
<https://bi-report-template-site-documentation.readthedocs.io/en/latest/explanation/one-canvas-philosophy.html>`_
makes this same argument on the reporting side: build one governed
report template once, and every question after that costs a bookmark
instead of a new page. The same argument holds a level down the stack, and
because every party involved here is technical, it compounds faster.
A platform team builds one template model that satisfies this rule
catalog; other teams then fork it, extend it, or build their own
models against the same enforced rules, with ``tmdl-preflight`` as the
gate that keeps every derived model honest. The business result is not
an engineering nicety: it is validated, deployable data models
reaching every department faster, because the review that used to
happen slowly and by hand now happens on a developer's machine before
the pull request is even opened.

.. toctree::
   :maxdepth: 1

   tutorials/getting-started
   how-to/modify-the-rules-list
   how-to/modify-the-auto-fixes
   reference/usage
   reference/customization
   contributing
