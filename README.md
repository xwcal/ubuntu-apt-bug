# [Bug #2017399: `apt-get -s upgrade` calculates the upgrade differently depending on whether InRelease files are present in the cache ](https://bugs.launchpad.net/ubuntu/+source/apt/+bug/2017399)

## Background
I have an in-house package manager that downloads packages from mirrors in parallel and maintains a versioned local repo. It has worked well since 2019. However, during a recent migration from 18.04 to 22.04 I encountered a strange bug: the install step keeps failing because the download step has missed the following packages: xserver-common xserver-xorg-core and xserver-xorg-legacy. It turns out that `apt-get -s`, which the download step uses to figure out the packages that need to be downloaded, produces different results depending on whether InRelease files are present in the cache (as specified in `Dir::State::Lists`). I suspect Phased-Update-Percentage is a related factor.

## How to reproduce
1. Launch a virtual machine with `xubuntu-22.04.2-desktop-amd64.iso` (sha256: c7072859113399bef0e680a08c987c360f41642d8089ebb8928689066a9c9759)

2. Download `minimal.tar.xz` (sha256: ff7c38c8db2f05813dd641da504182180bf81d72cca9306a0898684c6b839b65) and run `tar -xf minimal.tar.xz` under `/some/dir` to extract the index files. They were taken from a real scenario last week.

3. Replace the content of /etc/apt/sources.list with the following three lines:
```
deb file:/home/apt/repo jammy main universe multiverse restricted
deb file:/home/apt/repo jammy-security main universe multiverse restricted
deb file:/home/apt/repo jammy-updates main universe multiverse restricted
```

4. run `apt-get -s -o Dir::State::Lists=/some/dir/lists2 upgrade` and observe the output -- you should hopefully get:
```
The following packages have been kept back:
  linux-generic-hwe-22.04 linux-headers-generic-hwe-22.04 linux-image-generic-hwe-22.04
...
123 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
```

5. `rm /some/dir/*InRelease*` and then run `apt-get -s -o Dir::State::Lists=/some/dir/lists2 upgrade` again -- you should hopefully get:
```
The following packages have been kept back:
  linux-generic-hwe-22.04 linux-headers-generic-hwe-22.04 linux-image-generic-hwe-22.04 xserver-common xserver-xorg-core xserver-xorg-legacy
...
120 upgraded, 0 newly installed, 0 to remove and 6 not upgraded.
```

## Desired behavior
`apt-get -s` should produce consistent results regardless of when InRelease files are present in the cache.

# A Short Critique of Phased Updates

Some thoughts after I stumbled on phased updates during a recent migration from Bionic to Jammy: 

## Randomness and Repeatability

Per [Debian repository specs](https://wiki.debian.org/DebianRepository/Format#Phased-Update-Percentage), randomization is achieved by the following means:

```
To determine whether an update is applicable to a given machine, a client shall seed a random number generator (APT uses std::minstd_rand) with the string:

 sourcePackage-version-machineID 

where sourcePackage is the name of the source package, version the version, and machineID the contents of /etc/machine-id.

It shall then extract an integer from a uniform integer distribution between 0 and 100. 
```

To the uninitiated: this is not how a random number generator is meant to be used. If you want a repeatable sequence of random numbers, you seed the RNG once and then draw the desired number of samples. You don't reseed the RNG for each new sample, since if your seeds are already random, then the RNG step is pointless, whereas if your seeds aren't random, say, in an extreme case, if you always seed with the same number, your "random" number is guaranteed to be the same each time.

To achieve repeatable randomness with respect to both the package and machine, there is an obviously superior choice in terms of both randomness and portability: **just use a cryptographic hash function like sha256**.

This begs the question of how repeatable the design really is.
To quote [https://wiki.debian.org/MachineId](https://wiki.debian.org/MachineId):

```
what is the machine id actually used for?

    a comprehensive list is probably not possible without grepping the code 
```

That's to say, although for repeatable experiments it's foreseeably necessary to mess with the machine ID or share it publicly,
the consequences of such tinkering or disclosure are hard to assess, since nobody knows for sure how and where the machine ID is and will be used.

This leads to an easily conceivable alternative: since apt only uses the machine ID for the purpose of repeatable randomization, how about designating another user configurable value stored in, say `/etc/apt/phased-update-random-seed`, only for this particular purpose and by default initialized to the machine ID?

## User Choice

From [this response](https://bugs.launchpad.net/ubuntu/+source/apt/+bug/2017399/comments/1) to my bug report (one question I forgot to ask is why phase security updates at all) and a comment on [askubuntu](https://askubuntu.com/questions/1431940/what-are-phased-updates-and-why-does-ubuntu-use-them) that doesn't cite any source, it does appear that consideration has been given regarding the need to treat different packages (at least different classes of packages) differently when it comes to phasing. 

It's conceivable that a user may have preferences over how each particular package is phased. For example, if a GUI component has a troubled history with my Nvidia graphics, I may prefer to be highly conservative and wait until the last minute, but this shouldn't prevent me from being adventurous when it comes to other phased packages.

Unfortunately, it seems that a user currently has to decide between all or nothing, ie. between `APT::Get::Always-Include-Phased-Updates` or `APT::Get::Never-Include-Phased-Updates`.

How about allowing per package user choice between Always and Never?

Overall, I think for a such a prominent change to the cornerstone of probably half of today's Linux infrastructure, much remains to be desired.



