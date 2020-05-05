# How-to: build the PE for Android platform

* [Overview](#overview)
* [Initializing the source tree](#initializing-the-source-tree)
* [Docker environment initialization](#docker-environment-initialization)
* [Using Docker to build PE for Android](#using-docker-to-build-pe-for-android)

## Overview

PE for Android is an open-source fork of the Android Open Source Project
release for Android 9 (Pie). While the its Policy Manager and uPAL paradigms
offer much flexibility in implementing privacy-enhancing concepts as user-space
apps, we understand that there could be use cases where the platform still
needs to be modified. We invite other developers and researchers to extend PE
for Android for their own needs. This is a guide for how to clone and build
PE for Android from source.

These instructions assume familiarity with the Linux shell, the AOSP code
structure, and that Docker and [Google's `repo`
tool](https://source.android.com/setup/build/downloading#installing-repo) are
already installed on the system. For minimal complications, we recommend
building PE for Android on a 64-bit Linux system with at least 750 GBs of free
disk space. Additional CPU cores, memory, and solid-state storage will speed up
the compilation process.

## Initializing the source tree

The AOSP source tree is organized as a collection of [several hundred Git
repositories](https://android-review.googlesource.com/admin/repos), each
corresponding to a particular directory in the tree. Developers use the `repo`
tool to manage these Git repositories and perform batch operations on them.
Different AOSP builds exist for each target device. To simplify management of
all the component Git repositories, each target device has a `manifest.xml`
file that defines particular repositories and branches needed to build AOSP for
that target.

For PE for Android, we use the Generic System Image (GSI) target for Android 9
("Pie"). The following commands will initialize the metadata for this target.

```shell
> mkdir $PE_ANDROID_ROOT
> cd $PE_ANDROID_ROOT
> repo init -u https://android.googlesource.com/platform/manifest -b pie-gsi
```

After initializing the for metadata for Android 9, we include the
`local_manifests` metadata that points to the code specific to PE for Android.

```shell
> cd .repo
> git clone https://github.com/twosixlabs/pea_local_manifests.git local_manifests
> cd ..
```

With all the metadata in place, we can download the specified source files. Note that
this step  may take many minutes or hours, depending on your Internet connection.

```shell
> repo sync -c -j8
```
**FOR TEAMS**: These commands will download the AOSP source from Google's
servers. These servers have rate-limiting restrictions. If multiple team members
are cloning the source tree simultaneously, the rate-limiting will likely
trigger and cause errors in the cloning process. In order to mitigate this,
teams may wish to [set up a local mirror](https://source.android.com/setup/build/downloading#using-a-local-mirror)
so files are cloned from Google's servers only once. Developers subsequently
clone from the local mirror, and thus are not subject to rate-limiting. If
cloning from a local mirror, ensure that the `repo init` command above is
pointing to the mirror's manifest.

## Docker environment initialization

For convenience, we offer a Docker container preconfigured with all the
dependencies settings needed to build PE for Android. The following commands
will initialize the Docker container and launch it. These commands should
be done in a path *outside* the `$PE_ANDROID_ROOT` directory used in the
previous section.

```shell
> mkdir $PE_ANDROID_DOCKER
> cd $PE_ANDROID_DOCKER
> curl https://raw.githubusercontent.com/twosixlabs/PE_for_Android_dockerfile/master/Dockerfile > Dockerfile
> sudo docker build --no-cache -t peandroid:pie ./
```

## Using Docker to build PE for Android

After Docker builds the image for the environment, the following commands will
start an interactive session in a new Docker container. Let `$PE_ANDROID_ROOT`
be the path to your PE for Android source tree, defined earlier.

```shell
> cd $PE_ANDROID_DOCKER
> sudo docker run -v $PE_ANDROID_ROOT:/src/ -w /src -it --rm peandroid:pie
```
This maps `$PE_ANDROID_ROOT` to the Docker container's `/src` directory
and launches the container. From `/src` there are three shell scripts
to build different variants of PE for Android:

* `make_gsi.sh`: Builds a GSI
* `make_gsi_a.sh`: Builds a GSI for legacy A-partition devices
* `make_gsi_ab.sh`: Builds a GSI for legacy AB-partition devices
* `make_peandroid.sh`: Builds the SDK addons and emulator image

These scripts save their terminal outputs to `build_gsi.log`, `build_gsi_a.log`,
`build_gsi_ab.log`, and `addons.log`.

System image files are saved under
`$PE_ANDROID_ROOT/out/target/product/generic_arm64`, `$PE_ANDROID_ROOT/out/target/product/generic_arm64_a`, and `$PE_ANDROID_ROOT/out/target/product/generic_arm64_ab` for `make_gsi.sh`, `make_gsi_a.sh`, and`make_gsi_ab.sh`, respectively.

`make_peandroid.sh` saves the SDK addons under
`$PE_ANDROID_ROOT/out/host/linux-x86/sdk_addon/`.  Despite the path name, SDK
addons are platform-agnostic and should work with Linux, Windows, and Mac
versions of Android Studio alike. Emulator can be found under
`$PE_ANDROID_ROOT/out/target/product/generic_x86_64/`.

