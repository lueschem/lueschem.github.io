---
author: matthias_luescher
author_profile: true
description: "edi makes it very easy to document software releases! The setup is highly configurable so that it will match the needs of your project."
comments: true
title: "Documenting Releases"
---

At some point a new software release gets finished - and the development team is sure that it is the greatest ever
compilation with plenty of new features and zero bugs. Other teams will then start to ask for details about the new
release and where they can find the release notes. They also want to know whether "their" issues got fixed and if they
can finally introduce feature X and Y that got sold to the customer three months ago.
The answer of the development team will be: "Oh, wait a second - we will go through the 
changelog, the wiki and the issue tracker and write the release notes soon - but first we have to implement some new
features." Of course this is a manual, boring job and nobody will step up and complete it in the near future.

Well - this was the past. With [edi](https://www.get-edi.io) you can now generate release notes automatically.

![Documentation](/assets/images/blog/documentation.png){:class="img-responsive"}

In a nutshell, the boring job gets accomplished by some exciting pieces of software:

1. edi is harvesting the [changelog](https://manpages.debian.org/testing/dpkg-dev/deb-changelog.5.en.html)
   of each individual Debian package that makes it into the release.
2. edi converts the changelog data into Python data structures.
3. edi applies customizable regular expressions to the changelog entries (you can e.g. turn tagged issues into
   hyperlinks).
4. edi feeds the Python data structures into [Jinja2](https://jinja.palletsprojects.com/) templates.
5. The Jinja2 templates render e.g. [reStructuredText](https://en.wikipedia.org/wiki/ReStructuredText) text files.
6. [Sphinx](https://www.sphinx-doc.org/) can finally turn the text files into a web page or a nice pdf document.

The whole setup is highly configurable according to the needs of your project. Please take a look at the
[edi documentation](https://docs.get-edi.io) for more details.

For an example on how you can integrate it into your build pipeline 
please consult the [edi-pi project](https://github.com/lueschem/edi-pi/).

Here is a [sample document](/assets/pdfs/pi2-buster-armhf.pdf) that got created using the edi-pi project.

The document also gives more visibility to the incredible work that the Debian community is doing: Did you count how
many [common vulnerabilities and exposures](https://cve.mitre.org/) they fixed since December 1, 2019?