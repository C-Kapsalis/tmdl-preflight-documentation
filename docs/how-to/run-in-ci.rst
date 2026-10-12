How to run tmdl-preflight in CI
===================================

Goal: block a pull request that would break the model, before anyone
opens it in Power BI Desktop. The rule of thumb is **CI checks;
developers fix**: the pipeline runs the read-only ``check``, and
auto-fixes happen on a developer's machine, where the diff is reviewed
and committed like any other change.

Every example below runs from the repository root, which holds the
PBIP project, and assumes Python 3.10 or newer.

Before every commit: pre-commit
-----------------------------------

The repository ships a ``.pre-commit-hooks.yaml``. Add it to the
project's ``.pre-commit-config.yaml``:

::

    repos:
      - repo: https://github.com/c-kapsalis/tmdl-preflight
        rev: v1.1.0
        hooks:
          - id: tmdl-preflight

The hook checks the whole project rather than only the staged files,
because a TMDL defect is often caused in one file and reported in
another. Add ``args: [".", "--strict"]`` to block on warnings too. A
second hook, ``tmdl-preflight-fix``, applies the mechanical fixes and
re-checks; use it only if your team wants fixes applied on commit.

On every pull request: GitHub Actions
-----------------------------------------

``examples/ci/github-actions.yml`` in the repository is a complete
workflow; copy it to ``.github/workflows/tmdl-preflight.yml``. Its
core:

::

    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
    - run: pip install "tmdl-preflight @ git+https://github.com/c-kapsalis/tmdl-preflight@v1.1.0"
    - run: tmdl-preflight check . --format github

``--format github`` prints workflow commands, so each violation shows
up as an annotation on the changed line of the pull request diff. The
step fails the job on errors (and on warnings with ``--strict``).

To also see findings as code-scanning alerts, write SARIF and upload
it. The job needs ``security-events: write`` permission:

::

    - run: tmdl-preflight check . --format sarif > tmdl-preflight.sarif || true
      if: always()
    - uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: tmdl-preflight.sarif
        category: tmdl-preflight

For a readable summary on the run page, append the Markdown report to
the job summary:

::

    - run: tmdl-preflight check . --format markdown >> "$GITHUB_STEP_SUMMARY"
      if: always()

Azure DevOps
----------------

The same shape:

::

    steps:
      - task: UsePythonVersion@0
        inputs:
          versionSpec: "3.12"
      - script: pip install "tmdl-preflight @ git+https://github.com/c-kapsalis/tmdl-preflight@v1.1.0"
      - script: tmdl-preflight check $(Build.SourcesDirectory) --strict
        displayName: TMDL preflight

Gate only on new violations
-------------------------------

On a model that is not clean yet, record a baseline once, commit it,
and point CI at it:

::

    $ tmdl-preflight baseline .
    $ git add .tmdl-preflight-baseline.json

::

    - run: tmdl-preflight check . --baseline .tmdl-preflight-baseline.json --format github

The job now fails only on violations that are not in the baseline. As
the team fixes the old ones, ``tmdl-preflight baseline . --update``
shrinks the file; it never grows it unless ``--accept-new`` is given.
Defects that stop the model from opening can never be baselined. See
:doc:`../reference/configuration`.

Different settings for CI
-----------------------------

Keep one configuration file and give CI its own profile:

::

    [tool.tmdl-preflight.profiles.ci]
    strict = true

::

    - run: tmdl-preflight check . --profile ci --format github

Exit codes are the contract: ``0`` clean, ``1`` violations remain,
``2`` a usage, path, configuration or baseline problem, so a missing
model folder or a typo in the configuration fails the job loudly
instead of passing vacuously.
