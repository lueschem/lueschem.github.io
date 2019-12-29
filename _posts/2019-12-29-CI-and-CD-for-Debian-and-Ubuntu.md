---
author: matthias_luescher
author_profile: true
description: "Releasing edi for two Debian and three Ubuntu releases usually took about two hours of manual work. Now I cut it down to 5 minutes thanks to some additional automation involving GitHub, Travis CI, packagecloud and launchpad."
comments: true
title: "CI and CD for Debian and Ubuntu"
---

Developing some new features for edi is the fun part. Releasing it for two Debian and three Ubuntu releases
is the boring part and it took me around two hours to do it properly for each new release. As I do not like
repetitive tedious tasks that might even have a negative impact on quality if some steps are not properly executed I
decided to automate the last mile and implement continuous delivery (CD) for edi.

With the setup below generating a new release for Ubuntu Xenial, Bionic and Eoan and Debian stretch and buster
is now a matter of 5 minutes:

![pipeline](/assets/images/blog/edi_ci_cd.png){:class="img-responsive"}

Here are the important building blocks:

GitHub
------

[GitHub](https://github.com/lueschem/edi) was already my source code repository hosting of choice. However, my
branching setup was too simple for the continuous delivery pipeline and therefore I decided to introduce
[git-flow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow). The _master_ branch
now has as special position: Each time I do a merge to it a new release will get generated.
[Webhooks](https://developer.github.com/webhooks/) will trigger the update of the [documentation](https://docs.get-edi.io/en/latest/)
on [Read the Docs](https://readthedocs.org/) and the package builds for several Debian and Ubuntu releases on
[Travis CI](https://travis-ci.org/lueschem/edi).


Travis CI
---------

On [Travis CI](https://travis-ci.org) I make use of [Docker containers](https://docs.travis-ci.com/user/docker/) to build
my packages in the proper Debian and Ubuntu environments. This also involves [pytest](https://docs.pytest.org/en/latest/)
unit testing that currently covers about 70% of the code. The [Travis CI file](https://github.com/lueschem/edi/blob/master/.travis.yml)
looks pretty simple but some additional work gets done within a
[bash script](https://github.com/lueschem/edi/blob/master/travis/travis-build).
Finally the resulting artifacts get pushed to [launchpad](https://launchpad.net/) and [packagecloud](https://packagecloud.io/)
when a build was successful on the _master_ branch.

launchpad and packagecloud
--------------------------

Since 2016 I was pushing Ubuntu packages to my [launchpad ppa repository](https://launchpad.net/~m-luescher/+archive/ubuntu/edi-snapshots).
To avoid any breaking changes for the edi users I decided to stick to launchpad for the Ubuntu packages. Travis CI would support
[deployments to launchpad](https://docs.travis-ci.com/user/deployment/launchpad/) with a tailored plugin. As I was used to
[dput](http://manpages.ubuntu.com/manpages/bionic/man1/dput.1.html) I chose another path. From within the
[bash script](https://github.com/lueschem/edi/blob/master/travis/travis-build) I directly do the upload to launchpad using dput
(after a while I figured out that the default ftp upload probably failed due to
[this](https://blog.travis-ci.com/2018-07-23-the-tale-of-ftp-at-travis-ci) and that I should better use
[sftp](https://github.com/lueschem/edi/blob/master/travis/dput.cf) instead).

For the packages targeting Debian I was using a custom apt repository hosted on GitHub. This was not really suitable for CD and therefore
I decided to switch the Debian apt repository hosting to [packagecloud](https://packagecloud.io/). The package deployment to packagecloud
is straight forward using a tailored [Travis CI plugin](https://docs.travis-ci.com/user/deployment/packagecloud/).

Conclusion
----------

For the edi users it is now very simple and convenient to
[install the latest version](https://docs.get-edi.io/en/latest/getting_started.html#installing-edi-from-the-archive) from the 
[ppa](https://launchpad.net/~m-luescher/+archive/ubuntu/edi-snapshots) (Ubuntu) or
[packagecloud](https://packagecloud.io/get-edi/debian) (Debian). The obtained packages should have a consistent and predictable
quality because their build is now fully automated.

Of course I also enjoy the improved setup because releasing a new edi version now takes me five minutes instead of two hours!


