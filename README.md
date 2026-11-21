![HsinOS](https://github.com/hsin-os/manifest/raw/Moon_fox/img/hsin.png)

based on Miku UI AOSP-Based Project.

**How to Start！**

## Sync Sourcecode
------------


```shell
repo init -u https://github.com/hsin-os/manifest.git -b Moon_fox --git-lfs
```


```shell
repo sync -c -j$(nproc --all) --force-sync --no-clone-bundle --no-tags
```

## Start to build！！

```shell
# Init
. build/envsetup.sh

# Lunch
lunch hsin_[codename]-[release]-[build type]

# Make a diva !
m bacon
```
