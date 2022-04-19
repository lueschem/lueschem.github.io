---
author: matthias_luescher
author_profile: true
description: "Build a Debian based OS image from scratch, dispatch it to an embedded device and test it there - completely automated using GitHub Actions!"
comments: true
title: "Building and Testing OS Images with GitHub Actions"
---

In this blog post I present a game changer for the future development of edi based project configurations (such as
[edi-pi](https://github.com/lueschem/edi-pi), [edi-var](https://github.com/lueschem/edi-var) and
[edi-cl](https://github.com/lueschem/edi-cl)). The following
two items will help me - and hopefully others - a lot to further improve the quality of edi and its project
configurations:

- A fully automated setup of a [_self-hosted GitHub actions runner_](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners)
that is capable of running _edi commands_.
- A [_GitHub actions workflow_](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
that builds a Debian based _OS from scratch_, _dispatches_ it to a device and _tests_ it.

Self-hosted Runner
------------------

The following sketch depicts the setup of the self-hosted runner:

![Runner Setup](/assets/images/blog/CICDGitHubRunner.png){:class="img-responsive"}

As a prerequisite a GitOps ready Raspberry Pi 4 got flashed
(see [edi-pi](https://github.com/lueschem/edi-pi), command `sudo edi -v image create pi4-bullseye-arm64-gitops.yml`).
However, this time we do not turn the Raspberry Pi into a [kiosk terminal](/Surprisingly-Easy-IoT-Device-Management/).
Instead, we choose to go with a different configuration and enter the following key/value pairs using the 
[Mender configure](https://docs.mender.io/add-ons/configure) add on (step &#9312;):

![Mender Configure](/assets/images/blog/CICDRunnerConfig.png){:class="img-responsive"}

Accordingly - after receiving the config artifact &#9313; - the device will clone the
[playbook edi-gh-actions-runner-playbook](https://github.com/lueschem/edi-gh-actions-runner-playbook) &#9314;
(main branch). The playbook itself makes use of two roles &#9315;:

- [github_actions_runner](https://github.com/MonolithProjects/ansible-github_actions_runner): A role written by
Michal Muransky that installs a GitHub Actions runner.
- [edi_installer](https://github.com/lueschem/edi_installer): A role that installs edi including its prerequisites.

The roles will then fetch the runner binaries from GitHub &#9316; and a lot of Debian packages from the upstream Debian
repositories and the [edi PackageCloud repository](https://packagecloud.io/get-edi/debian) &#9317;.

If something should go wrong, then the Mender [remote terminal](https://docs.mender.io/add-ons/remote-terminal) add-on
might be useful:

![Debug Terminal](/assets/images/blog/CICDRunnerTerminal.png){:class="img-responsive"}

However - once everything is properly setup - nothing should go wrong anymore and the self-hosted runner will connect
to the edi-ci-cd GitHub repository:

![Registered Runner](/assets/images/blog/CICDRegisteredRunner.png){:class="img-responsive"}

Please note that the edi-ci-cd project is a private repository according to the
[GitHub security recommendation](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners#self-hosted-runner-security).
In order to still make the setup transparent, I provide the cloned repository
[edi-ci-cd-public](https://github.com/lueschem/edi-ci-cd-public).

As a bonus even a complete OS update on the runner is possible and the whole setup will get re-applied thanks to
[this commit](https://github.com/lueschem/edi-pi/commit/a01b1fe9832f5de46687aefcfcce05676caf66a1).

OS Build and Test Workflow
--------------------------

Using the self-hosted runner we are now able to trigger our
[OS build, dispatch and test workflow](https://github.com/lueschem/edi-ci-cd-public/blob/main/.github/workflows/os-deployment.yml):

![Workflow Setup](/assets/images/blog/CICDGitHubActionsOSWorkflow.png){:class="img-responsive"}

As a first step (the only manual step) we tell the workflow which configuration (here `iot-gate-imx8-bullseye-arm64.yml`)
of which repository (here `edi-cl` using the `main` branch) shall be applied to our chosen device (referenced by the
Mender device id) &#9312;:

![Run Workflow](/assets/images/blog/CICDRunWorkflow.png){:class="img-responsive"}

The job will then be sent to our self-hosted runner &#9313;. The runner will check out the source code of edi-ci-cd and
the specified edi project configuration (here [edi-cl](https://github.com/lueschem/edi-cl)) &#9314;. edi will then take
care of building the specified OS artifact (here `iot-gate-imx8-bullseye-arm64.yml`) from scratch. During this
process a lot of Debian packages will get fetched from various Debian repositories &#9315;. Using the
[Mender REST API](https://github.com/lueschem/edi-ci-cd-public/blob/main/mender-api) the resulting Mender OS artifact
will get uploaded to Mender &#9316; and then dispatched to our chosen device &#9317;.

After our test device got successfully updated, we run some tests that are based on [pytest](https://www.pytest.org/)
and [Testinfra](https://testinfra.readthedocs.io/) &#9318;.

Testinfra is a pretty powerful pytest plugin and - as an example - the following test makes sure that all systemd
services are running properly on our test device:

``` python
def test_systemd_overall_status(host):
    cmd = host.run("systemctl is-system-running")
    assert cmd.rc == 0
    assert "running" in cmd.stdout
```

Another test checks that our root device got properly resized and comes with the expected mount points:

``` python
import re
import pytest


def test_root_device(host):
    cmd = host.run("df / --output=pcent")
    assert cmd.rc == 0
    match = re.search(r"(\d{1,3})%", cmd.stdout)
    assert match
    # if the usage is below 50% then the root device got properly resized
    assert int(match.group(1)) < 50


def test_resize_completion(host):
    assert host.file("/etc/edi-resize-rootfs.done").exists


@pytest.mark.parametrize("mountpoint", ["/", "/data", "/boot/firmware", ])
def test_mountpoints(host, mountpoint):
    assert host.mount_point(mountpoint).exists
```

The result of the whole workflow gets nicely presented on GitHub &#9319;:

![Workflow Summary](/assets/images/blog/CICDWorkflowSummary.png){:class="img-responsive"}

The attentive reader will now ask the legitimate question: Where does the workflow know the various credentials from?

Using GitHub Actions you have the possibility to store [encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets).

I have summarized the required secrets [here](https://github.com/lueschem/edi-ci-cd-public/blob/main/README.md).

Conclusion
----------

After having completed all this automation, I can now enjoy the flow of the bike trail while my edi OS workflow is
running. If something should go wrong, then a GitHub e-mail will make me aware that I have to fix something once I am
back home.

The whole stuff presented here should be helpful also for other edi based projects. Apart from
[GitHub Actions](https://github.com/features/actions) I did not introduce any new technology. Instead, I was re-using
already familiar tools ([Ansible](https://www.ansible.com/), [Mender](https://mender.io/),
[Debian](https://www.debian.org/), [pytest](https://www.pytest.org/) and [edi](https://www.get-edi.io)) and hardware.

It should also be fairly easy to use [Jenkins](https://www.jenkins.io/) or [GitLab](https://about.gitlab.com/) instead
of [GitHub Actions](https://github.com/features/actions). 