# Background
I have an in-house package manager that downloads packages from mirrors in parallel and maintains a versioned local repo. It has worked well since 2019. However, during a recent migration from 18.04 to 22.04 I encountered a strange bug: the install step keeps failing because the download step has missed the following packages: xserver-common xserver-xorg-core and xserver-xorg-legacy. It turns out that `apt-get -s`, which the download step uses to figure out the packages that need to be downloaded, produces different results depending on whether InRelease files are present in the cache (as specified in `Dir::State::Lists`). I suspect Phased-Update-Percentage is a related factor.

# How to reproduce
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

# Desired behavior
`apt-get -s` should produce consistent results regardless of when InRelease files are present in the cache.
