# GitHub Actions and `act`

[![Check Markdown Links](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/CheckMarkdownLinks.yml/badge.svg)](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/CheckMarkdownLinks.yml)

I use GitHub [Actions][3] to run automated builds, tests, and checks on my repositories. I also like to use the [`act`][1] tool created by GitHub user [nektos][4] to develop and test my workflows before pushing them to GitHub.

`act` uses [Docker][5] and customized [images][6] to create a local environment similar to the environment used by GitHub Actions. The default `act` image is intentionally [incomplete][7] to keep its size manageable. While it works well for many cases, there are cases which require extra workflow steps in order to get the action to run locally.

## Arduino `compile-sketches` Action

The main problem that I have run into is with Arduino's [`compile-sketches`][8] action. Changes made in release [v1.1.0][10] changed the Python and related tools configuration which is incompatible with the default `act` image ([`catthehacker/ubuntu:act-latest`][9]).

In particular, the following error will be thrown when trying to use `compile-sketches` v1.1.0 and the `ubuntu:act-latest` image:

``` text
/var/run/act/workflow/1-composite-2.sh: line 5: pipx: command not found
Failure - Main Action setup
Job failed
```

### Fixing the `compile-sketches` Error

To fix this problem, `python`, `poetry`, and `pipx` need to be installed locally into the `act` Docker image.

While the workflow could be configured to install these tools regardless of whether the action is running on GitHub or locally using `act`, I prefer to install the tools only when running locally. This can be done by checking for the environment variable [`env.ACT`][11] before running an installation step.

To get the `compile-sketches` action to run locally, update your workflow as follows:

```yaml
    steps:                           # Existing code
      - uses: actions/checkout@main  # Existing code
# *** Add the code below ***
      - name: setup-python (using nektos/act locally)
        if: ${{ env.ACT }}
        uses: actions/setup-python@v4
        with:
          python-version: '>=3.11'
      - name: Install Poetry (using nektos/act locally)
        if: ${{ env.ACT }}
        uses: snok/install-poetry@v1
      - name: Install pipx (using nektos/act locally)
        if: ${{ env.ACT }}
        run: pip install pipx 
# *** Add the code above ***
      - uses: arduino/compile-sketches@v1  # Existing code
```

## Other Errors

I occasionally run into other errors that aren't specific to the actions themselves. In particular, I have seen timeout and rate-limiting errors that can generally be cleared by re-running the action after waiting a few minutes.

Examples of other errors:

```text
Downloading index: package_index.tar.bz2 Get "https://downloads.arduino.cc/packages/package_index.tar.bz2": dial tcp: lookup downloads.arduino.cc on 192.168.1.1:53: read udp 192.168.1.1:33527->192.168.1.1:53: i/o timeout
```

```text
arduino:avr-gcc@7.3.0-atmel3.6.1-arduino7 read tcp 192.168.1.1:57018->104.18.1.1:80: read: connection reset by peer
Error during install: read tcp 192.168.1.1:57018->104.18.1.1:80: read: connection reset by peer
```

```text
Error: Error response from daemon: Head "https://ghcr.io/v2/catthehacker/ubuntu/manifests/act-latest": Get "https://ghcr.io/token?scope=repository%3Acatthehacker%2Fubuntu%3Apull&service=ghcr.io": context deadline exceeded
```

```text
::error::API rate limit exceeded for 73.1.1.1. (But here's the good news: Authenticated requests get a higher rate limit. Check out the documentation for more details.)
-  ::error::API rate limit exceeded
```

### GitHub Actions Extension for VSCode

The [GitHub Actions][2] extension for VSCode does not recognize the `env.ACT` variable and flags an issue in the Problems window when you open an action YAML file for editing:

```text
Context access might be invalid: ACT
```

This is documented in several issues ([67][67], [61][61], [47][47]) and there does not appear to be a plan to fix it. Some issue comments claim that it is fixed, but it is not.

While the basic issue is slightly annoying, a more annoying aspect of the issue is that the message does not go away when you close the file, and the Problems window get cluttered with the messages. The only way to clear the messages after closing the file editor is to disable/enable the extension itself or quit and restart VSCode.

## Example Workflows

I have several example workflows available in my [.github repository][12]:

- [compile-sketches][13]
  - Arduino `compile-sketches` workflow updated with the fix mentioned above and with matrix build definitions for various hardware platforms
- [markdown-link-check][14]
  - Checks for dead links in Markdown files, automatically runs once a month
  - Note that you also need to create a file named `mlc_config.json`
- [arduino-lint][15]
  - Used with libraries published to the Arduino library manager to confirm that they meet the Arduino Library Spec rules
- [markdownlint][16]
  - Used to validate "clean" markdown code.
  - I typically do not run this as an action but instead use the related [VSCode extension][21] to check my Markdown code
  - Note that you also need to create a file named `markdownlintconfig.json`
- [build-dependent-repos][17]
  - Automatically trigger a compile-sketches action on repos that depend on a library that I updated

## References

- [`act` tool][1]
- GitHub [Actions][3]
- [Docker][5]
- Docker [images][6] for `act`
- [`compile-sketches`][8] action
- [markdown-link-check][18] action
- [arduino-lint][19] action
- [markdownlint][20] action
- [Markdownlint extension][21] for VSCode

## License

The software and other files in this repository are released under what is commonly called the [MIT License][100]. See the file [`LICENSE.txt`][101] in this repository.

[1]: https://github.com/nektos/act
[2]: https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-github-actions
[3]: https://docs.github.com/en/actions
[4]: https://github.com/nektos
[5]: https://www.docker.com
[6]: https://github.com/nektos/act/blob/master/IMAGES.md
[7]: https://github.com/nektos/act#default-runners-are-intentionally-incomplete
[8]: https://github.com/marketplace/actions/compile-arduino-sketches
[9]: https://github.com/catthehacker/docker_images
[10]: https://github.com/arduino/compile-sketches/releases/tag/v1.1.0
[11]: https://github.com/nektos/act#skipping-steps
[12]: https://github.com/Andy4495/.github
[13]: https://github.com/Andy4495/.github/blob/main/workflow-templates/arduino-compile-sketches-matrix.yml
[14]: https://github.com/Andy4495/.github/blob/main/workflow-templates/CheckMarkdownLinks.yml
[15]: https://github.com/Andy4495/.github/blob/main/workflow-templates/arduino-lint.yml
[16]: https://github.com/Andy4495/.github/blob/main/workflow-templates/markdownlint.yml
[17]: https://github.com/Andy4495/.github/blob/main/workflow-templates/build-dependent-repos.yml
[18]: https://github.com/gaurav-nelson/github-action-markdown-link-check
[19]: https://github.com/marketplace/actions/arduino-arduino-lint-action
[20]: https://github.com/marketplace/actions/markdownlint-cli
[21]: https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint
[47]: https://github.com/github/vscode-github-actions/issues/47
[61]: https://github.com/github/vscode-github-actions/issues/61
[67]: https://github.com/github/vscode-github-actions/issues/67
[100]: https://choosealicense.com/licenses/mit/
[101]: ./LICENSE.txt
[//]: # ([200]: https://github.com/Andy4495/GitHub-actions-and-act)

[//]: # (This is a way to hack a comment in Markdown. This will not be displayed when rendered.)
