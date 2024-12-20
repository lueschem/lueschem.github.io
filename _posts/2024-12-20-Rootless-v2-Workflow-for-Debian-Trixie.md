---
author: matthias_luescher
author_profile: true
description: "All edi project configurations have been upgraded to the rootless v2 workflow. The OCI image based workflow will make the development process even more convenient."
comments: true
title: "Rootless v2 Workflow for Debian Trixie"
---

As [announced earlier this year](/Rootless-Creation-of-Debian-OS-and-OCI-Images/) `edi` has adopted a new workflow.
In the meantime the whole fleet of devices supported by edi project configurations can benefit from this new workflow:

[![Devices Supported by Debian Trixie](/assets/images/blog/debian_trixie.png){:class="img-responsive"}](/assets/images/blog/debian_trixie.png)

- Raspberry Pi 2, 3, 4, 5: [Debian trixie, edi-pi](https://github.com/lueschem/edi-pi)
- Compulab IOT-GATE-iMX8: [Debian trixie, edi-cl](https://github.com/lueschem/edi-cl)
- Variscite VAR-SOM-MX8M-NANO: [Debian trixie, edi-var](https://github.com/lueschem/edi-var)

The most notable change is the switch from LXD to Buildah and Podman (or Docker if preferred):

[![Workflow v2](/assets/images/blog/workflow_v2_artifacts.png){:class="img-responsive"}](/assets/images/blog/workflow_v2_artifacts.png)

All project configurations are now based on Debian trixie which is currently in a state called "testing". Due to the
lack of timely security updates Debian trixie is not yet suitable for a rollout, but it is already fairly stable and a
good match for starting a new development based on a very recent software stack. If you want to play it safe, then all
the above edi project configurations have a branch called `debian_bookworm` that delivers Debian bookworm based
artifacts by making use of the v1 workflow. `edi` will of course keep supporting the v1 workflow as longevity is a key
topic for embedded devices.

The new v2 workflow uses a number of new tools that must be provided in a recent version. Therefore, all trixie project
configurations will cowardly refuse to work if you do not have at least Debian bookworm or Ubuntu 24.04 as host
operating system.

The OS tweaking is done using a [single Ansible playbook](https://github.com/lueschem/edi-pi/blob/master/plugins/playbooks/os_setup/main.yml)
that can be extended in a modular manner using Ansible collections such as
[debian_setup](https://github.com/lueschem/debian_setup).

The Mender integration has been upgraded to the [modular C++ client](https://github.com/mendersoftware/mender).

# Outlook

The v2 workflow is the recommended setup for any new project. Also existing projects can be switched over to the v2
workflow within a reasonable amount of time. Even complex image scenarios involving encrypted disks can be done in a
completely rootless manner thanks to LUKS2. Distrobox will definitely improve the developer experience while the OCI
images will simplify the CI/CD pipeline. The integration of Ansible collections will increase reusability.