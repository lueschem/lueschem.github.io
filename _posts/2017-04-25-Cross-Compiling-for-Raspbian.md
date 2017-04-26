container setup
---------------

compile hello world
-------------------

openssl
-------

apt-get source libssl1.0.0
cd openssl-1.0.1t

export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf

Finished running lintian.

real	4m45.315s
user	3m53.980s
sys	0m28.184s
lueschem@raspbian-cross-01:~/edi-workspace/openssl-1.0.1t$

man-db
------

apt-get source man-db

lueschem@raspbian-cross-01:~/edi-workspace$ sudo apt-get build-dep -a armhf man-db
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages have unmet dependencies:
 zlib1g-dev:armhf : Depends: zlib1g:armhf (= 1:1.2.8.dfsg-2) but it is not going to be installed
E: Build-dependencies for man-db could not be satisfied.

lueschem@raspbian-cross-01:~/edi-workspace$ apt-cache policy zlib1g
zlib1g:
  Installed: 1:1.2.8.dfsg-2+b1
  Candidate: 1:1.2.8.dfsg-2+b1
  Version table:
 *** 1:1.2.8.dfsg-2+b1 0
        500 http://httpredir.debian.org/debian/ jessie/main amd64 Packages
        100 /var/lib/dpkg/status
lueschem@raspbian-cross-01:~/edi-workspace$

apt-get source zlib1g

cd zlib-1.2.8.dfsg/

dch --bin-nmu --distribution unstable

export DEB_BUILD_OPTIONS=nocheck; time debuild -us -uc -aarmhf -B

sudo dpkg -i ../zlib1g_1.2.8.dfsg-2+b1_armhf.deb

sudo dpkg -i ../zlib1g-dev_1.2.8.dfsg-2+b1_armhf.deb

sudo apt-get build-dep -a armhf man-db

cd ../man-db-2.7.0.2/

debuild -us -uc -aarmhf

scp ../man-db_2.7.0.2-5_armhf.deb pi@raspberrypi:

ssh pi@raspberrypi

sudo dpkg -i man-db_2.7.0.2-5_armhf.deb

compile and install kernel
--------------------------

explanations
------------





