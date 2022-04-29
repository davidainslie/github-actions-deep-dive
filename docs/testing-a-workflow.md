# Testing a Workflow

A follow-on from: Build a workflow in project [github-actions-deep-dive-lesson](https://github.com/davidainslie/github-actions-deep-dive-lesson).

This time we'll go through our [third lesson](https://github.com/davidainslie/github-actions-deep-dive-lesson-3).

To test, we'll use `matrices` in GHA. GitHub Actions will run a job for each value in a matrix to cover multiple (test) cases, versions, or other lists.
E.g.

```yaml
- matrix:
  os: [ubuntu-18.04, ubuntu-20.04, Windows, MacOS]
  python: [3.4, 3.8]

- runs-on: ${{matrix.os}}
- uses: actions/setup-python@v2
  with:
    python-version: ${{matrix.python}}
```

Our job would now be run 8 times:

|           | Python 3.4 | Python 3.8 |
| --------- | ---------- | ---------- |
| Ubuntu-18 | Job 1      | Job 5      |
| Ubuntu-20 | Job 2      | Job 6      |
| Windows   | Job 3      | Job 7      |
| MacOS     | Job 4      | Job 8      |

