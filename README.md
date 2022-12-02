# sonarcloud-scoverage-fix

This action aims to fix the default behavior of the SonarCloud Scoverage sensor, mismatching the import of scala sources.

## What's the matter?

If you tried using the "SonarCloud" analysis tool with the "scala" language, the first thing you tried is setting up the "Scoverage" coverage generation tool to work with it. Since the last scala version, the 3.2 one, the Scoverage tool has been nominated as the de facto tool for its purpose in scala language. Its support for scala 3 and the poor interaction of JaCoCo with Dotty, the scala 3 compiler, make it the best of its kind.

Nonetheless, the SonarCloud sensor has problems using it while publishing this action on the Marketplace. It is possible to specify
the path where the generated data is, but the problem arises when reading it. If the file generation happens in a GitHub runner, for example, thanks to a GitHub Action running the sbt command ``generateReport``, all paths in that file are absolute. Furthermore, their root is always the ``/github/home`` folder because, as per
[GitHub documentation](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#file-systems),
all commands run with that folder as the working folder.

Following the same documentation, when the repository is cloned locally to the runner with the ``checkout`` action, the ``/github/workspace`` folder stores all code. When the sensor tries to detect the absolute path of each source file, it is different from the one contained in the Scoverage report.

[This answer](https://community.sonarsource.com/t/failed-to-resolve-files-with-scoverage/68825/4) to the question first highlighted this incorrect behavior. In turn, it received no confirmation of its correctness, but it is correct.
[Relevant XKCD](https://xkcd.com/979/). Maybe the answer is a bit vague, but certainly in the right direction, given the sensor
warns us about this error showing us the following warning.

> ``WARN: Fail to resolve ... file(s). No coverage data will be imported on the following file(s): /home/runner/work/...``

So, dear people from the future, I'm here to reassure you that a solution is possible and provided in this repository. I created a new action that forwards all behavior to the
[SonarCloud Action](https://github.com/SonarSource/sonarcloud-github-action), except that, before doing so, it corrects all paths
in your Scoverage report file using the ``sed`` and ``pwd`` commands, available in every Linux distribution, even in the WSL.
That's it. That's the solution.

## So, what should I do?

You need to pass the path of the Scoverage report location, complete with the XML filename, to this action using the
``scoverageReport`` key. This key is required. An example of workflow using it is the following one.

```YAML
name: Workflow

on:
  pull_request:

jobs:
  example:
    name: Example job
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Setup scala environment
        uses: olafurpg/setup-scala@v13
        with:
          java-version: openjdk@1.17
      - name: Generate coverage report
        run: sbt clean coverage test coverageReport
      - name: SonarCloud scan fixed
        uses: pervasive-cats/sonarcloud-scoverage-fix@v1.0.0
        with:
            scoverageReport: target/scala-3.2.1/scoverage-report/scoverage.xml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

For everything else, use this action as you would use the SonarSource one, passing all inputs you would have given to it. The
interface is the same for compatibility reasons and because now there is no need to explain how it works.
[SonarCloud already did this for me](https://github.com/SonarSource/sonarcloud-github-action#usage).

## What about the future?

I hope the guys at SonarCloud fix this ridiculous bug. But, in the meantime, you can always use my action. :)
