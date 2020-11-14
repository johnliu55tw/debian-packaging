# Add an executable script to an existing Ubuntu package

This document will guide you to:

- Prepare your system and configure Launchpad

- Retrieve the source code of an existing Ubuntu package

- Add a "testing.sh" script to the package and install it to `/usr/bin/`.

- Add a "post-install" script

- Re-package it as a `deb`

- Upload and host it using your PPA on Launchpad so everyone can install it

I will be using `hello` as an example for re-packaging. I am using Ubuntu 20.04
on a VM, but will give it a try using Docker container.

## Preparation

### Install and configure required packages

Make sure you have all the required software installed:

```bash
$ sudo apt install gnupg pbuilder ubuntu-dev-tools apt-file debhelper
```

Then you need to set up `pbuilder`. `pbuilder` allows you to build packages
locally on your machine. It serves a couple of purposes:

- The build will be done in a minimal and clean environment. This helps you
make sure your builds succeed in a reproducible way, but without modifying your
local system

- There is no need to install all necessary build dependencies locally

- You can set up multiple instances for various Ubuntu and Debian releases

To set `pbuilder` up:
```bash
$ pbuilder-dist focal create
```

### Generate your GPG key

Create a GPG key for signing your packages. Running the command
and follow the instruction to generate one:

```bash
$ gpg --gen-key
```

then you should see a message similar to this:

```
pub   rsa3072 2020-11-14 [SC] [expires: 2022-11-14]
      2734D5090EEC923C2FAB20373EAFB4567F851F68
uid                      John Liu <johnliu55tw@gmail.com>
sub   rsa3072 2020-11-14 [E] [expires: 2022-11-14
```

> **Note:** You can also run `gpg --full-generate-key` to fine-tune the key
> (encryption algorithm, expiration of the key, ...).

### Create Launchpad account and import your GPG key

You will need a [Launchpad](https://launchpad.net/+login) account to create
PPA on Launchpad. Click the link to register if you don't have one. After you
got an account, upload your GPG key to Launchpad.

Retrieve the **fingerprint** of your GPG key with the following command:

```bash
$ gpg --fingerprint <Your email>
```

It will print out the GPG fingerprint similar to the above:

```
pub   rsa3072 2020-11-14 [SC] [expires: 2022-11-14]
      2734 D509 0EEC 923C 2FAB  2037 3EAF B456 7F85 1F68
uid           [ultimate] John Liu <johnliu55tw@gmail.com>
sub   rsa3072 2020-11-14 [E] [expires: 2022-11-14]
```

The **last eight hexadecimal digits** of the second line is the key ID, in my
case it's `7F851F68`. Run this command to submit your key to Ubuntu keyserver:

```bash
$ gpg --keyserver keyserver.ubuntu.com --send-keys <Key ID> 
```

In my case the command will be:

```bash
$ gpg --keyserver keyserver.ubuntu.com --send-keys 7F851F68
```

Then head to https://launchpad.net/~/+editpgpkeys and copy the
"Key fingerprint" into the text box, in my case it's
`2734 D509 0EEC 923C 2FAB  2037 3EAF B456 7F85 1F68`. Now click "Import Key",
and you should see a prompt on the page similar to this:

> A message has been sent to `johnliu55tw@gmail.com`,
> encrypted with the key
> `3072R/2734D5090EEC923C2FAB20373EAFB4567F851F68`. To
> confirm the key is yours, decrypt the message and follow the
> link inside.

Launchpad will send an email to you with message encrypted by the Key you just
uploaded. You must descrypt the message in order to finish the import process.
If your email client doesn't decrypt it for you automatically, we can decrypt
it using `gpg` command line tool. Copy the message started from
`-----BEGIN PGP MESSAGE-----` to `-----END PGP MESSAGE-----` (including these
two lines) into a text file (let's say `msg.txt`) and run the following
command:

```bash
$ gpg -d msg.txt
```

Then you will see the decrypted message shows on the screen. Follow the
instruction to finish the import process.

### Create PPA on Launchpad

After you imported your key, you can create a PPA now. Head to
[https://launchpad.net/~](https://launchpad.net/~) and click
"Create a new PPA". Enters URL and Display Name, then click "Activate".

> **Note:** If you deleted a PPA, you will have to wait up to an hour before
> you can recreate a PPA with the same name.

## Retrieve the source code of a package

First, you need to know the name of the package you want to modify. You could
use [Ubuntu Package Search](https://packages.ubuntu.com/) to search for it.
I will use [hello](https://packages.ubuntu.com/focal/hello), an example package
based on GNU hello, as an example.

The ubuntu-dev-tools has a tool called `pull-lp-source` that we could
use to grab the source code for the "hello" package:

> **Note:** The command will pull multiple files and store into current working
> direction. I would recommend creating a new directory and `cd` into it
> before you run the command.

```bash
$ pull-lp-source hello focal
```

Now you should see several files appeared:

```bash
$ ls -l
drwxrwxr-x 13 johnliu johnliu   4096 Nov 14 01:09 hello-2.10
-rw-rw-r--  1 johnliu johnliu   6560 Nov 13 23:37 hello_2.10-2ubuntu2.debian.tar.xz
-rw-rw-r--  1 johnliu johnliu   1847 Nov 13 23:37 hello_2.10-2ubuntu2.dsc
-rw-rw-r--  1 johnliu johnliu 725946 Nov 13 23:32 hello_2.10.orig.tar.gz
```

The source code locates in folder `hello-2.10`. `cd` into it and let's do some
changes.

## Modify the package

### Adding script `testing.sh` using quilt

First, we need to use `quilt` to create a new patch:

```bash
$ quilt new johnliu.patch
```
where `johnliu.patch` is the name of the patch.

Because we're adding a new file `testing.sh` instead of changing existed one,
we need to **create an empty file first** so `quilt` can track it:

```bash
$ touch testing.sh
```

Now adding the file to quilt:
```bash
$ quilt add testing.sh
```

Now you can change the file. I will make the script simply echo my name:
```
#!/bin/sh

echo 'this is a test from John Liu'
```

Updating the patch after you made some change (you can do this as often as you
like):

```bash
$ quilt refresh
```

You can also add description to the patch using `quilt`:
```bash
$ quilt header --dep3 -e
```

After you've finished, run the command:
```bash
$ quilt pop -a
```

And that's it. You can check the patch file in `debian/patches/`.

### Make sure the script will be installed to /usr/bin

There are many files under `debian/` that control how a package should be
installed. To specify additional files that needs to be in the system, we could
use [`debian/install`](https://www.debian.org/doc/manuals/maint-guide/dother.en.html#install).
Create the file and put the following content into it:

```
testing.sh usr/bin
```

One-liner:
```bash
$ echo 'testing.sh usr/bin' > debian/install
```

### Add a "post-install" script

It's convenient to run custom scripts at different stages while the package is
installing. This is also achieved by creating special files under `debian/`.
To add a script run after the package got installed, create a file named
`postinst` under `debian/` and make sure it's executable. I will add a simple
script that shows my name:

```
#!/bin/sh

echo 'this is a test from John Liu'
```

For custom scripts that runs at different stages when the package is installed,
see [Package maintainer scripts and installation procedure](https://www.debian.org/doc/debian-policy/ch-maintainerscripts.html).

### Documenting the fix

Every Debian and Ubuntu package source includes `debian/changelog` which tracks
the change of each version. You could use `dch` tool to help you record your
change:

```bash
$ dch -i
```

This will add a boilerplate changelog entry for you and launch an editor where
you can fill in the blanks. Here's an example:

```
hello (2.10-2ubuntu2ppa1) focal; urgency=medium

  * Add `testing.sh` and install it to /usr/bin.
  
  * Add `postinst` script  

 -- John Liu <johnliu55tw@gmail.com>  Sat, 14 Nov 2020 12:57:35 +0800
```

Three import things to note here:

1. **Version number (`2.10-2ubuntu2ppa1`)**: If you're creating an alternative version of a package
   already available in Ubuntu's repositories, you should ensure that:
   - Your package supersedes the official Ubuntu version
   - Future Ubuntu versions will supersede your package

   I've added a suffix `ppa1` to the original version number to achieve this.
   See https://help.launchpad.net/Packaging/PPA/BuildingASourcePackage#Versioning
   for more detail about versioning your package.

2. **Upload Ubuntu release (`focal`)**: The default value is `UNRELEASED`,
   which would cause build to fail. I'm using 20.04 so I changed it to `focal`.

3. **Name and email address (`John Liu <johnliu55tw@gmail.com>`)**: Make sure
   the email address is equal to the one you used to create the GPG key, or
   the build process can't sign the `source.changes` file automatically, which
   would cause the upload PPA process to fail.

See
[3.8.1 Documenting the fix](https://packaging.ubuntu.com/html/fixing-a-bug.html#documenting-the-fix)
for more about how to document your change. 

### Testing the change

```bash
$ debuild -S -d -us -uc
```

```bash
$ pbuilder-dist focal build ../hello_2.10-2johnliu1.dsc
```

```bash
$ dpkg -I ~/pbuilder/*_result/hello_*.deb
```

## Re-package it as deb and upload to your PPA

With the changelog entry written and saved, run `debuild` again to generate the
source package and sign it:

```bash
$ debuild -S -d
```

Now you can use `dput` command to upload the `*.changes` file:
```bash
$ dput ppa:<Your Launchpad ID>/<Your PPA URL> <source.changes>
```

In my case it's:
```bash
$ dput ppa:johnliu55tw/ppa ../hello_2.10-2ubuntu2ppa1_source.changes
```

You can check your PPA on Launchpad website for the correct URL.

> **Note:** Launchpad builds the pacakges onsite, and does not accept deb
> files!

You will receive an email that tells you whether the package is successfully
uploaded and accepted. Be sure to check for the email.

Notice that after it's uploaded, it takes some time for your source package to
be built and published on Launchpad. Go to your PPA page, check the *Status*
and *Build Status* of your package and make sure they are all finished.

![PPA Package finshed](./doc-images/ppa-package-done.png)

## Install the package from your PPA

> **Note:** Make sure your packages are built and published!

In order to install the package from your PPA, you have to add the PPA to your
system:

```bash
$ sudo add-apt-repository ppa:johnliu55tw/ppa
$ sudo apt-get update
```

Then you can install your package by `apt`. In my case I want to install
`hello`:

```bash
$ sudo apt install hello
```

Notice that if the version of your package does not supersedes the offical
Ubuntu version (i.e. your package is not newer), `apt` will install the
offical package, not yours.

## References

- [Ubuntu Packaging Guide](https://packaging.ubuntu.com/html/)

- [Packaging/PPA - Launchpad Help](https://help.launchpad.net/Packaging/PPA)

- [Using Quilt - Debian Wiki](https://wiki.debian.org/UsingQuilt)

## Huh?

### Error occurred when I run `debuild` command

When I first try to run `debuild -S -d -us -uc` I got the error:
```
johnliu@johnliu-ubuntu-2004-vm:~/hello/hello-2.10$ debuild -S -d -us -uc
 dpkg-buildpackage -us -uc -ui -S -d
dpkg-buildpackage: info: source package hello
dpkg-buildpackage: info: source version 2.10-2johnliu1
dpkg-buildpackage: info: source distribution focal
dpkg-buildpackage: info: source changed by John Liu <johnliu@johnliu-ubuntu-2004-vm>
 dpkg-source --before-build .
 debian/rules clean
dh clean
make: dh: Command not found
make: *** [debian/rules:3: clean] Error 127
dpkg-buildpackage: error: debian/rules clean subprocess returned exit status 2
debuild: fatal error at line 1182:
dpkg-buildpackage -us -uc -ui -S -d failed
```

The `dh` command is provided by package `debhelper`, but is not listed in 
required package in the
[Getting Set Up](https://packaging.ubuntu.com/html/getting-set-up.html) page.
I need to install this package.

### Error occurred when I run `dput` command
```
Checking signature on .changes
gpg: /home/johnliu/repackage-htop/htop_2.2.0-2ubuntu1_source.changes: error 58: gpgme_op_verify
gpgme_op_verify: GPGME: No data
```

Main cause is that the file `*.changes` is not signed. I have to correct
file `debian/changelog`:

- UNRELEASED

### Create patch for the `testing.sh`

Following article
[Fixing a bug in Ubuntu](https://packaging.ubuntu.com/html/fixing-a-bug.html),
when I try to create a new patch using command `edit-patch 99-new-patch`, some
error occurred:
```bash
Normalizing patch path to 99-new-patch
Normalizing patch name to 99-new-patch.patch
No series file found
```

Main cause is that the `hello` package does not have any patches so
`debian/patches/series` cannot be found. Need to create a patch for it.

### Failed to add my PPA to my system

```
johnliu@johnliu-ubuntu-2004-vm:~/hello/hello-2.10$ sudo add-apt-repository ppa:johnliu55tw/ppa
 
 More info: https://launchpad.net/~johnliu55tw/+archive/ubuntu/ppa
Press [ENTER] to continue or Ctrl-c to cancel adding it.

Hit:1 http://tw.archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://tw.archive.ubuntu.com/ubuntu focal-updates InRelease [111 kB]                        
Get:3 http://tw.archive.ubuntu.com/ubuntu focal-backports InRelease [98.3 kB]                                
Get:4 http://tw.archive.ubuntu.com/ubuntu focal amd64 Contents (deb) [40.9 MB]                               
Err:5 http://ppa.launchpad.net/johnliu55tw/ppa/ubuntu focal InRelease                                             
  403  Forbidden [IP: 91.189.95.83 80]
Get:6 http://security.ubuntu.com/ubuntu focal-security InRelease [107 kB]                                         
Get:7 http://security.ubuntu.com/ubuntu focal-security i386 Contents (deb) [8,320 kB]                                                                                  
Get:8 http://tw.archive.ubuntu.com/ubuntu focal i386 Contents (deb) [32.2 MB]                                                                                          
Get:9 http://security.ubuntu.com/ubuntu focal-security amd64 Contents (deb) [16.6 MB]                                                                                  
Get:10 http://tw.archive.ubuntu.com/ubuntu focal-updates/main amd64 Packages [655 kB]                                                                                  
Get:11 http://tw.archive.ubuntu.com/ubuntu focal-updates/main i386 Packages [370 kB]                                                                                   
Get:12 http://tw.archive.ubuntu.com/ubuntu focal-updates/main amd64 DEP-11 Metadata [232 kB]                                                                           
Get:13 http://tw.archive.ubuntu.com/ubuntu focal-updates amd64 Contents (deb) [20.4 MB]                                                                                
Get:14 http://tw.archive.ubuntu.com/ubuntu focal-updates i386 Contents (deb) [11.0 MB]                                                                                 
Get:15 http://tw.archive.ubuntu.com/ubuntu focal-updates/main amd64 c-n-f Metadata [11.0 kB]                                                                           
Get:16 http://tw.archive.ubuntu.com/ubuntu focal-updates/universe amd64 Packages [689 kB]                                                                              
Get:17 http://tw.archive.ubuntu.com/ubuntu focal-updates/universe i386 Packages [512 kB]                                                                               
Get:18 http://tw.archive.ubuntu.com/ubuntu focal-updates/universe amd64 DEP-11 Metadata [205 kB]                                                                       
Get:19 http://tw.archive.ubuntu.com/ubuntu focal-updates/universe amd64 c-n-f Metadata [13.0 kB]                                                                       
Get:20 http://tw.archive.ubuntu.com/ubuntu focal-updates/multiverse amd64 DEP-11 Metadata [2,468 B]                                                                    
Get:21 http://tw.archive.ubuntu.com/ubuntu focal-backports amd64 Contents (deb) [7,720 B]                                                                              
Get:22 http://tw.archive.ubuntu.com/ubuntu focal-backports i386 Contents (deb) [6,243 B]                                                                               
Get:23 http://tw.archive.ubuntu.com/ubuntu focal-backports/universe amd64 DEP-11 Metadata [1,768 B]                                                                    
Reading package lists... Done                                                                                                                                          
E: Failed to fetch http://ppa.launchpad.net/johnliu55tw/ppa/ubuntu/dists/focal/InRelease  403  Forbidden [IP: 91.189.95.83 80]
E: The repository 'http://ppa.launchpad.net/johnliu55tw/ppa/ubuntu focal InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

You can see that when it's updating my PPA, it received 403 Forbidden.
```
Err:5 http://ppa.launchpad.net/johnliu55tw/ppa/ubuntu focal InRelease                                             
  403  Forbidden [IP: 91.189.95.83 80]
```

I googled and found this issue is caused by
[the PPA has no package](https://askubuntu.com/questions/1212715/getting-a-403-error-with-a-ppa).
It turns out that receiving the "Accepted" email doesn't mean your package
is ready to ship via PPA. You have to wait until it's built and published,
then others can install it through your PPA:

![PPA Package status](./doc-images/ppa-package-status.png)

I ran `apt update` after some time and the PPA got updated without any error.

One thing really confuses me is that even though the *Status* column shows
"Published", I still can't fetch the newer version package using `apt update`.
I have to wait until the *Build Status* column became a green check then I
could fetch the package.

after the first successful upload, I
then upload a newer version of the package, and ran `apt update` after its
status become "Published". However, I still can install the newer version of
the package.

### The permission of "testing.sh" after installed to /usr/bin

The permission of the file is 644, but after it got installed to `/usr/bin`,
it becomes 755. Maybe there are some rules that automatically changed the
permission because I want it in `/usr/bin`?
