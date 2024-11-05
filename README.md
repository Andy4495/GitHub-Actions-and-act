# GitHub Actions and `act`

[![Check Markdown Links](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/check-links.yml/badge.svg)](https://github.com/Andy4495/GitHub-Actions-and-act/actions/workflows/check-links.yml)

I use GitHub [Actions][3] workflows to run automated builds, tests, and checks on my repositories.

I develop and test my workflows before pushing them to GitHub with the [`act`][1] tool created by GitHub user [nektos][4].

## GitHub Actions

This section documents some conventions I use with my actions, along with examples of workflows that I use across my repos. See GitHub's comprehensive [documentation][3] for more details on action terminology and workflow syntax.

### GitHub Reusable Workflows

GitHub Actions have a feature called [reusable workflows][24] which essentially allows one workflow to call another workflow. This makes it convenient to have a "base" workflow (referred to as a "called workflow") in a central location that does most of the work, and a per-repository workflow (referred to as a "caller workflow") which configures the settings needed to run the base workflow.

Taking advantage of reusable workflows becomes especially useful when you need to make a change to the functionality of an action that is used in multiple repos (like when I had to fix the issue with `act` as described [below][25]). Instead of having to make the change in every repo which uses the action, all that needs to change is the single base workflow. See [.github/workflows][29] for the reusable workflows that my repos call.

### Workflow Structure

#### Trigger on `workflow_dispatch`

I have a `workflow_dispatch` trigger (including an optional input string) on all my workflows so that I can trigger them manually from the [repo Actions screen][30]:

```yaml
  workflow_dispatch:
    inputs:
      message:
        description: Message to display in job summary
        required: false
        type: string
```

#### Matrix Strategy

For any workflow that can trigger multiple job runs based on different combinations of variables (e.g., compiling across multiple platforms), I use a [matrix strategy][26]. Even if the repo only needs to be compiled for one platform, I still use a matrix strategy for consistency with my other repos and my reusable workflows.

Example for [compiling arduino sketches][27] across multiple platforms:

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
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/refs/heads/main/json/package_energia_minimal_msp430_index.json'
          - arch: msp-F5529
            fqbn: 'energia:msp430:MSP-EXP430F5529LP'
            platform-name: 'energia:msp430'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/refs/heads/main/json/package_energia_minimal_msp430_index.json'
          - arch: msp432
            fqbn: 'energia:msp432:MSP-EXP432P401R'
            platform-name: 'energia:msp432'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/refs/heads/main/json/package_energia_minimal_msp432_index.json'
          - arch: tivac
            fqbn: 'energia:tivac:EK-TM4C123GXL'
            platform-name: 'energia:tivac'
            platform-sourceurl: 'https://raw.githubusercontent.com/Andy4495/TI_Platform_Cores_For_Arduino/refs/heads/main/json/package_energia_minimal_tiva_index.json'
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
- [check-links][14]
  - Checks for dead links in Markdown files using the [lychee tool][75]. Automatically runs once a month.
- [arduino-lint][15]
  - Used with libraries published to the Arduino library manager to validate that they conform to the Arduino Library Spec rules.
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

[`act`][1] uses [Docker][5] or [Podman][76] with customized [images][6] to create a local environment similar to the environment used by GitHub Actions. The default `act` image is intentionally [incomplete][7] to keep its size manageable. While it works well for many actions, there are issues that arise since the environments are not quite the same.

### Python Installation Error When Running `compile-sketches` Action

You may see the following error when running the `compile-sketches` job:

```text
::error::rm: cannot remove '/opt/hostedtoolcache/Python/3.11.2': Directory not empty
```

This only seems to happen when a new cache is being configured on the first run of matrix builds where one matrix run may be populating the cache at the same time a parallel matrix run is trying to access the cache.

To fix this, delete the relevant cache:

```shell
rm -r ~/.cache/act/actions-setup-python@v5   
```

And then run `act` with a single matrix job before running the full matrix. For example:

```shell
act --matrix arch:msp432
```

While a new action cache architecture has been implemented in `act`, it does not appear to fix this particular race condition, but you can also try it if the problem persists:

```shell
act --use-new-action-cache
```

### `arduino-compile-sketches` Action

Release [v1.1.0 of the `compile-sketches` action][10] changed the Python and related tools configuration to make the action incompatible with the default `act` image ([`catthehacker/ubuntu:act-latest`][9]).

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

This may be related to one of these issues: [1785][1785], [1912][1912], [1913][1913]. However, the new action cache implemented in `act` does not appear to fix this issue. Also, see `act` issue [2419][77].

### Docker Setup With Windows WSL

When running `act` under Windows WSL, you may see an error similar to the following:

```text
Could not get auth config from docker config: error getting credentials - err: exec: "docker-credential-desktop.exe": executable file not found in $PATH, out: ''
```

This can be fixed by updating the file (within WSL) `~/.docker/config.json` and changing `credsStore` to `credStore`. You may need to restart Docker for this change to take effect.

### Using Podman Instead of Docker on Linux

While Docker Desktop is easy to install and update on MacOS and Windows, it is another matter trying to get it to work on Linux. I use Ubuntu, but the complications are common to all flavors of Linux.

Installing Docker Desktop (or just Docker Engine) requires a multi-step process, not a simple `apt install docker`. Upgrading to a newer version also requires a separate download and install and cannot be done directly from the app. Making matters worse is that Docker Desktop does not work with Ubuntu 24.04 LTS (despite a [comment to the contrary][78]). Maybe it works now, but I have given up on going through the pain of installing only to find out it doesn't work.

I therefore recommend using [Podman][76] as your containerization tool on Linux. To [install Podman][80] on Ubuntu, run:

```shell
sudo apt update
sudo apt install podman
```

To configure `act` to use Podman, run the following steps after installing Podman:

```shell
sudo systemctl disable --now podman.socket
```

Run the following as the user that will be running `act`; do not `sudo`:

```shell
systemctl enable --now --user podman podman.socket 
```

And finally, set the `DOCKER_HOST` environment variable. This would normally done in your shell startup file (e.g. `~/.bashrc`), but could also be done manually before you run `act`:

```shell
export DOCKER_HOST=unix:///run/user/1000/podman/podman.sock
```

The file path may be different on your system. You can find the correct path by running `podman info` and looking at the `remoteSocket` value. Be sure to prepend the file path with `unix://`.

I have found that Podman executes actions a little slower than Docker. It is still quite usable, and probably wouldn't be noticed if I didn't have experience running the actions in Docker. This slower execution is likely caused by the overhead of running in userspace instead of the kernel (rootless mode) and from the pasta network driver.

### Ubuntu 24.04 WiFi Issues

I have run into a fairly repeatable issue with Ubuntu completely freezing (cursor stops moving, reboot required to clear the issue) when running `act`. This issue happens with both Podman and Docker. I only see the problem when using WiFi; when connected to ethernet (WiFi disabled), I cannot reproduce the issue.

I have only seen the issue with my `compile-sketches` action. By default, this action runs a matrix build with two or more containers running in parallel, although I have seen the problem less frequently when forcing the action to run a single matrix entry (with the `--matrix`) option, so that only a single container is running.

This problem appears to be specific to Ubuntu 24.04; I have not seen it on similarly configured laptop running Ubuntu 22.04.

At this point, the only thing I can recommend is to run matrix builds one at a time, by manually specifying the matrix combination using the `--matrix` option. For example, when running `compile-sketches` on my [SWI2C][79] library, run the following commands:

```shell
act -j compile-sketches --use-new-action-cache --matrix arch:avr
act -j compile-sketches --use-new-action-cache --matrix arch:msp
act -j compile-sketches --use-new-action-cache --matrix arch:msp432
act -j compile-sketches --use-new-action-cache --matrix arch:tivac
act -j compile-sketches --use-new-action-cache --matrix arch:stm32
act -j compile-sketches --use-new-action-cache --matrix arch:esp8266
act -j compile-sketches --use-new-action-cache --matrix arch:esp32
```

### Matrix Strategy Issue with `act`

This [issue][2003] has been fixed with the release of `act` version [0.2.54][32]. More details can be found [here][31].

## References

- [`act` tool][1]
- GitHub [Actions][3]
- [Docker][5]
- Docker [images][6] for `act`
- [Podman][76]
- [`compile-sketches`][8] action
- [check-links][18] action
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
[14]: https://github.com/Andy4495/.github/blob/main/workflow-templates/check-links.yml
[15]: https://github.com/Andy4495/.github/blob/main/workflow-templates/arduino-lint.yml
[16]: https://github.com/Andy4495/.github/blob/main/workflow-templates/markdownlint.yml
[17]: https://github.com/Andy4495/.github/blob/main/workflow-templates/build-dependent-repos.yml
[18]: https://github.com/lycheeverse/lychee-action
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
[75]: https://github.com/lycheeverse/lychee
[76]: https://podman.io/
[77]: https://github.com/nektos/act/issues/2419
[78]: https://forums.docker.com/t/cannot-get-docker-desktop-to-run-on-ubuntu-24-04/141004/62
[79]: https://github.com/Andy4495/SWI2C
[80]: https://podman.io/docs/installation
[1785]: https://github.com/nektos/act/issues/1785
[1912]: https://github.com/nektos/act/pull/1912
[1913]: https://github.com/nektos/act/pull/1913
[2003]: https://github.com/nektos/act/issues/2003
[100]: https://choosealicense.com/licenses/mit/
[101]: ./LICENSE.txt
[//]: # ([200]: https://github.com/Andy4495/GitHub-actions-and-act)

[//]: # (This is a way to hack a comment in Markdown. This will not be displayed when rendered.)
