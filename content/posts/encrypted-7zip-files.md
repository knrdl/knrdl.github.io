---
title: "Generate encrypted 7zip files"
date: "2024-02-13"
tags: [zip, bash]
lang: "en"
---

Using 7zip files as encrypted backups is a quick and dirty workaround for a proper backup solution.

First generate a **strong** password, e.g.: `pwgen -1sB 128`

# Backup without compression

`7z a -mx=0 -mhe=on -m0=Copy -p"$PASSWORD" TARGET.7z SOURCE_FOLDERS_*`

* `mx=0` and `m0=Copy` mean: just copy the files, do not compress them

* `mhe=on` means: also encrypt the 7z headers, so the password is also required for listing the file tree

# Backup with compression

`7z a -mx=3 -mhe=on -ms=on -p"$PASSWORD" TARGET.7z SOURCE_FOLDERS_*`

* `ms=on` treats everything as a single file (find duplicates)


# Check the archive file

`7z l -slt -p"$PASSWORD" TARGET.7z`

* `7zAES:19` means secure AES-256 encryption (plus some password hashing, see [here](https://sourceforge.net/p/p7zip/patches/25/#3da5)).