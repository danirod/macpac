# macpac

(Work in progress. Please do not get too excited yet.)

**macpac** is an experiment to port the Pacman package manager to macOS.
Inspired by MSYS2 for Windows, which also uses Pacman as its package manager.
My theory is that if MSYS2 can recompile Pacman, it cannot be impossible
to do the same on macOS.

The goal is to get commands like `pacman` or `makepkg` to run in macOS.
In a similar fashion to other package managers like Homebrew or Nix, a separate
root dir will be used, and every package installed using macpac will live there.

## Current status

**pacman compiles**, and that's good. I'm still figuring out the bootstrapping
process. Some makedependencies have to be installed outside of macpac. I suggest
Homebrew.

## /macpac directory

The PKGBUILDs in this repository will use `/macpac` as the prefix. Either you
follow the instructions to create a `/macpac` directory as well, or you
remember to change it by whatever directory you want to use.

If I continue with this project, eventually I'll make an install script that does
this, but I'm going to copy what Nix does on Mac, which is: create an APFS
container and mount it at the root of your file system.

The instructions are the following, assuming you have a disk formatted as APFS.
(You should have, because it has been default for Apple for years.)

1. Edit or create a file called `/etc/synthetic.conf`. Add the following line:

```
macpac
```

2. Run the following command to force macOS to re-evaluate the synthetic file:

```bash
sudo /System/Library/Filesystems/apfs.fs/Contents/Resources/apfs.util -t
```

You should verify that a directory called `/macpac` has appeared in the root of
your file system.

3. Create a new APFS container and mount it to `/macpac`:

```bash
sudo diskutil apfs addVolume [disk] APFS macpac -mountpoint /macpac
```

Where `[disk]` is the container name of your APFS file system. For instance, `disk3`.
You can run `diskutil apfs list | head` to list your APFS containers. The value
you want to use is probably the one that appears next to
`APFS Container Reference`. You can also use the Disk Utility app if you want.

### How to uninstall this shit

Use the Disk Utility app to remove the `macpac` container. Find the `macpac`
volume in the sidebar, then right click it and use `Delete`. **Be careful because
you will lose data.**

Then modify the `/etc/synthetic.conf` file and remove the line that has `macpac`.
If the file becomes empty, you can just remove it.

You will have to reboot your computer to remove the `/macpac` directory. You cannot
use `apfs.util -t` to remove directories for entries that are not present anymore
in the `/etc/synthetic.conf` file.

## How to compile

You cannot just run `makepkg` for pacman's PKGBUILD because you don't have
makepkg installed yet. You have to bootstrap first. You will have to manually
compile and manually install pacman and a few other dependencies before you can
have an environment where `makepkg` works and can be used to compile other
packages.

(In the future, I'd like to provide a binary distribution with a precompiled
version of pacman, so that bootstrapping is not required, but I'm still figuring
out which packages need to be part of the binary distribution.)

### Installing dependencies

Install the following dependencies: `bash meson coreutils libarchive`.
**You are encouraged to use Homebrew, although technically you can install them
anyhow you want**. In fact, my suggestion is to create a separate Homebrew
environment like this:

```bash
# Visit a clean location on your system.
mkdir ~/macpac-homebrew
cd ~/macpac-homebrew

# Create a new installation of Homebrew.
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/main | tar xz --strip-components 1 -C homebrew
eval "$(homebrew/bin/brew shellenv)"
brew update --force --quiet
chmod -R go-w "$(brew --prefix)/share/zsh"

# Remove directories from PATH
export PATH=$PWD/homebrew/bin:/usr/local/bin:/usr/bin:/bin
```

The idea with the last `export PATH` is to remove your default Homebrew,
and any other package you might have installed system-wide on your Mac,
to prevent conflicts. This will be a clean environment where dependency
errors will be easily detected.

### Bootstrapping pacman

Then, proceed to compile pacman manually in order to bootstrap it.

* You will need to download 6.1.0 from
  <https://gitlab.archlinux.org/pacman/pacman/-/releases/v6.1.0/downloads/pacman-6.1.0.tar.xz>
* Extract pacman somewhere and `cd` to the directory.
* Apply the patches from `packages/bootstrap/pacman/*.patch` using
  `patch -p1 < $path_to_pacman/0001-*.patch` and `patch -p1 < $path_to_pacman/0002-*.patch`.
* Compile Pacman using the following options to force Pacman to use a directory of your choice. Here I am using /macpac, but you can use wherever you want:

    meson setup build \
        --prefix=/macpac \
        --sysconfdir=/macpac/etc \
        --localstatedir=/macpac/var \
        -Di18n=false \
        -Dmakepkg-template-dir=/macpac/share/makepkg-template
    ninja -C build
    ninja -C build install

Remember to change `/macpac` to something else if you are using a different directory.

After the installation is successful, you will have Pacman installed!

```bash
$ /macpac/bin/pacman --version

 .--.                  Pacman v6.1.0 - libalpm v14.0.0
/ _.-' .-.  .-.  .-.   Copyright (C) 2006-2024 Pacman Development Team
\  '-. '-'  '-'  '-'   Copyright (C) 2002-2006 Judd Vinet
 '--'
                       This program may be freely redistributed under
                       the terms of the GNU General Public License.
```

### Installing pacman through pacman

The `pacman` command you are using is not installed through `pacman`, so
it will not appear when running `pacman -Q`. You should use now the
`makepkg` command to actually compile pacman via the PKGBUILD in this
repository.

```bash
cd $repo/packages/bootstrap/pacman
makepkg
```

If the compile is not successful and you see errors related to the `b2sums`
command, make sure that the `b2sum` command, part of the `coreutils`,
is available.

If you have installed Coreutils using Homebrew, keep in mind that the formula
prepends a `g` to the command name. You'll have to edit the formula and remove
the offending argument in the `./configure` call to make Homebrew compile a
`b2sum` command instead of a `gb2sum`. **Or, alternatively, you might just
compile Coreutils from scratch on your own and call it a day. It should be easy
to do it.**

If the compile is successful, you should be able to reinstall pacman using
the compiled package like this:

```
pacman -U --overwrite='*' pacman-6.1.0-1-arm64.pkg.tar.gz
```

You have to use the `--overwrite='*'` argument because otherwise pacman will
refuse to install the package because the files already exist. This will not be
necessary with the rest of the packages because you will be using `makepkg` and
`pacman` to install them.

### Installing other packages

My suggestion is to install `coreutils` next through pacman, so you can stop
depending on the external one. `coreutils` is a dependency required to
install any other package via Pacman due to the checksum verification, so
it's worth doing it now.

So, again:

```bash
cd $repo/packages/core/coreutils
makepkg
pacman -U coreutils-*.pkg.tar.gz
```

Other packages are available in the repository as I compile them.

## What do you want to do?

I don't know. This is just a prototype.

I don't think if it's actually worth to have a new package manager for macOS.
Once I finish adding more packages, I'll figure out if end the experiment or
continue adding more packages.

The packages I want to compile are the basic GNU userland packages like
**coreutils**, **file**, **grep**, **bash** and so on; basic compiler tools
like **make**, **automake**, **awk**, **python** and **meson**. Probably
a few more.

Not every package has to be installed because macOS comes with a lot of UNIX
standard tools preinstalled. I think it can be fun anyway. The list of packages
is based on the list of basic system software part of the Linux From Scratch
book: <https://www.linuxfromscratch.org/lfs/view/stable/chapter08/chapter08.html>.
(Not every package in this list will be installed; there is no point in porting
systemd to macOS.)

After that, I'll decide if it's worth continueing or not.
