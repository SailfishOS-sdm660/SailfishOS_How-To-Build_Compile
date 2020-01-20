# How To Build SailfishOS #

## Starting from ground zero

When starting out, the ideal situation would be creating another user just for building to keep the environment consistent:
```
HOST $

sudo useradd porter -s /bin/bash -m -G wheel -c "SFOS Builder"
sudo passwd porter
su - porter
```

To make the host environment suitable for building you'll need the following changes made:

1. [Append these lines to your `~/.bashrc`](files/.bashrc)
2. [Create a `~/.hadk.env` with the following content](files/.hadk.env)
3. [Create a `~/.mersdkubu.profile` with the following content](files/.mersdkubu.profile)
4. [Create a `~/.mersdk.profile` with the following content](files/.mersdk.profile)
5. Get the `repo` command:
```
HOST $

mkdir ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod +x ~/bin/repo
```

## Setup the Platform SDK

Here's where the magic magic happens in terms of building Sailfish OS itself. To set it up we need to create some initial directories for the SUSE based Platform SDK chroot and extract it:
```
HOST $

exec bash
sudo mkdir -p $PLATFORM_SDK_ROOT/{targets,toolings,sdks/sfossdk}
sudo ln -s /srv/mer/sdks/sfossdk/srv/mer/sdks/ubuntu/ /srv/mer/sdks/ubuntu
cd && curl -k -O http://releases.sailfishos.org/sdk/installers/latest/Jolla-latest-SailfishOS_Platform_SDK_Chroot-i486.tar.bz2
sudo tar --numeric-owner -p -xjf Jolla-latest-SailfishOS_Platform_SDK_Chroot-i486.tar.bz2 -C $PLATFORM_SDK_ROOT/sdks/sfossdk
mkdir -p $ANDROID_ROOT
sfossdk
```

After entering the Platform SDK you get to choose your first target device! I chose `cheeseburger` as that's the only device I own:
```
Which hybris-15.1 device would you like to build for?

  1. whyred (Xiaomi Redmi Note 5 Pro)
  2. dumpling     (OnePlus 5T)
  
  Choice: (1/2) 1
Cloning droid HAL & configs for whyred... done!

Env setup for whyred
```

When gaining control of the prompt we need to fetch the HADK Android tools for utils such as mkbootimg:
```
PLATFORM_SDK $

sudo zypper ref -f
sudo zypper --non-interactive in bc pigz atruncate android-tools-hadk
```
**NOTE:** Repository errors for `adaptation0` can be safely ignored here and in the future.

## Adding SFOS build target

In the Platform SDK we use Scratchbox to build packages for the target device architecture. Releases for the SDK targets can be found [here](http://releases.sailfishos.org/sdk/targets/) if another version is desired. To build against the latest public release e.g. `3.2.1.20` at the time of writing, the following command should be run:
```
PLATFORM_SDK $ cd && sdk-manage target install $VENDOR-$DEVICE-$PORT_ARCH http://releases.sailfishos.org/sdk/targets/Sailfish_OS-$RELEASE-Sailfish_SDK_Target-$PORT_ARCH.tar.7z --tooling SailfishOS-$RELEASE --tooling-url http://releases.sailfishos.org/sdk/targets/Sailfish_OS-$RELEASE-Sailfish_SDK_Tooling-i486.tar.7z
```

**NOTE:** You can add an entry for another device model by simply choosing it when entering Platform SDK again and installing another target!

To verify that the install(s) have succeeded, executing `sdk-assistant list` should yield something like this:
```
PLATFORM_SDK $ sdk-assistant list
SailfishOS-3.2.1.20
|-xiaomi-whyred-armv7hl
`-oneplus-dumpling-armv7hl
```

## Setup the HABUILD SDK

Next we'll pull down & extract the Ubuntu 14.04 chroot environment where LineageOS HAL parts shall be built:
```
PLATFORM_SDK $

TARBALL=ubuntu-trusty-20180613-android-rootfs.tar.bz2
cd && curl -O https://releases.sailfishos.org/ubu/$TARBALL
UBUNTU_CHROOT=$PLATFORM_SDK_ROOT/sdks/ubuntu
sudo mkdir -p $UBUNTU_CHROOT
sudo tar --numeric-owner -xjf $TARBALL -C $UBUNTU_CHROOT
sudo sed "s/\tlocalhost/\t$(</parentroot/etc/hostname)/g" -i $UBUNTU_CHROOT/etc/hosts
cd $ANDROID_ROOT
habuild
```

When pulling down large amounts of source code it is a good idea to configure git as to not limit our available resources and cause other issues:
```
HA_BUILD $

git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

## Cleaning up

Once you can enter both PLATFORM_SDK and HA_BUILD environments, you can safely delete the leftover chroot filesystem archives from your home directory:
```
HA_BUILD $ cd && rm Jolla-latest-SailfishOS_Platform_SDK_Chroot-i486.tar.bz2 ubuntu-*-android-rootfs.tar.bz2
```

## Initializing local repo

When everything is ready to go we can finally init the local source repository:
```
HA_BUILD $

cd $ANDROID_ROOT
repo init -u git://github.com/SailfishOS-sdm636/SailfishOS_manifest.git -b android-8.1

```

## Build environments

After initial setup you will have the `PLATFORM_SDK` and `HA_BUILD` environments ready to build the required parts. To enter `HA_BUILD` env you first need to go through `PLATFORM_SDK`.

How to enter `PLATFORM_SDK`:
```
HOST $ sfossdk
PLATFORM_SDK $
```

How to enter `HA_BUILD`:
```
PLATFORM_SDK $ habuild
HA_BUILD $
```

Leaving an environment can be achieved by simply entering `exit` or pressing `CTRL + D`
```
HA_BUILD $ exit
PLATFORM_SDK $
```

## Syncing local repository

At this point the process of downloading source code for LineageOS and libhybris will start. This will also be done when updating source code repos.

At first this may take a while depending on your internet connection speed (with 200 mbit/s it'll take ~8 mins for reference):
```
HA_BUILD $ repo sync -c -j`nproc` --fetch-submodules --no-clone-bundle --no-tags
```

If this is your first time building, execute the following line to finalize the environment:
```
HA_BUILD $ . build/envsetup.sh && breakfast $DEVICE && export USE_CCACHE=1
```

## Building HAL parts

Now we will build the required parts of LineageOS for HAL to function properly under SFOS. This usually takes around 6 minutes on 16 Zen 2 threads (R7 3700X) for the first time. To start the process, enter:
```
HA_BUILD $ mka hybris-hal
```

**NOTE:** If this was your first time building the droid HAL side, the following needs to be also executed for working camera, video playback & recording etc; this shouldn't take very long:
```
echo "MINIMEDIA_AUDIOPOLICYSERVICE_ENABLE := 1" > external/droidmedia/env.mk
mka droidmedia audioflingerglue
```

During the `hybris-hal` build process `hybris-*.img` boot images in `out/target/product/$DEVICE/` will be generated. When kernel and other Android side changes are done afterwards the image can be regenerated using:
```
HA_BUILD $ mka hybris-boot
```

## Building SFOS packages & rootfs<a name="building-sfos-packages-rootfs"></a>

To pull updates and start (re)building all locally required SFOS packages & the rootfs, run the following command (full build takes ~20 minutes for me):
```
PLATFORM_SDK $ build_all_packages
```
As the rootfs `mic` build command line is now included in `build_packages.sh` steps, this is all you need to get a rather tiny (~340 MB) flashable SFOS zip file! Look into the [flashing guide](FLASHING.md) on how to proceed afterwards.

When just droid configs have been modified, `build_device_configs` will be enough. Same goes for droid HAL stuff with `build_droid_hal` instead. Building with these commands instead will be substantially faster than rebuilding everything (which is unnecessary 99% of the time anyways).

The rootfs build can still be triggered manually too if required via `run_mic_build`. This makes sense if you've just modified `droid-configs` for example and have no need to rebuild all packages for no reason again :)
