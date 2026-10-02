# gh-actions-templates-public
GitHub Actions Templates

Reusable workflows (`on: workflow_call`) shared by the VIPER repositories
(toolviper, xradio, graphviper, astroviper, flowviper, testviper, benchviper).
Call them from a workflow in your repository:

```yaml
jobs:
  call-testing-linux:
    uses: nrao/gh-actions-templates-public/.github/workflows/python-testing-linux-template.yml@main
    with:
      cov_project: "mypackage"
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

## CI policy

| Event | What runs |
|---|---|
| Push to any branch other than `main` | Lint (pre-commit or `ruff-template.yml`) and Linux tests on Python 3.13 |
| Pull request, push to `main` (a merge) | Everything: lint, Linux and macOS tests on Python 3.12, 3.13 and 3.14, notebooks, integration tests, package builds |

The VIPER callers implement this as follows:

- They trigger the Linux template on both `push` and `pull_request` and pass
  `python-versions: '["3.12", "3.13", "3.14"]'` and
  `push-python-versions: '["3.13"]'`. The template then tests only 3.13 on
  pushes to branches other than the default branch.
- They trigger the other testing templates on `pull_request` and on `push` to
  `main` only.

Python 3.11 is still allowed by the packages' `requires-python`, but it is no
longer tested there.

The template defaults are kept backward compatible (for example the Linux and
macOS defaults are still 3.11-3.13 and the push split is off unless
`push-python-versions` is set), because other repositories call these
templates `@main` too.

## Templates

| Template | Purpose | Main inputs (default) |
|---|---|---|
| `python-testing-linux-template.yml` | pytest + coverage on Linux, uploads coverage and test results to Codecov | `cov_project` (required), `python-versions` (`["3.11", "3.12", "3.13"]`), `push-python-versions` (`""` = use `python-versions`), `test-path` (`tests/`), `gcc-version` (`""`), `ignore-requires-python` (`false`) |
| `python-testing-macos-template.yml` | pytest on macOS arm64 in a conda-forge env with python-casacore | `python-versions` (`["3.11", "3.12", "3.13"]`), `test-path`, `xcode-version` (`""`), `ignore-requires-python` |
| `run-ipynb-template.yml` | Executes every notebook under `docs/`; fails if any notebook fails | `python-version` (`3.12`), `gcc-version`, `ignore-requires-python` |
| `python-testing-integration-template.yml` | Installs ToolVIPER, XRADIO, GraphVIPER and AstroVIPER together and runs all their tests; on a push to `main` it dispatches to casangi/testviper | `python-versions` (`["3.13"]`), `<package>_ref` (`main`), `gcc-version` (`14`), `ignore-requires-python` |
| `python-testing-casatools-template.yml` | pytest with casatools instead of python-casacore | `cov_project` (required), `python-versions` (`["3.11", "3.12"]`; casatools has no cp314 wheels), `casatools-version` (`""`), `pytest_ignore` |
| `ruff-template.yml` | `ruff check` and `ruff format --check` for repositories without pre-commit | `ruff-version` (`""` = pyproject pin or latest), `src` (`.`), `format-check` (`true`) |
| `python-publish-cngi-template.yml` | Builds sdist + wheel of a pure-Python package, uploads to PyPI for tag refs | `pypi-url` (required); secret `PYPI_TOKEN` |
| `python-publish-cpp-cngi-template.yml` | cibuildwheel wheels (CPython 3.12-3.14; Linux x86_64 + aarch64 on native runners, macOS arm64) + sdist, uploads to PyPI for release events | `pypi-url` (required), `import-name` (repository name), `cibw-build` (`cp312-* cp313-* cp314-*`), `ignore-requires-python` (`false`); secret `PYPI_TOKEN` |
| `python-publish-template.yml` | Same as the cngi publish template (non-strict metadata check), for callers using `secrets: inherit` | `pypi-url` (required) |
| `python-publish-cpp-template.yml` | cibuildwheel wheels (Linux x86_64, macOS arm64) + sdist, uploads to PyPI for tag refs | `pypi-url` (required), `import-name`, `cibw-build`, `ignore-requires-python` |
| `black-template.yml` | **Deprecated**, use `ruff-template.yml` | |

The publish templates only upload for releases/tags, so callers can also
trigger them on pull requests to build-check the package.

All secrets are optional at the `workflow_call` level so pull requests from
forks (which receive no secrets) can still run; the steps that need a secret
use it only where it is available.

## Supporting a new Python version

Every VIPER package caps `requires-python` below the next minor Python
version, and the packages depend on each other (toolviper -> graphviper ->
xradio -> toolviper; astroviper -> graphviper; flowviper -> astroviper). After
raising the caps, a test job on the new version still cannot install the
siblings from PyPI until they are re-released. During that window callers set
`ignore-requires-python: true`, which makes pip ignore `Requires-Python`
(`PIP_IGNORE_REQUIRES_PYTHON=1`) and prints a warning annotation. Remove it
once the new releases are on PyPI.
