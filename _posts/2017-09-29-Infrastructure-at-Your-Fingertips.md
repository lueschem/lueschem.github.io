---
author: matthias_luescher
author_profile: true
description: "GitHub is a good starting point for your open source project. However, there is more to be discovered as your project is getting more mature."
comments: true
---

edi helps you to build development infrastructure but in turn it also uses plenty of
it. This blog post is about the latter topic: I am still fascinated
how easily you can setup a powerful and worthwhile development environment for an open
source project.

The following walkthrough shall help you to get started even more quickly than I did.

Here is the "big picture":

![infrastructure](/assets/images/blog/edi_infrastructure.png){:class="img-responsive"}

Source Code Management
----------------------

Pretty much the first thing I did when I started the edi project was a `git init`.
Probably the second command was a `git push ...` to [GitLab](https://www.gitlab.com). Although
I like GitLab a lot I later migrated to [GitHub](https://github.com/) due to
personal preferences.

Web Page
--------

After some initial development in "stealth mode" I decided that I want to
create a web page for the project. After a short evaluation phase I decided to go with a
static web page hosted on [GitHub Pages](https://pages.github.com/). As I am not a designer
nor a web developer I was very glad to find a nice [Jekyll](https://jekyllrb.com/) theme for
my page ([Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/)). With this
initial aid the web page was up and running quickly. Since GitHub pages serves static pages
you have to kind of delegate the possibility of adding comments to your page to another
web service. After another short evaluation I decided to go with [Disqus](https://disqus.com/).

[Ansible](https://www.ansible.com) was kind of my signpost on how to document a Python
project: The documentation is close to the source code and written in
[restructured text](https://en.wikipedia.org/wiki/ReStructuredText). Although the blog post
you  are reading right now is written in [markdown](https://en.wikipedia.org/wiki/Markdown)
I prefer to use restructured text as soon as the documents get bigger. The tool
[Sphinx](http://www.sphinx-doc.org) then turns my documentation into a web page and now
we are ready to use a first service outside of the GitHub universe:
[Read the Docs](https://readthedocs.org/) serves the
[edi documentation](https://docs.get-edi.io).

To register https://www.get-edi.io and https://docs.get-edi.io I visited the web page of
[Cyon](https://www.cyon.ch/) and bought and configured my domain.

Analytics
---------

Maybe you remember the good old times where you used to put a "counter" on your web page.
Nowadays [Google Analytics](https://analytics.google.com) takes over this part. It is a
matter of a few minutes to activate Google Analytics on your Read the Docs documentation and
on GitHub pages. With Google Analytics you gain plenty of insights into how people access and
use your content.

Software Distribution
---------------------

All of a sudden we wanted to use edi at work and therefore I was looking for a solution
that makes it convenient for my coworkers to download and install edi. Since edi is currently
Ubuntu centric and as we are also using Ubuntu at work, the choice was obvious:
[Launchpad](https://launchpad.net/) offers a build server for Debian packages. After a
successful build edi can be installed from the
[personal package archive](https://launchpad.net/~m-luescher/+archive/ubuntu/edi-snapshots)
hosted on launchpad. Launchpad does not look fancy but it provides a rock solid service
with signed uploads and draconic build checks.

Continuous Integration
----------------------

As the code base of edi keeps growing quality assurance becomes more and more important.
After yet another short evaluation I decided to go with
[Travis CI](https://travis-ci.org/). Therefore I added a
[`.travis.yml`](https://github.com/lueschem/edi/blob/master/.travis.yml) to the edi project
and registered it at Travis CI. Since edi is not a pure Python project (it has many non
Python dependencies) the continuous integration took a bit more time. Currently Travis CI
only supports Ubuntu 14.04 but edi requires at least Ubuntu 16.04. This is where
[docker](https://www.docker.com/) comes to the rescue: Travis CI allows me to pull the latest
Ubuntu 16.04 image. With a few instructions I can easily install additional dependencies from
an Ubuntu apt server and then start testing within the docker container.

Drawings
--------

I am a longtime user and fan of [Inkscape](https://inkscape.org/) for
freestyle drawings (like the edi logo). For drawings where special shapes (e.g. UML) are
needed I was searching for an appropriate tool. Not long ago I started to use
[draw.io](https://about.draw.io/) and I really like this tool (the "big picture" above was
my second draw.io drawing).


Conclusion
----------

The tools and services I have described above are a pleasure to work with. Apart from the
DNS registration all is free for open source projects. The nice thing is that you can also
use the above setup for a company that produces closed source software. In this case it is
no longer free but you can profit from a scalable infrastructure that is very
refined.

Although many different parties are involved in the above setup, the overall integration
is great and I frequently discover new goodies:

- The result of Travis CI can be displayed on Read the Docs.
- draw.io can store drawings directly to GitHub.
- GitHub displays the Travis CI results per commit.
- Read the Docs gets automatically notified by GitHub.
- Multiple components are using single sign on.
- G-Mail displays links to issues directly on the overview page.
- You can directly share this blog post on Google+ and LinkedIn.

After all, the convenience of the above setup is difficult to top in a self hosted company
environment.

