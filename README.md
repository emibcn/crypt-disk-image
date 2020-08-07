# crypt-disk-image

### Create, open and close a qemu thin provisioned QCow2 image containing a crypted filesystem

The idea is to have a thin provisioned file (which grows as more space is required) containing
sensible data which needs to be crypted. This is done by using `qemu-img` to create a QCow2
thin provisioned image. Then, use `qemu-nbd` to serve it ithough a socket and `nbd-client` to
map it to a device. This device is used to save a LUKS container, in which an Ext4 filesystem
is created. Finally, this filesystem is mounted on some user's directory so files can be
manipulated transparently.

The `crypt-disk-image-create` script takes care of creating the disk image (with all it's internal
components), the mounpoint and a script to help open and close the disk. Uses a simpler method
to open and close the NBD device, as it does not needs to be suspend safe.

The `crypt-disk-image` is called by the autogenerated script mentioned above. This is the
responsible of doing all the steps for opening or closing the disk.

# Installation

## Dependencies

You need to ensure all dependencies are met. How is done depends on each system. To check if
all of them are met, run:

```
crypt-disk-image-create -c
```

The needed commands are: `sudo qemu-img qemu-nbd nbd-client cryptsetup mkfs.ext4 fsck.ext4 fstrim`

You will probably need to install only those: `sudo qemu-img qemu-nbd nbd-client cryptsetup mkfs.ext4 fsck.ext4`

On a Debian/Ubuntu, it's accomplished with something like:

```
sudo apt install -y sudo qemu-img qemu-nbd nbd-client cryptsetup e2fsprogs
```

## Copy scripts to somewhere in `PATH`

To mitigate the fact that both scripts need to be run as `root`, you'll probably want them in some global
`PATH` (for example, `/usr/local/bin/`). This way, a malicious user needs `root` access to modify them.

```
cd
git clone https://github.com/emibcn/crypt-disk-image.git
sudo cp crypt-disk-image/crypt-disk-image* /usr/local/bin/
```

## Add it to `sudoers`

As both scripts need to be run as root, you might want to allow this execution without password. For doing this, you can
create a sudoer file like this with `visudo /etc/sudoers.d/91-crypt-disk-image`:

```
# Allow mounting encrypted filesystem

myuser ALL=(ALL) NOPASSWD: /usr/local/bin/crypt-disk-image, /usr/local/bin/crypt-disk-image-create
```

# Usage

Both scripts have a very detailed `--help` option, logging and in-script comments. However, here are some basic examples:

## Creating an image

You can create an image with the next command:

- Image on `${PWD}/disk.img`, mounted on `${PWD}/disk`, using `cmdipass` to retrieve the password with `'disk'` key previously
  created in KeePass and saving the easing script into `${HOME}/bin/disk`:

```
crypt-disk-image-create --mount=disk -p "cmdipass get-one 'disk' --index=0 --password-only" -x $HOME/bin/disk disk.img
```

## Using an image

The script above outputs the commands that have to be executed in order to open or close the image which, in this case, will be:

```
# To open the image, considering that $HOME/bin is in user's PATH
disk open

# The contents can be manipulated in the mountpoint
ls -la disk/

# To close the image
disk close
```

# CAUTION!

### Thin provisioning, `fstrim` and information leaks
As the idea is to keep the image as small as possible (thin provisioned), it's needed to
allow `fstrim` to work all the way from the contained filesystem down to the QCow2 image.
This leaks some information which some people may find an insecure decision. If you agree,
you'll probably prefer to don't thin provision and, then, don't use NBD with QCow2 at all.

More: http://asalor.blogspot.com/2011/08/trim-dm-crypt-problems.html

### Root permissions
This script needs to be run as `root` to perform several actions. Some of those actions
might be insecure to perform over an untrusted image file. For example, from
https://manpages.debian.org/testing/qemu-utils/qemu-nbd.8.en.html :

    "a malicious guest may have prepared the image to attempt to trigger kernel bugs in
    partition probing or file system mounting."

Some extra precautions have been taken when working with files and directories to mitigate
the fact that this is beeing run by a user with sudo powers, for example, only create files
and directories under a directory owned by the user. However, those precautions would not
be enough (in general or in your particular case), so don't use it in untrusted environments.

### Password store
To open the LUKS device, a password is needed. It's recommended to use a randomly generated one
at least 32 characters long (as with `pwgen -sy 32`). To ensure no password is stored in any
command prompt (visible in `/proc` and other places, like `top` or `ps`), the password is
passed by pipe executing a command specified by `PW_CMD` argument. This command is executed by
the original owner (not root) using the `sudo` command.

It's recommended to use a password store to store and retrieve this password. You need to
create the password before creating or opening the crypted filesystem.

- A very insecure example:
  ```
  echo "MyPassword"
  ```

- An insecure example suitable for some orchestation environments, like Docker or Kubernetes:
  ```
  cat /run/secrets/my-password
  ```

- An example for `PW_CMD` connected to KeePass:
  ```
  cmdipass get-one "disk" --index=0 --password-only
  ```

  `cmdipass` reads a password from an open KeePass using `http-connector` KeePass' plugin.
  As it is executed with user's permissions (and not root), mitigates possible security problems.
  KeePass needs to be unlocked and with the desired database active.
  Looks for an entry named `disk`.
  `cmdipass` will use user's config file.

- Another example, with KWallet:
  ```
  kwalletcli -f "kwalletcli" -e "disks-documents"
  ```

- Another example, using 'pass': https://www.passwordstore.org/
  ```
  pass "Disks/Documents"
  ```

### Use at your own risk!!

### Use at your own risk!! Not kidding!

### Use at your own risk!! Seriously!

# License
The scripts and documentation in this project are released under the [GNU General Public License v3.0](https://github.com/emibcn/crypt-disk-image/blob/master/LICENSE).
