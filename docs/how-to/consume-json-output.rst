Consume JSON output for tool integration
========================================

Goal: feed a preflight report into another tool (a dashboard, a bot, a
custom gate) instead of reading the text report yourself.

Getting the JSON
--------------------

Both ``check`` and ``fix`` accept ``--json``:

::

    $ tmdl-preflight check MyProject/ --json > preflight.json

The shape
------------

::

    {
      "summary": {
        "errors": 1,
        "warnings": 0,
        "infos": 0,
        "fixes_applied": 0
      },
      "fixes": [],
      "violations": [
        {
          "rule": "M003",
          "severity": "error",
          "message": "lineageTag '...' duplicates the one on table Products (...)",
          "file": ".../tables/Stores.tmdl",
          "line": 3,
          "object": "table Stores",
          "fixable": true
        }
      ]
    }

This is the exact structure ``PreflightReport.to_dict()`` returns, so
the same shape is available from the plain Python API without going
through the CLI at all:

::

    from tmdl_preflight.rules.base import Context
    from tmdl_preflight.rules import default_ruleset
    from tmdl_preflight.engine import check

    report = check(Context("MyProject/"), default_ruleset())
    payload = report.to_dict()

A worked example: posting a summary line
----------------------------------------------

::

    import json
    import subprocess

    result = subprocess.run(
        ["tmdl-preflight", "check", "MyProject/", "--json"],
        capture_output=True, text=True,
    )
    payload = json.loads(result.stdout)
    summary = payload["summary"]
    print(
        f"{summary['errors']} error(s), {summary['warnings']} warning(s), "
        f"{summary['fixes_applied']} fix(es) applied"
    )

``violations`` reflects the final state of the run: for ``fix``, that
is the post-repair state, so anything still listed there needed a
human even after auto-fixing ran.
