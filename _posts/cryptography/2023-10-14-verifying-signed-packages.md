---
layout: post
title:  "Verifying signed packages"
date:   2023-10-14 20:30:00 +0200
categories: ["pgp", "checksum", "sha", "gpg", "verify","package","keys","apt","rpm","dnf","yum"]
---

I recently decided to improve my approach to security when it comes to packages I download, hence whenever possible I try to verify the packages I download before installation.

When it comes to verifying that a package is what you expect it to be, there are multiple levels.
The first is verifying that the package contents haven't been altered, and second is verifying the the package was created by the party that you are expecting the package from.

I recommend visitng [Apache Infra's Docs on release signing](https://infra.apache.org/release-signing.html), it has detailed information on how to sign and verify their packages, though this knowledge can be applied to any `gpg` signed package.

## Verify package integrity
Verifying that the package hasn't been altered is done by means of [hashing](https://www.codecademy.com/resources/blog/what-is-hashing/) the contents of the package and verifying the output of that hash.

The site where you download the package from may also show a hash value along with the corresponding hashing-algorithm used. The algorithms which are generaly used are *md5*, *sha-1*, and *sha-256*, though md5 and sha-1 are to be considered deprecated.

We can test this out with a package we create ourselves and see what info we would need to pass to anyone who we share it with.

Let's create a package by just adding some content to a file.

```bash
$ echo "If SHA was by my side" > mypackage
```

Now we can check the hash of this package with the different algorithms.

```bash
# md5
$ md5sum mypackage
8619256a49102efc10dbfadc6b401008  mypackage

# sha-1
$ sha1sum mypackage
f83476d857c00529eb158da4f3983f0332540111  mypackage

# sha-256
$ sha256sum mypackage
93b418e9ebd888275f59728e41a47010326b5ba49237885dc100539b4b048af7  mypackage
```

Note that even if the content of the file is changed slightly the result of the hashes will be significantly different. Let's try it out by adding a character and looking at the resulting change in hashes.

```bash
$ echo "If SHAQ was by my side" > mypackage
```

```bash
$ md5sum mypackage
60ef5150dd25080699bb272830d490a5  mypackage

# sha-1
$ sha1sum mypackage
13849c93a6d50def5ea3709e2ebbd96ed51310c3  mypackage

# sha-256
$ sha256sum mypackage
cf1f28e4eb8388a5ce9be9e2b505d1c7e86ce0d14286825d80baaeac82b68631  mypackage
```

It's clear from the outputs that changing the content will alter the hash. When a package is uploaded along with its hash-value and corresponding algorithm we can do the above inspection to verify that the hash that the uploader calculated matches what we calculate after downloading the package. That way you can confirm that the package has not been modified in anyway after the upload.


## Verify package signature

By verifying the signature you can verify the authenticity of a file and be sure of the author who signed the file, as signatures are unique to the keys used by signers.
For most packages you'll be able to do a verification using `gpg` (GNU Privacy-Guard) which is an implemantation of PGP (Pretty Good Privacy). I used to always confuse the two when trying to run the commands, but now just remembering that the **g** in `gpg` stands for GNU (a Linux OS) helps me remember which I need to use.
When it comes to package managers they will usually have an internal mechanism which does the verification. You'll generally see an instruction step on the provider's site for importing a public key. This public key is then used by the package manager to verify the download.

Let's try out the `gpg` verification on a package from the Apache Infra site. I'll be running the commands from a Debian docker container, if you don't have docker installed you can do so by following instructions from [Install Docker](https://docs.docker.com/engine/install/).

Starting up and entering the container.
```bash
docker run -it debian:12
```

Once in the container, we'll first need to install some required packages.
```bash
$ apt update
$ apt install curl gpg -y
```

Next we'll download the files we need, which are the installation file, the Apache foundation public keys, and the package signature. Note depending when you're doing this exercise you're output values may differ due to newly available packages or signatures, however the versions shown here will give the same outputs if they are still available.

```bash
# create directory for files
$ mkdir test && cd test

# httpd server package
$ curl -LO https://dlcdn.apache.org/httpd/httpd-2.4.57.tar.gz

# Sha256 hash value
$ curl -LO https://downloads.apache.org/httpd/httpd-2.4.57.tar.gz.sha256

# Package signature
$ curl -LO https://downloads.apache.org/httpd/httpd-2.4.57.tar.gz.asc
```

Let's do a check of the hash to see if the file may have been tampered with using the `.sha256` file provided.

```bash
# check shasum of downloaded package
$ sha256sum httpd-2.4.57.tar.gz
bc3e7e540b83ec24f9b847c6b4d7148c55b79b27d102e21227eb65f7183d6b45  httpd-2.4.57.tar.gz

# check value provided by Apache
$ cat httpd-2.4.57.tar.gz.sha256
bc3e7e540b83ec24f9b847c6b4d7148c55b79b27d102e21227eb65f7183d6b45 *httpd-2.4.57.tar.gz
```
The two hash values match, so now we can be certain that the package wasn't tampered with after upload.

### Verify signature
We will need to download the public keys before we can successfully verify the package, but let's see what happens if we try to verify while having only the signature

#### gpg

```shell
$ gpg --verify httpd-2.4.57.tar.gz.asc
gpg: directory '/root/.gnupg' created
gpg: keybox '/root/.gnupg/pubring.kbx' created
gpg: assuming signed data in 'httpd-2.4.57.tar.gz'
gpg: Signature made Sun Apr  2 16:08:52 2023 UTC
gpg:                using RSA key 65B2D44FE74BD5E3DE3AC3F082781DE46D5954FA
gpg: Can't check signature: No public key
```
When we run the above `gpg` command it will try to find a file that is similarly named but without the `.asc` extension, meaning it will verify that signature against `httpd-2.4.57.tar.gz`. From the output we read `gpg: assuming signed data in 'httpd-2.4.57.tar.gz'`. On the final line of the output we see that it can't verify the signature without the public key. The output also gives us information about the key that signed it, it can be identified by the last 16 digits, which in this case are `82781DE46D5954FA`

```bash
# Import public keys
$ curl -sL https://downloads.apache.org/httpd/KEYS | gpg --import
```
The above command imports all the public keys of the developers who sign the releases for the Apache foundation. Now that we have all the public keys, among them we'll find the one which signed our downloaded package. To find that we can use `gpg` along with the `grep` command, with the 16 digits from the previous output, in order to filter out the one we are looking for.

```bash
$ gpg --list-keys| grep 82781DE46D5954FA -B1 -A3
pub   rsa4096 2009-11-17 [SC]
      65B2D44FE74BD5E3DE3AC3F082781DE46D5954FA
uid           [ unknown] Eric Covener <covener@apache.org>
uid           [ unknown] Eric Covener <ecovener@us.ibm.com>
sub   rsa4096 2009-11-17 [E]
```
Go check out [Anatomy of a GPG key](https://davesteele.github.io/gpg/2014/09/20/anatomy-of-a-gpg-key/) for an  in-depth explanation of the gpg key output.

Now that we're sure that we have the key used to verify the signature installed in the systems, we can finally verify the autenticity of the file. This time we see the package signer's information along with the note `Good signature`.

```bash
$ gpg --verify httpd-2.4.57.tar.gz.asc
gpg: assuming signed data in 'httpd-2.4.57.tar.gz'
gpg: Signature made Sun Apr  2 16:08:52 2023 UTC
gpg:                using RSA key 65B2D44FE74BD5E3DE3AC3F082781DE46D5954FA
gpg: Good signature from "Eric Covener <covener@apache.org>" [unknown]
gpg:                 aka "Eric Covener <ecovener@us.ibm.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 65B2 D44F E74B D5E3 DE3A  C3F0 8278 1DE4 6D59 54FA
```

The output gives a warning about the key not being certified. This is because we haven't trusted the public key, our system is familiar with the key and can validated that the key is related to the signature however we don't actually who the owner of the key is. Since we haven't met the signer in person and have them verify that it is indeed their key, we're then just going of the assumption that the key does indeed belong to them.

Let's say we believe the person is who they claim to be in the key and we want our system to trust keys. First we'll have to have our own set of PGP keys, then use the private key to sign the person's keys.

When generating your set of keys you'll be prompted for our name and email.
```bash
# Generate PGP key pair
gpg --default-new-key-algo rsa4096 --gen-key
```

Then to sign the person's public key we can use the email attached to the key. You'll get a few prompts to make sure you want to trust the key

```bash
# Trust public key
gpg --sign-key covener@apache.org
```

Now if we try to verify the package again we'll get an output without the warning, also the `unknown` in the output has changed to `full`.

```bash
# Verify package
$ gpg --verify httpd-2.4.57.tar.gz.asc
gpg: assuming signed data in 'httpd-2.4.57.tar.gz'
gpg: Signature made Sun Apr  2 16:08:52 2023 UTC
gpg:                using RSA key 65B2D44FE74BD5E3DE3AC3F082781DE46D5954FA
gpg: Good signature from "Eric Covener <covener@apache.org>" [full]
gpg:                 aka "Eric Covener <ecovener@us.ibm.com>" [full]
```

#### apt (debian/ubuntu)

Let's use VSCode installation as an example, you can find the actual steps [here](https://code.visualstudio.com/docs/setup/linux). I'll be doing the steps in a `debian:12` container with `gpg` and `curl` installed. I'll do the steps different here to better understand the process.


```bash
# create working directory
$ apt update && apt install -y curl gpg
$ mkdir downloads && cd downloads

# Download the public key
$ curl -LO https://packages.microsoft.com/keys/microsoft.asc

# Generate an ascii output in binary OpenPGP format file
$ gpg --dearmor microsoft.asc

# Place OpenPGP file in directory apt public keys
$ cp microsoft.asc.gpg /etc/apt/keyrings/packages.microsoft.gpg

# Add microsoft repository to download VSCode from. Not the `signed-by` value matches the OpenPGP file
$ echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] \
      https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list

# Continue with the remaining steps
$ apt install apt-transport-https
$ apt update
$ apt install code
```

When first using the `gpg` commands and came across the *armor* term I was a bit confused, but this is just a way to convert between binary and ascii. Check out the short explanation on [ASCII-Armor](https://www.techopedia.com/definition/23150/ascii-armor).

In the above script all should have gone well and vscode would be downloaded to the system. Now let's try something different, let's see what would happen when things go wrong.

If when adding the repository we leave out  `signed-by=/etc/apt/keyrings/packages.microsoft.gpg` from the command and then run `apt update` we'll geta warning and an error. We would get the same if the `signed-by` was present but the actual OpenPGP file `/etc/apt/keyrings/packages.microsoft.gpg` was absent.

Warning
```bash
W: GPG error: https://packages.microsoft.com/repos/code stable InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY EB3E94ADBE1229CF
```

Error
```bash
E: The repository 'https://packages.microsoft.com/repos/code stable InRelease' is not signed.
```

These are because by default any package installed with `apt` must be signed and the public key of the signer must be installed on the system. The `gpg` verification can be disbaled (though not recommended) by passing the option `--allow-unauthenticated` to the `apt` command. If this must be setup permanently (again not recommended) then it can be done using a config file in `/etc/apt/apt.conf.d/`. The file can be named `99nosigcheck` with the below content.
```conf
# /etc/apt/apt.conf.d/99nosigcheck
APT::Get::AllowUnauthenticated "true";
```

I wanted to verify the `.deb` package you can download on the VSCode site, and it took a while to get a download-link I could use in the terminal. But to my surprise the package don't seem to be signed, not could I find any detached signature. They do however provide the sha256 hashes for the downloads, so that's nice.

```bash
# Download package
$ curl -LO https://az764295.vo.msecnd.net/stable/f1b07bd25dfad64b0167beb15359ae573aecd2cc/code_1.83.1-1696982868_amd64.deb

# Import public key to gpg
$ cat  microsoft.asc | gpg --import

# Verify imported key
$ gpg --list-keys
/root/.gnupg/pubring.kbx
------------------------
pub   rsa2048 2015-10-28 [SC]
      BC528686B50D79E339D3721CEB3E94ADBE1229CF
uid           [ unknown] Microsoft (Release signing) <gpgsecurity@microsoft.com>

# Verify package
$ gpg --verify code_1.83.1-1696982868_amd64.deb
gpg: no signed data
gpg: can't hash datafile: No data
```

#### dnf/yum (rhel/fedora)

When using `dnf` or `yum` you'll just need to specify the public key in the repo info and enable `gpgcheck` for that repo. Using [VSCode](https://code.visualstudio.com/docs/setup/linux#_rhel-fedora-and-centos-based-distributions) as an example. I'll be using the `fedora:38` image.

```bash
# Add repo info to yum repos
$ sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1" > /etc/yum.repos.d/vscode.repo'

# Refresh repo
$ dnf check-update
```

In the above command I've left out `\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc`, this is in order to see what happens if the key is not added.

Now if we try to install code, it will download all the packages to cache and fail at the end when trying to install code as it can't pass the signature verification step.

```bash
$ dnf install code -y
.....
.....
Public key for code-1.83.1-1696982959.el7.x86_64.rpm is not installed
The downloaded packages were saved in cache until the next successful transaction.
You can remove cached packages by executing 'dnf clean packages'.
Error: GPG check FAILED
```

If we now tell the repo where it can find the gpgkey, this can be either a file `gpgkey=file://<path-to-file>` or a url `gpgkey=https://<file-url-path>`

```bash
# Add repo info to yum repos
$ sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'

# Refresh repo
$ dnf check-update

# Install VSCode
$ dnf install code -y
......
......
Complete!
```

Now we have a succesfull installation.


#### rpm (rhel/fedora)

For `rpm` signed packages see [RPM and GPG: How to verify Linux packages before installing them](https://www.redhat.com/sysadmin/rpm-gpg-verify-packages). I'll just summarize some information on this page.

I'll be using the `redhat/ubi8-minimal` docker image.

```bash
$ mkdir downloads && cd downloads

# Download package
$ curl -LO https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/e/epel-release-8-19.el8.noarch.rpm

# Download public key
$ curl -LO http://download.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-8
```

With `rpm` signing a portion of the package is signed, instead of having a separate signature, the signature is embedded in the package.
We can view the signature and the required key by running the below command.
```bash
# Inspect package signature
$ rpm -qp epel-release-8-19.el8.noarch.rpm --qf '%{NAME}-%{VERSION}-%{RELEASE} %{SIGPGP}\n'

warning: epel-release-8-19.el8.noarch.rpm: Header V4 RSA/SHA256 Signature, key ID 2f86d6a1: NOKEY
epel-release-8-19.el8 89023304000108001d16210494e279eb8d8f25b21810adf121ea45ab2f86d6a10502643d4a69000a091021ea45ab2f86d6a169a00ffe23f1bb750a8581e89db4ed61167babbec3fe1e8ac5c4f8616b11cfa9c9d99d59a842127d48a58ecfd4c7a5fc461316456005bb419bd365ecc121211a79f8a69aae8fd9310807c261efc5513d2ecfac7780998058a55370c34b9dc0c282173b05e04d8ac385b93cdc57a913984acd896e40cb6f1d9b0e4c58f4a33eb86cb34bdbf148b55352ead898811658ad5ff66073169a44c84aff96e3fcd8217ae6a8d4241f80f8e69cc9c8d3e323f9fc9f3a282fadbf871f44c02d19ed63f45550d51a0da55618d852bc33d5854b1ef1601164d844be545c2d6cc66a43afbb49e5b710b4dc7465b1c48e8bbfa8be5360412d74c84a48fe60a13df993ea46bca633570ec1436dcb208a427437fa809fdc40345a8880fb5ab06e7ac43e566de13e5f847e1b9b8a9828c3183a07a76e75820f3b5fa84b5e40c4cc273491a904121bd89f986b5c2dc477a5c94864609461fad8435ef576c591b6cceaafa916b63aa0e049dbecdb40bfab506f0ae53c423f795537caeca15b69c9a5f6f14cdf811cec69060367f5a7088782a27d9cc667c7c7cb6a2b0b83fac8027e595f1e94756dd9db847b62dcf8f53390268703bea2d0e1a97cba340b8f41a8b5f0b54fa802319d1c063804f9aa3943a947cbdda3dad9f8dd425976fa2f9a8b0422560641c41c5dba5fe3234ac2359dbbfd31be8d01a0886c4e770000a45f4b72c93c0334e23959a1252864
```

In the output we see a `NOKEY` message referring to a key-id `2f86d6a1`, the long string of numbers in the output is the signature that we need to verify. In order to verify the package we will need to import the public key using rpm. It's not possible to use `gpg` to verify a `rpm` signed package. But first let's try verifying the package without importing the key and see the output.

```bash
# Verify package signature
$ rpm --checksig epel-release-8-19.el8.noarch.rpm
epel-release-8-19.el8.noarch.rpm: digests SIGNATURES NOT OK
```

Due to the unavailable key we get a `SIGNATURES NOT OK` message. Let's see how the output changes after importing the key.

```bash
# Import pgp key into rpm
$ rpm --import RPM-GPG-KEY-EPEL-8
```

Once we've imported a key to `rpm` we can list all the keys using
```bash
$ rpm -qa gpg-pubkey*
gpg-pubkey-fd431d51-4ae0493b
gpg-pubkey-2f86d6a1-5cf7cefb
gpg-pubkey-d4082792-5b32db75
```

One of them has the id we're interested in `2f86d6a1`, i.e. `gpg-pubkey-2f86d6a1-5cf7cefb`. We can also inspect it using the below command.
```bash
$ rpm -qi gpg-pubkey-2f86d6a1-5cf7cefb
Name        : gpg-pubkey
Version     : 2f86d6a1
Release     : 5cf7cefb
Architecture: (none)
Install Date: Sat Oct 14 18:50:22 2023
Group       : Public Keys
Size        : 0
License     : pubkey
Signature   : (none)
Source RPM  : (none)
Build Date  : Wed Jun  5 14:17:31 2019
Build Host  : localhost
Relocations : (not relocatable)
Packager    : Fedora EPEL (8) <epel@fedoraproject.org>
Summary     : gpg(Fedora EPEL (8) <epel@fedoraproject.org>)
Description :
-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: rpm-4.14.3 (NSS-3)

mQINBFz3zvsBEADJOIIWllGudxnpvJnkxQz2CtoWI7godVnoclrdl83kVjqSQp+2
dgxuG5mUiADUfYHaRQzxKw8efuQnwxzU9kZ70ngCxtmbQWGmUmfSThiapOz00018
+eo5MFabd2vdiGo1y+51m2sRDpN8qdCaqXko65cyMuLXrojJHIuvRA/x7iqOrRfy
a8x3OxC4PEgl5pgDnP8pVK0lLYncDEQCN76D9ubhZQWhISF/zJI+e806V71hzfyL
/Mt3mQm/li+lRKU25Usk9dWaf4NH/wZHMIPAkVJ4uD4H/uS49wqWnyiTYGT7hUbi
ecF7crhLCmlRzvJR8mkRP6/4T/F3tNDPWZeDNEDVFUkTFHNU6/h2+O398MNY/fOh
yKaNK3nnE0g6QJ1dOH31lXHARlpFOtWt3VmZU0JnWLeYdvap4Eff9qTWZJhI7Cq0
Wm8DgLUpXgNlkmquvE7P2W5EAr2E5AqKQoDbfw/GiWdRvHWKeNGMRLnGI3QuoX3U
pAlXD7v13VdZxNydvpeypbf/AfRyrHRKhkUj3cU1pYkM3DNZE77C5JUe6/0nxbt4
ETUZBTgLgYJGP8c7PbkVnO6I/KgL1jw+7MW6Az8Ox+RXZLyGMVmbW/TMc8haJfKL
MoUo3TVk8nPiUhoOC0/kI7j9ilFrBxBU5dUtF4ITAWc8xnG6jJs/IsvRpQARAQAB
tChGZWRvcmEgRVBFTCAoOCkgPGVwZWxAZmVkb3JhcHJvamVjdC5vcmc+iQI4BBMB
AgAiBQJc9877AhsPBgsJCAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRAh6kWrL4bW
oWagD/4xnLWws34GByVDQkjprk0fX7Iyhpm/U7BsIHKspHLL+Y46vAAGY/9vMvdE
0fcr9Ek2Zp7zE1RWmSCzzzUgTG6BFoTG1H4Fho/7Z8BXK/jybowXSZfqXnTOfhSF
alwDdwlSJvfYNV9MbyvbxN8qZRU1z7PEWZrIzFDDToFRk0R71zHpnPTNIJ5/YXTw
NqU9OxII8hMQj4ufF11040AJQZ7br3rzerlyBOB+Jd1zSPVrAPpeMyJppWFHSDAI
WK6x+am13VIInXtqB/Cz4GBHLFK5d2/IYspVw47Solj8jiFEtnAq6+1Aq5WH3iB4
bE2e6z00DSF93frwOyWN7WmPIoc2QsNRJhgfJC+isGQAwwq8xAbHEBeuyMG8GZjz
xohg0H4bOSEujVLTjH1xbAG4DnhWO/1VXLX+LXELycO8ZQTcjj/4AQKuo4wvMPrv
9A169oETG+VwQlNd74VBPGCvhnzwGXNbTK/KH1+WRH0YSb+41flB3NKhMSU6dGI0
SGtIxDSHhVVNmx2/6XiT9U/znrZsG5Kw8nIbbFz+9MGUUWgJMsd1Zl9R8gz7V9fp
n7L7y5LhJ8HOCMsY/Z7/7HUs+t/A1MI4g7Q5g5UuSZdgi0zxukiWuCkLeAiAP4y7
zKK4OjJ644NDcWCHa36znwVmkz3ixL8Q0auR15Oqq2BjR/fyog==
=84m8
-----END PGP PUBLIC KEY BLOCK-----
```

If we now try to verify the package we'll get a good response.

```bash
# Verify package signature
$ rpm --checksig epel-release-8-19.el8.noarch.rpm
epel-release-8-19.el8.noarch.rpm: digests signatures OK
```

With the keys imported we now get a `signatures OK` message indicating a success. I just realized while writing this that the output is capitalized when the check fails, and it's normal when it succeeds.
