# PKGBUILDs
Contains PKGBUILD files for creating Arch Linux packages:

* Packages for my own applications and libraries such as
  [Syncthing  Tray](https://github.com/Martchus/syncthingtray),
  [Tag Editor](https://github.com/Martchus/tageditor),
  [Password Manager](https://github.com/Martchus/passwordmanager), ...
* Packages [I maintain in the AUR](https://aur.archlinux.org/packages/?O=0&SeB=M&K=Martchus&outdated=&SB=v&SO=d&PP=50&do_Search=Go):
    * misc packages, eg. Subtitle Composer, openelec-dvb-firmware, Jangouts
    * `mingw-w64-*` packages which allow to build for Windows under Arch Linux,
      eg. FreeType 2, Qt 5 and Qt 6
    * `static-compat-*` packages containing static libraries to build self-contained
      applications running on older GNU/Linux distributions under Arch Linux
    * `android-*` packages which allow to build for Android under Arch Linux,
      eg. iconv, Boost, OpenSSL, CppUnit, Qt 5 and Kirigami
    * `apple-darwin-*` packages which allow to build for MaxOS X under Arch
      Linux, eg. osxcross and Qt 5 (still experimental)
* Other packages imported from the AUR to build with slight modifications

So if you like to improve one of my AUR packages, just create a PR here.

## Binary repository
I also provide a [binary repository](https://martchus.no-ip.biz/repo/arch/ownstuff/os)
containing the packages found in this repository and a lot of packages found in
the AUR:

```
[ownstuff-testing]
SigLevel = Optional TrustAll
Server = https://martchus.no-ip.biz/repo/arch/$repo/os/$arch
Server = https://ftp.f3l.de/~martchus/$repo/os/$arch

[ownstuff]
SigLevel = Optional TrustAll
Server = https://martchus.no-ip.biz/repo/arch/$repo/os/$arch
Server = https://ftp.f3l.de/~martchus/$repo/os/$arch
```

The testing repository is required if you have the official testing repository
enabled. (Packages contained by ownstuff-testing are linked against packages
found in the official testing repository.)

The repository is focusing on x86_64 but some packages are also provided for
i686 and aarch64.

Note that I can not assure that required rebuilds always happen fast enough
(since the official developers obviously don't wait for me before releasing their
packages from staging).

Requests regarding binary packages can be tracked on the issue tracker of this
GitHub project as well, e.g. within the [general discussion
issue](https://github.com/Martchus/PKGBUILDs/issues/94).

## Container image, building packages within a container
The directory `devel/container` contains the script `imagebuild` to build a
container image suitable to run Arch Linux's `makepkg` script so you can build
from PKGBUILDs on any environment where Docker, Podman or any other suitable
container runtime is available.

It also contains a script called `makecontainerpkg` which behaves like
`makechrootpkg` from Arch Linux's devtools but uses the previously mentioned
container image. Therefore it does *not* require devtools, a chroot setup and
systemd-nsapwn. Instead, any container runtime should be sufficient (tested with
Docker and Podman).

The usage of `makecontainerpkg` is very similar to `makechrootpkg`. Simply run
the script in a directory containing a `PKGBUILD` file. If the directory
contains a file called `pacman.conf` and/or `makepkg.conf` those files are
configured to be used during the build. The call syntax is the following:
```
makecontainerpkg [cre args] --- [makepkg args]
```

Set the environment variable `CRE` to the container runtime executable (by
default `docker`) and set `CRE_IMAGE` to use a different container image.

Example where the host pacman cache and ccache directories are mounted into the
container and a package rebuild is forced via `makepkg`'s flag `-f`:
```
makecontainerpkg -v /var/cache/pacman/pkg/ -v /run/media/devel/ccache:/ccache -- -f CCACHE_DIR=/ccache
```

Example using podman on a non-Arch system:
```
CRE=podman ../../devel/container/makecontainerpkg -v /hdd/cache/pacman/pkg:/var/cache/pacman/pkg -v /hdd/chroot/remote-config-x86_64:/cfg
```

It makes still sense to specify a cache directory, even though pacman is not
used on the host system. Here also a directory containing a custom `pacman.conf`
and `makepkg.conf` is mounted into the container.

### Podman-specific remarks
To use podman (instead of Docker) simply set `export CRE=podman`.

To be able to run podman without root, you need to ensure user/group IDs can be
mapped. The mapping is configured in the files `/etc/subuid` and `/etc/subgid`.
Use `sudo usermod --add-subuids 200000-265536 --add-subgids 200000-265536 $USER`
to configure it for the current user and verify the configuration via
`grep $USER /etc/sub{u,g}id`.

To change storage paths so e.g. containers are stored at a different location,
edit `~/.config/containers/storage.conf` (or `/etc/containers/storage.conf` for
system-wide configuration) to set `runroot` and `graphroot` to different
locations.

Finally, run `podman system migrate` to apply.

### Investigation of build failures
By default, `makecontainerpkg` removes the container in the end. Set `DEBUG=1`
to prevent that. Then one can use e.g. `podman container exec -it … bash` to
enter the container for manual investigation. Set `DEBUG=on-failure` to only
keep the container in case of a failure.

### Using Arch-packages on another distribution via a container
If you want to cross-compile software on non-Arch distributions you can make use
of the `android-*` and `mingw-w64-*` packages provided by this repository using
an Arch Linux container. The container image mentioned before is also suitable
for this purpose.

Here are some example commands how one might do that:
```
# create a directory to store builds and new container and start it
mkdir -p /hdd/build/container
podman container create -it \
  --name archlinux-devel-container \                    # give container a meaningful name
  -v /hdd/cache/pacman/pkg:/var/cache/pacman/pkg \      # share pacman cache accross different containers
  -v /hdd/build/container:/build \                      # expose build directory to host
  -v /hdd/projects:/src \                               # access source files from host
  -v /hdd/chroot/remote-config-x86_64:/cfg \            # mount directory containing pacman.conf/makepkg.conf
  archlinux-base-devel
podman container start archlinux-devel-container

# configure pacman to use config from mounted directory
podman container exec archlinux-devel-container bash -c "$(cat devel/container/containersync)"

# start interactive shell in container
podman container exec -it archlinux-devel-container bash

# install stuff you want
podman container exec -it archlinux-devel-container \
  pacman -Syu ninja git mingw-w64-cmake qt6-{base,tools} mingw-w64-qt6-{base,tools,translations,svg,5compat}

# configure the build using mingw-w64 packages, e.g. run CMake
podman container exec -it archlinux-devel-container x86_64-w64-mingw32-cmake \
  -G Ninja \
  -S /src/c++/cmake/PianoBooster \
  -B /build/pianobooster-x86_64-w64-mingw32-release \
  -DPKG_CONFIG_EXECUTABLE:FILEPATH=/usr/bin/x86_64-w64-mingw32-pkg-config \
  -DQT_PACKAGE_NAME:STRING=Qt6

# conduct the build using mingw-w64 packages, e.g. invoke Ninja build system via CMake
podman container exec -it archlinux-devel-container bash -c '
  source /usr/bin/mingw-env x86_64-w64-mingw32 && \
  cmake --build /build/pianobooster-x86_64-w64-mingw32-release --verbose'

# configure the build using android packages, e.g. run CMake
podman container exec -it archlinux-devel-container bash -c '
  android_arch=aarch64
  source /usr/bin/android-env $android_arch && \
  android-$android_arch-cmake \
    -G Ninja \
    -S /src/c++/cmake/subdirs/passwordmanager \
    -B /build/passwordmanager-android-$android_arch-release \
    -DCMAKE_FIND_ROOT_PATH="${ANDROID_PREFIX}" \
    -DANDROID_SDK_ROOT="${ANDROID_HOME}" \
    -DPKG_CONFIG_EXECUTABLE:FILEPATH=/usr/bin/android-$android_arch-pkg-config \
    -DQT_PACKAGE_PREFIX:STRING=Qt6 \
    -DKF_PACKAGE_PREFIX:STRING=KF6'

# conduct the build using android packages, e.g. invoke Ninja build system via CMake
podman container exec -it archlinux-devel-container bash -c '
  source /usr/bin/android-env aarch64 && \
  cmake --build /build/passwordmanager-android-aarch64-release --verbose'

# use additional Android-related tooling from container
# note: These are just example values. The ports for pairing and connection distinct.
phone_ip=192.168.178.42 pairing_port=34765 pairing_code=922102 connection_port=32991
podman container exec -it archlinux-devel-container \
  /opt/android-sdk/platform-tools/adb pair "$phone_ip:$pairing_port" "$pairing_code"
podman container exec -it archlinux-devel-container \
  /opt/android-sdk/platform-tools/adb connect "$phone_ip:$connection_port"
podman container exec -it archlinux-devel-container \
  /opt/android-sdk/platform-tools/adb logcat

# get rid of the container when no longer needed
podman container stop archlinux-devel-container
podman container rm archlinux-devel-container
```

Note that these commands are intended to be run without root (see section
"Podman-specific remarks" for details). In this case files that are created
from within the container in the build and source directories will have your
normal user/group outside the container which is quite convenient (within the
container they will be owned by root).

### Other approaches
There's also the 3rdparty repository
[docker-mingw-qt5](https://github.com/mdimura/docker-mingw-qt5) which contains
an image with many mingw-w64 package pre-installed.

## Structure
Each package is in its own subdirectoy:
```
default-pkg-name/variant
```
where `default-pkg-name` is the default package name (eg. `qt5-base`) and
`variant` usually one of:

* `default`: the regular package
* `git`/`svn`/`hg`: the development version
* `mingw-w64`: the Windows version (i686/dw2 and x86_64/SEH)
* `android-{aarch64,armv7a-eabi,x86-64,x86}`: the Android version (currently
  only aarch64 actively maintained/tested)
* `apple-darwin`: the MacOS X version (still experimental)

The repository does not contain `.SRCINFO` files.

---

The subdirectoy `devel` contains additional files, mainly for development
purposes. The subdirectoy `devel/archive` contains old packages that are no
longer updated (at least not via this repository).

## Generated PKGBUILDs
To avoid repetition some PKGBUILDs are generated. These PKGBUILDs are determined
by the presence of the file `PKGBUILD.sh.ep` besides the actual `PKGBUILD` file.
The `PKGBUILD` file is only present for read-only purposes in this case - do
*not* edit it manually. Instead, edit the `PKGBUILD.sh.ep` file and invoke
`devel/generator/generate.pl`. This requires the `perl-Mojolicious` package to
be installed. Set the environment variable `LOG_LEVEL` to adjust the log level
(e.g. `debug`/`info`/`warn`/`error`). Template layouts/fragments are stored
within `generator/templates`.

### Documentation about the used templating system
* [Syntax](https://mojolicious.org/perldoc/Mojo/Template#SYNTAX)
* [Helper](https://mojolicious.org/perldoc/Mojolicious/Plugin/DefaultHelpers)
* [Utilities](https://mojolicious.org/perldoc/Mojo/Util)

## Contributing to patches
Patches for most packages are managed in a fork of the project under my GitHub
profile. For instance, patches for `mingw-w64-qt5-base` are managed at
[github.com/Martchus/qtbase](https://github.com/Martchus/qtbase).

I usually create a dedicated branch for each version, eg. `5.10.1-mingw-w64`. It
contains all the patches based on Qt 5.10.1. When doing fixes later on, I
usually preserve the original patches and create a new branch, eg.
`5.10.1-mingw-w64-fixes`.

So in this case it would make sense to contribute directly there. To fix an
existing patch, just create a fixup commit. This (unusual) fixup workflow aims
to keep the number of additional changes as small as possible.

To get the patches into the PKGBUILD files, the script
`devel/qt5/update-patches.sh` is used.

### Mass rebasing of Qt patches
This is always done by me. Please don't try to help here because it will only
cause conflicts. However, the workflow is quite simple:

1. Run `devel/qt5/rebase-patches.sh` on all Qt repository forks or just
   `devel/qt5/rebase-all-patches.sh`
    * eg. `rebase-patches.sh 5.11.0 5.10.1 mingw-w64-fixes` to create branch
     `5.11.0-mingw-w64` based on `5.10.1-mingw-w64-fixes`
    * after fixing possible conflicts, run
      `devel/qt5/continue-rebase-patches.sh`
    * otherwise, that's it
    * all scripts need to run in the Git repository directory of the Qt module
      except `rebase-all-patches.sh` which needs the environment variable
      `QT_GIT_REPOS_DIR` to be set
2. Run `devel/qt5/update-patches.sh` or `devel/qt5/update-all-patches.sh` to
   update PKGBUILDs
    * eg. `devel/qt5/update-all-patches.sh "" mingw-w64 qt6` to consider all
      mingw-w64-qt6-\* packages

## Brief documentation about mingw-w64-qt packages
The Qt project does not support building Qt under GNU/Linux when targeting
mingw-w64. With Qt 6 they also stopped 32-bit builds. They also don't provide
static builds targeting mingw-w64. They are also relying a lot on their bundled
libraries while my builds aim to build dependencies separately. So expect some
rough edges when using my packaging.

Nevertheless it make sense to follow the official documentation. For concrete
examples how to use this packaging with CMake, just checkout the mingw-w64
variants of e.g. `syncthingtray` within this repository. The Arch Wiki also has
a [section about mingw-w64
packaging](https://wiki.archlinux.org/index.php/MinGW_package_guidelines).

Note that the ANGLE and "dynamic" variants of Qt 5 packages do not work because
they would require `fxc.exe` to build.

### Tested build and deployment tools for mingw-w64-qt5 packages
Currently, I test with qmake and CMake. With both build systems it is possible
to use either the shared or the static libraries. Please read the comments in
the PKGBUILD file itself and the pinned comments in [the
AUR](https://aur.archlinux.org/packages/mingw-w64-qt5-base) for further
information.

There are also pkg-config files, but those aren't really tested.

`qbs` and `windeployqt` currently don't work very well (see issues). Using the
static libraries or mxedeployqt might be an alternative to windeployqt.

### Tested build and deployment tools for mingw-w64-qt6 packages
In order to build a Qt-based project using mingw-w64-qt6 packages one also needs
to install the regular `qt6-base` package for development tools such as `moc`.
The packages `qt6-tools`, `qt6-declarative` and `qt6-shadertools` contain also
native binaries which might be required by some projects. At this point the
setup can break if the version of regular packages and the versions of the
mingw-w64 packages differ. I cannot do anything about it except trying to
upgrade the mingw-w64 packages as fast as possible. There's actually a lengthy
discussion about this topi on the
[Qt development mailinglist](https://lists.qt-project.org/pipermail/development/2021-September/041732.html)
so the situation might improve in the future. Note that as of
qtbase commit `5ffc744b791a114a3180a425dd26e298f7399955` (requires Qt > 6.2.1)
one can specify `-DQT_NO_PACKAGE_VERSION_CHECK=TRUE` to ignore the strict
versioning check.

Currently, I test only CMake. It is possible to use either the shared or the
static libraries. The static libraries are installed into a nested prefix
(`/usr/i686-w64-mingw32/static` and `/usr/x86_64-w64-mingw32/static`) so this
prefix needs to be prepended to `CMAKE_FIND_ROOT_PATH` for using the static
libraries. To generally prefer static libraries one might use the helper scripts
provided by the `mingw-w64-cmake-static` package.

The build systems qbs and qmake are not tested. It looks like Qt's build system
does not install pkg-config files anymore and so far no effort has been taken to
enable them.

Note that windeployqt needed to be enabled by the official/regular `qt6-tools`
package but would likely not work very well anyways. Using the static libraries
or mxdeployqt might be an alternative for windeployqt.

### Static plugins and CMake
Qt 5 initially didn't support it so I added patches to make it work. After Qt 5
added support I still kept my own version because I didn't want to risk any
regressions (which would be tedious to deal with). So the [official
documentation](https://doc.qt.io/qt-5/qtcore-cmake-qt-import-plugins.html) does
**not** apply to my packages. One simply has to link against the targets of the
wanted static plugins manually.

However, for Qt 6 I dropped my patches and the official documentation applies. I
would still recommended to set the target property `QT_DEFAULT_PLUGINS` of
relevant targets to `0` and link against wanted plugin targets manually. At
least in my cases the list of plugins selected by default seemed needlessly
long. I would also recommended to set the CMake variable
`QT_SKIP_AUTO_QML_PLUGIN_INCLUSION` to a falsy value because this pulls in a lot
of dependencies which are likely not needed.

### Further documentation
The directory `qt5-base/mingw-w64` contains also a README with more Qt 5
specific information.

## Running Windows executables built using mingw-w64 packages with WINE
It is recommended to use the scripts `x86_64-w64-mingw32-wine` and
`i686-w64-mingw32-wine` provided by the `mingw-w64-wine` package. These scripts
are a wrapper around the regular `wine` binary ensuring all the DLLs provided by
`mingw-w64-*`-packages of the relevant architecture can be located. It also uses
a distinct `wine` prefix so your usual configuration (e.g. tailored to run
certain games) does not go into the way and is also not messed with.

Here are nevertheless some useful hints to run WINE manually:

* Set the environment variable `WINEPREFIX` to use a distinct WINE-prefix if
  wanted.
* Set `WINEPATH` for the search directories of needed DLLs, e.g.
  `WINEPATH=$builds/libfoo;$builds/libbar;/usr/x86_64-w64-mingw32`.
* Set `WINEARCH` to `win32` for a 32-bit environment (`win64` is the default
  which will get you a 64-bit environment)
* Set `WINEDLLOVERRIDES` to control loading DLLs, e.g.
  `WINEDLLOVERRIDES=mscoree,mshtml=` disables the annoying Gecko popup.
* To set environment variables like `PATH` or `QT_PLUGIN_PATH` for the Windows
  program itself use the following approach:
    1. Open `regedit`
    2. Go to `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment`
    3. Add/modify the variable, e.g. set
       `PATH=C:\windows\system32;C:\windows;Z:\usr\x86_64-w64-mingw32\bin` and
       `QT_PLUGIN_PATH=Z:/usr/x86_64-w64-mingw32/lib/qt6/plugins`
* It is possible to run apps in an headless environment but be aware that WINE
  is not designed for this. For instance, when an application crashes WINE
  still attempts to show the crash window and the application stays stuck in
  that state.
* See https://wiki.winehq.org/Wine_User's_Guide for more information

## Static GNU/Linux libraries
This repository contains several `static-compat-*` packages providing static
libraries intended to distribute "self-contained" executables. These libraries
are built against an older version of glibc to be able to run on older
distributions without having to link against glibc statically. The resulting
binaries should run on distributions with glibc 2.26 or newer (or Linux 4.4 and
newer when linking against glibc statically), e.g. openSUSE Leap 15.0, Fedora
27, Debian 10 and Ubuntu 18.04. The packages might not be updated as regularly
as their normal counterparts but the idea is to provide an environment with a
recent version of GCC/libstdc++ and other libraries such as Qt and Boost but
still be able to run the resulting executables on older distributions.

To use the packages, simply invoke `/usr/static-compat/bin/g++` instead of
`/usr/bin/g++`. The package `static-compat-environment` provides a script to set
a few environment variables to make this easier. There are also packages
providing build system wrappers such as `static-compat-cmake`.

It would be conceivable to make fully statically linked executables. However, it
would not be possible to support OpenGL because glvnd and vendor provided OpenGL
libraries are always dynamic libraries. It makes also no sense to link against
glibc (and possibly other core libraries) statically as they might use `dlopen`.
Therefore this setup aims for a partially statically linked build instead, where
stable core libraries like glibc/pthreads/OpenGL/… are assumed to be provided by
the GNU/Linux system but other libraries like libstdc++, Boost and Qt are linked
against statically. This is similar to AppImage where a lot of libraries are
bundled but certain core libraries are expected to be provided by the system.

Examples for resulting binaries can be found in the release sections of my
projects [Tag Editor](https://github.com/Martchus/tageditor/releases) and
[Syncthing Tray](https://github.com/Martchus/syncthingtray/releases). Those are
Qt 6 applications and the resulting binaries run on the mentioned platforms
supporting X11 and Wayland natively.

Note that I decided to let static libraries live within the subprefix
`/usr/static-compat` (in contrast to `-static` packages found in the AUR). The
main reason is that my packaging requires a custom glibc/GCC build for
compatibility and I suppose that simply needs to live within its own prefix.
Besides, the version might not be kept 100 % in sync with the shared
counterpart. Hence it makes sense to make the static packages independent with
their own headers and configuration files to avoid problems due to mismatching
versions. Additionally, some projects (such as Qt) do not support installing
shared and static libraries within the same prefix at the same time because the
config files would clash.

## Copyright notice and license
Copyright © 2015-2023 Marius Kittler

All code is licensed under [GPL-2-or-later](LICENSE).
