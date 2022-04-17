---
author: matthias_luescher
author_profile: true
description: "Build a Debian based OS image from scratch, dispatch it to an embedded device and test it there - completely automated using GitHub Actions!"
comments: true
title: "Building and Testing OS Images with GitHub Actions"
---

Self-hosted Runner
------------------

![Mender Configure](/assets/images/blog/CICDRunnerConfig.png){:class="img-responsive"}

![Runner Setup](/assets/images/blog/CICDGitHubRunner.png){:class="img-responsive"}

![Debug Terminal](/assets/images/blog/CICDRunnerTerminal.png){:class="img-responsive"}

OS Build and Test Workflow
--------------------------

![Run Workflow](/assets/images/blog/CICDRunWorkflow.png){:class="img-responsive"}

![Workflow Setup](/assets/images/blog/CICDGitHubActionsOSWorkflow.png){:class="img-responsive"}

![Workflow Summary](/assets/images/blog/CICDWorkflowSummary.png){:class="img-responsive"}

``` python
def test_systemd_overall_status(host):
    cmd = host.run("systemctl is-system-running")
    assert cmd.rc == 0
    assert "running" in cmd.stdout
```

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

Conclusion
----------

