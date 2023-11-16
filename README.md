# GitHub Actions and `act`

[![Check Markdown Links](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/CheckMarkdownLinks.yml/badge.svg)](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/CheckMarkdownLinks.yml)

I use GitHub [Actions][3] workflows to run automated builds, tests, and checks on my repositories.

I develop and test my workflows before pushing them to GitHub with the [`act`][1] tool created by GitHub user [nektos][4].

## GitHub Actions

This section documents some conventions I use with my actions, along with examples of workflows that I use across my repos. See GitHub's comprehensive [documentation][3] for more details on action terminology and workflow syntax.

### GitHub Reusable Workflows

GitHub Actions have a feature called [reusable workflows][24] which essentially allows one workflow to call another workflow. This makes it convenient to have a "base" workflow (referred to as a "called workflow") in a central location that does most of the work, and a per-repository workflow (referred to as a "caller workflow") which configures the settings needed to run the base workflow.

Taking advantage of reusable workflows becomes especially useful when you need to make a change to the functionality of an action that is used in multiple repos (like when I had to fix the issue with `act` as described [below][25]). Instead of having to make the change in every repo which uses the action, all that needs to change is the single base workflow. See [.github/workflows][29] for the reusable workflows that my repos call.

### Workflow Structure

I have a `workflow_dispatch` trigger (including an optional input string) on all my workflows so that I can trigger them manually from the [repo Actions screen][30]:

```yaml
  workflow_dispatch:
    inputs:
      message:
        description: Message to display in job summary
        required: false
        type: string
```

For any workflow that can trigger multiple job runs based on different combinations of varibles, I use a [matrix strategy][26]. Even if the repo only needs to be compiled for one platform, I still use a matrix strategy for consistency with my other repos and my reusable workflows.

For [compiling arduino sketches][27] across multiple platforms:

```yaml
  compile-sketches: 
    strategy:
      matrix:
        include:
          - arch: avr
            fqbn: 'arduino:avr:uno'
            platform-name: 'arduino:avr'
            platform-sourceurl: 'https://downloads.arduino.cc/packages/package_index.json'
          - arch: msp-G2
            fqbn: 'energia:msp430:MSP-EXP430G2553LP'
            platform-name: 'energia:msp430'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/main/json/package_energia_minimal_MSP_107_index.json'
          - arch: msp-F5529
            fqbn: 'energia:msp430:MSP-EXP430F5529LP'
            platform-name: 'energia:msp430'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/main/json/package_energia_minimal_MSP_107_index.json'
          - arch: msp432
            fqbn: 'energia:msp432r:MSP-EXP432P401R'
            platform-name: 'energia:msp432r'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/main/json/package_energia_minimal_MSP432r_index.json'
          - arch: tivac
            fqbn: 'energia:tivac:EK-TM4C123GXL'
            platform-name: 'energia:tivac'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/main/json/package_energia_minimal_TM4C_104_alternate_index.json'
          - arch: esp8266
            fqbn: 'esp8266:esp8266:thing'
            platform-name: 'esp8266:esp8266'
            platform-sourceurl: 'http://arduino.esp8266.com/stable/package_esp8266com_index.json'
```

For [triggering builds][28] on multiple repos which are dependent on a library:

```yaml
  build-dependent-repos:
    strategy:
      matrix:
        include:
          - name: Trigger build on <Repo Name>
            repo: <Repo name>
```

### Workflow Examples

I have several template workflows available in my [.github repository][12]:

- [arduino-compile-sketches][13]
  - Arduino [`compile-sketches`][8] workflow with matrix build definitions for various hardware platforms.
  - Update the matrix list as needed for the repo being compiled.
- [markdown-link-check][14]
  - Checks for dead links in Markdown files. Automatically runs once a month.
  - Note that you also need to create a file named `mlc_config.json`.
- [arduino-lint][15]
  - Used with libraries published to the Arduino library manager to confirm that they meet the Arduino Library Spec rules.
  - This workflow as written does not work with `act` because it requires the `GITHUB_TOKEN` secret which `act` does not supply by default. This could be fixed with a little extra code and some local setup, but I don't find this necessary because the [arduino-lint tool][33] can be easily run directly from the command line:

    ```shell
    arduino-lint --verbose --compliance strict --library-manager update 
    ```

- [markdownlint][16]
  - Used to validate "clean" markdown code.
  - Note that you also need to create a file named `markdownlintconfig.json`
  - *I typically do not run this as an action,* but instead use the related [VSCode extension][21] to check my Markdown code.
- [build-dependent-repos][17].
  - Automatically trigger a compile-sketches action on repos that depend on a library that I updated.
  - This also uses a matrix strategy to define the list of repos that are triggered.

## The `act` Tool for Testing Workflows Locally

[`act`][1] uses [Docker][5] with customized [images][6] to create a local environment similar to the environment used by GitHub Actions. The default `act` image is intentionally [incomplete][7] to keep its size manageable. While it works well for many actions, there are issues that arise since the environments are not quite the same.

### `arduino-compile-sketches` Action

One problem occurs when using Arduino's [`compile-sketches`][8] action. Release [v1.1.0 of the `compile-sketches` action][10] changed the Python and related tools configuration to make the action incompatible with the default `act` image ([`catthehacker/ubuntu:act-latest`][9]).

In particular, the following messages will be appear in the `act` output when trying to use `compile-sketches` v1.1.0 and the `ubuntu:act-latest` image, and the job will fail:

``` text
| installing poetry from spec 'poetry==1.4.0'...
| ⚠️  File exists at
|     /root/.local/bin/poetry and points
|     to /root/.local/venv/bin/poetry,
|     not /root/.local/pipx/venvs/poetry
|     /bin/poetry. Not modifying.
|   installed package poetry 1.4.0, installed using Python 3.11.2
|     - poetry (symlink missing or pointing to unexpected location)
| done! ✨ 🌟 ✨
```

#### Fixing the `compile-sketches` Errors

To fix this problems, `/root/.local/bin` needs to be added to the shell's PATH so that the `poetry` executable can be found:

```yaml
    steps:                           # Existing code
      - uses: actions/checkout@main  # Existing code
# *** Add the code below ***
      - name: Update PATH if using nektos/act locally
        if: ${{ env.ACT }}
        run: |
          echo "/root/.local/bin" >> $GITHUB_PATH
```

By checking for the environment variable [`env.ACT`][11] first, the PATH update step is only run when using `act`.

Note that versions of the `ubuntu:act-latest` image published before 18-Sep-2023 have additional issues, but the latest published version only requires the above change (see [61][61-images] and [74][74]).

### Timeout and Rate-limiting Errors

When using `act`, I occasionally run into timeout and rate-limiting errors that aren't specific to the actions themselves. These errors can generally be cleared by waiting a few minutes and then re-running the action.

Examples of the errors:

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

### GitHub Actions Extension for VSCode and `act` Environment Variables

The [GitHub Actions][2] extension for VSCode does not recognize the `env.ACT` variable and flags an issue in the Problems window when you open an action YAML file for editing:

```text
Context access might be invalid: ACT
```

This is documented in several issues ([67][67], [61][61], [47][47]) and there does not appear to be a plan to fix it. Some issue comments claim that it is fixed, but it is not.

While the basic issue is slightly annoying, a more annoying aspect is that the message does not go away when you close the editor window. This causes the Problems window to become cluttered with the messages. The only way to clear the messages after closing the file editor is to disable/enable the extension or quit and restart VSCode.

### `act` Caching Reusable Workflows

The `act` tool caches external repos (i.e., GitHub repos that aren't the current repo under test). This can create issues when debugging reusable workflows. If you push a change to a reusable workflow stored in another repo, `act` will use a cached version of the workflow without the updates. This can make it appear that a fix didn't work even though it should have.

I work around this by deleting the cache directory any time I update the reusable workflows. The default cache location is `~/.cache/act`. My reusable workflows are stored in my `.github` repo on the `main` branch, so I run the following to delete the cache:

```bash
rm -rf ~/.cache/act/Andy4495-.github@main
```

This may be related to one of these issues: [1785][1785], [1912][1912], [1913][1913]. I haven't researched this any further, and there may be other solutions than just deleting the cache directory.

### Docker Setup With Windows WSL

When running `act` under Windows WSL, you may see an error similar to the following:

```text
Could not get auth config from docker config: error getting credentials - err: exec: "docker-credential-desktop.exe": executable file not found in $PATH, out: ''
```

This can be fixed by updating the file (within WSL) `~/.docker/config.json` and changing `credsStore` to `credStore`. You may need to restart Docker for this change to take effect.

### Matrix Strategy Issue with `act`

This [issue][2003] has been fixed with the release of `act` version [0.2.54][32]. More details can be found [here][31].

## References

- [`act` tool][1]
- GitHub [Actions][3]
- [Docker][5]
- Docker [images][6] for `act`
- [`compile-sketches`][8] action
- [markdown-link-check][18] action
- [arduino-lint][19] action
- [arduino-lint][32] command-line tool
- [markdownlint][20] action
- [Markdownlint extension][21] for VSCode
- [GitHub Actions][2] extension for VSCode

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
[13]: https://github.com/Andy4495/.github/blob/main/workflow-templates/arduino-compile-sketches.yml
[14]: https://github.com/Andy4495/.github/blob/main/workflow-templates/CheckMarkdownLinks.yml
[15]: https://github.com/Andy4495/.github/blob/main/workflow-templates/arduino-lint.yml
[16]: https://github.com/Andy4495/.github/blob/main/workflow-templates/markdownlint.yml
[17]: https://github.com/Andy4495/.github/blob/main/workflow-templates/build-dependent-repos.yml
[18]: https://github.com/gaurav-nelson/github-action-markdown-link-check
[19]: https://github.com/marketplace/actions/arduino-arduino-lint-action
[20]: https://github.com/marketplace/actions/markdownlint-cli
[21]: https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint
[24]: https://docs.github.com/en/actions/using-workflows/reusing-workflows
[25]: #fixing-the-compile-sketches-errors
[26]: https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
[27]: https://github.com/Andy4495/Template-Repo/blob/main/.github/workflows/arduino-compile-sketches.yml
[28]: https://github.com/Andy4495/Template-Repo/blob/main/.github/workflows/build-dependent-repos.yml
[29]: https://github.com/Andy4495/.github/tree/main/.github/workflows
[30]: ./extras/Actions_screen.jpg
[31]: https://github.com/Andy4495/act-workflow-test
[32]: https://github.com/nektos/act/releases/tag/v0.2.54
[33]: https://arduino.github.io/arduino-lint/1.2/
[47]: https://github.com/github/vscode-github-actions/issues/47
[61]: https://github.com/github/vscode-github-actions/issues/61
[67]: https://github.com/github/vscode-github-actions/issues/67
[61-images]: https://github.com/catthehacker/docker_images/issues/61
[74]: https://github.com/catthehacker/docker_images/pull/74
[1785]: https://github.com/nektos/act/issues/1785
[1912]: https://github.com/nektos/act/pull/1912
[1913]: https://github.com/nektos/act/pull/1913
[2003]: https://github.com/nektos/act/issues/2003
[100]: https://choosealicense.com/licenses/mit/
[101]: ./LICENSE.txt
[//]: # ([200]: https://github.com/Andy4495/GitHub-actions-and-act)

[//]: # (This is a way to hack a comment in Markdown. This will not be displayed when rendered.)
