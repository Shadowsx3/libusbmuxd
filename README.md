# libusbmuxd

*A client library for applications to handle usbmux protocol connections with iOS devices.*

![](https://github.com/libimobiledevice/libusbmuxd/actions/workflows/build.yml/badge.svg)

> This fork is intended as a macOS/iPhone Wi-Fi command fix for
> `libimobiledevice` tools. It lets commands such as `ideviceinfo -n`,
> `idevicediagnostics -n`, and `idevicepair -n validate` find paired iPhones
> over Wi-Fi through Xcode/CoreDevice when Apple's usbmuxd does not report the
> device in the normal network device list.

## Table of Contents
- [Features](#features)
- [Building](#building)
  - [macOS Homebrew install for iPhone Wi-Fi commands](#macos-homebrew-install-for-iphone-wi-fi-commands)
  - [Prerequisites](#prerequisites)
    - [Linux (Debian/Ubuntu based)](#linux-debianubuntu-based)
    - [macOS](#macos)
    - [Windows](#windows)
  - [Configuring the source tree](#configuring-the-source-tree)
  - [Building and installation](#building-and-installation)
  - [macOS arm64 local install with libimobiledevice tools](#macos-arm64-local-install-with-libimobiledevice-tools)
- [Usage](#usage)
- [Contributing](#contributing)
- [Links](#links)
- [License](#license)
- [Credits](#credits)

## Features

This project is a client library to multiplex connections from and to iOS
devices alongside command-line utilities.

It is primarily used by applications which use the [libimobiledevice](https://github.com/libimobiledevice/)
library to communicate with services running on iOS devices.

The library does not establish a direct connection with a device but requires
connecting to a socket provided by the usbmuxd daemon.

The usbmuxd daemon is running upon installing iTunes on Windows and Mac OS X.

The [libimobiledevice project](https://github.com/libimobiledevice/) provides an open-source reimplementation of
the [usbmuxd daemon](https://github.com/libimobiledevice/usbmuxd.git/) to use on Linux or as an alternative to communicate with
iOS devices without the need to install iTunes.

Some key features are:

- **Protocol:** Provides an interface to handle the usbmux protocol
- **Port Proxy:** Provides the `iproxy` utility to proxy ports to the device
- **Netcat:** Provides the `inetcat` utility to expose a raw connection to the device
- **Cross-Platform:** Tested on Linux, macOS, Windows and Android platforms
- **Flexible:** Allows using the open-source or proprietary usbmuxd daemon

Furthermore the Linux build optionally provides support using inotify if
available.

## Building

### macOS Homebrew install for iPhone Wi-Fi commands

This is the recommended install path for macOS arm64 users who want
`libimobiledevice` commands to work against an iPhone over Wi-Fi. The tap
installs this fork of `libusbmuxd` and a matching `libimobiledevice` build whose
tools link to it.

```shell
brew tap Shadowsx3/tools
brew install Shadowsx3/tools/libusbmuxd Shadowsx3/tools/libimobiledevice
```

Make sure your shell resolves the tapped tools from Homebrew:

```shell
eval "$(/opt/homebrew/bin/brew shellenv)"
which ideviceinfo
which idevicediagnostics
idevice_id --version
otool -L "$(which ideviceinfo)" | grep -E 'libimobiledevice|libusbmuxd'
```

On Apple Silicon with the default Homebrew prefix, the `otool` output should
show `ideviceinfo` using Homebrew's `libimobiledevice` and this tap's
`libusbmuxd`. The important line is:

```text
/opt/homebrew/opt/libusbmuxd/lib/libusbmuxd-2.0.7.dylib
```

Enable Wi-Fi access for the iPhone and get the device UDID:

```shell
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
xcrun xctrace list devices
xcrun devicectl list devices
```

Then connect the iPhone over USB once, open Finder, select the iPhone under
Locations, enable "Show this iPhone when on Wi-Fi", and click Apply. Keep the
Mac and iPhone on the same network, disconnect USB, and verify:

```shell
idevice_id -n -l
ideviceinfo -n -u <UDID> -k DeviceName
ideviceinfo -n -u <UDID> -q com.apple.mobile.battery
idevicediagnostics -n -u <UDID> ioregentry AppleSmartBattery
```

The `AppleSmartBattery` IORegistry command exposes raw battery values such as
`AppleRawBatteryVoltage`, `Amperage`, `AppleRawCurrentCapacity`,
`AppleRawMaxCapacity`, `DesignCapacity`, `Temperature`, and `CycleCount`.

If automatic discovery finds the device but a command cannot connect to the
right Wi-Fi endpoint, set the endpoint explicitly. Use the UDID shown by
`xcrun xctrace list devices` or `xcrun devicectl list devices`, then get the
iPhone's local IP from Settings > Wi-Fi > current network > info button.

Direct local IP form:

```shell
export LIBUSBMUXD_NETWORK_DEVICES="<UDID>=<local-ip>"
```

For example:

```shell
export LIBUSBMUXD_NETWORK_DEVICES="00008150-000629380108401C=192.168.1.20"
```

Classic Bonjour hostname form:

```shell
export LIBUSBMUXD_NETWORK_DEVICES="<UDID>=<iPhone-hostname>.local"
```

For example:

```shell
export LIBUSBMUXD_NETWORK_DEVICES="00008150-000629380108401C=Basss-iPhone.local"
```

Persist it for new zsh shells if needed:

```shell
echo 'export LIBUSBMUXD_NETWORK_DEVICES="<UDID>=<local-ip>"' >> ~/.zshrc
```

Use `dscacheutil -q host -a name <iPhone-hostname>.local` or
`ping <iPhone-hostname>.local` to confirm the hostname resolves on your LAN.
Bonjour hostnames commonly derive from the device name, for example "Jane's
iPhone" often appears as `Janes-iPhone.local`.

### Prerequisites

You need to have a working compiler (gcc/clang) and development environent
available. This project uses autotools for the build process, allowing to
have common build steps across different platforms.
Only the prerequisites differ and they are described in this section.

libusbmuxd requires [libplist](https://github.com/libimobiledevice/libplist) and [libimobiledevice-glue](https://github.com/libimobiledevice/libimobiledevice-glue). On Linux, it also requires [usbmuxd](https://github.com/libimobiledevice/usbmuxd) installed on the system, while macOS has its own copy and on Windows AppleMobileDeviceSupport package provides it.
Check [libplist's Building](https://github.com/libimobiledevice/libplist?tab=readme-ov-file#building) and [libimobiledevice-glue's Building](https://github.com/libimobiledevice/libimobiledevice-glue?tab=readme-ov-file#building)
section of the respective README on how to build them. Note that some platforms might have them as a package.

#### Linux (Debian/Ubuntu based)

* Install all required dependencies and build tools:
  ```shell
  sudo apt-get install \
  	build-essential \
  	pkg-config \
  	checkinstall \
  	git \
  	autoconf \
  	automake \
  	libtool-bin \
  	libplist-dev \
  	libimobiledevice-glue-dev \
  	usbmuxd
  ```
  In case libplist-dev, libimobiledevice-glue-dev, or usbmuxd are not available, you can manually build and install them. See note above.

#### macOS

* Make sure the Xcode command line tools are installed. Then, use either [MacPorts](https://www.macports.org/)
  or [Homebrew](https://brew.sh/) to install `automake`, `autoconf`, `libtool`, etc.

  Using MacPorts:
  ```shell
  sudo port install libtool autoconf automake pkgconfig
  ```

  Using Homebrew:
  ```shell
  brew install libtool autoconf automake pkg-config
  ```

#### Windows

* Using [MSYS2](https://www.msys2.org/) is the official way of compiling this project on Windows. Download the MSYS2 installer
  and follow the installation steps.

  It is recommended to use the _MSYS2 MinGW 64-bit_ shell. Run it and make sure the required dependencies are installed:

  ```shell
  pacman -S base-devel \
  	git \
  	mingw-w64-x86_64-gcc \
  	make \
  	libtool \
  	autoconf \
  	automake-wrapper \
  	pkg-config
  ```
  NOTE: You can use a different shell and different compiler according to your needs. Adapt the above command accordingly.

### Configuring the source tree

You can build the source code from a git checkout, or from a `.tar.bz2` release tarball from [Releases](https://github.com/libimobiledevice/libusbmuxd/releases).
Before we can build it, the source tree has to be configured for building. The steps depend on where you got the source from.

Since libusbmuxd depends on other packages, you should set the pkg-config environment variable `PKG_CONFIG_PATH`
accordingly. Make sure to use a path with the same prefix as the dependencies. If they are installed in `/usr/local` you would do

```shell
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
```

* **From git**

  If you haven't done already, clone the actual project repository and change into the directory.
  ```shell
  git clone https://github.com/libimobiledevice/libusbmuxd
  cd libusbmuxd
  ```

  Configure the source tree for building:
  ```shell
  ./autogen.sh
  ```

* **From release tarball (.tar.bz2)**

  When using an official [release tarball](https://github.com/libimobiledevice/libusbmuxd/releases) (`libusbmuxd-x.y.z.tar.bz2`)
  the procedure is slightly different.

  Extract the tarball:
  ```shell
  tar xjf libusbmuxd-x.y.z.tar.bz2
  cd libusbmuxd-x.y.z
  ```

  Configure the source tree for building:
  ```shell
  ./configure
  ```

Both `./configure` and `./autogen.sh` (which generates and calls `configure`) accept a few options, for example `--prefix` to allow
building for a different target folder. You can simply pass them like this:

```shell
./autogen.sh --prefix=/usr/local
```
or
```shell
./configure --prefix=/usr/local
```

Once the command is successful, the last few lines of output will look like this:
```
[...]
config.status: creating config.h
config.status: executing depfiles commands
config.status: executing libtool commands

Configuration for libusbmuxd 2.1.0:
-------------------------------------------

  Install prefix: .........: /usr/local
  inotify support (Linux) .: no

  Now type 'make' to build libusbmuxd 2.1.0,
  and then 'make install' for installation.
```

### Building and installation

After configuring the source tree, build and install the library and bundled
tools with:

```shell
make -j"$(sysctl -n hw.ncpu 2>/dev/null || nproc)"
make install
```

If the install prefix is a system-owned directory such as `/usr/local`, the
second command might need `sudo`.

### macOS arm64 local install with libimobiledevice tools

The following workflow installs this modified `libusbmuxd` and the
`libimobiledevice` command-line tools into a local, non-system prefix on Apple
Silicon Macs. It avoids overwriting Homebrew packages and makes the tools use
this library first. Use this fork when USB-based commands work but Wi-Fi
commands like `ideviceinfo -n -u <UDID>` or `idevicediagnostics -n -u <UDID>`
fail because the iPhone is not detected as a network device.

Use a prefix without spaces. Some autotools/libtool steps are unreliable when
the build or install path contains whitespace.

```shell
export LOCAL_PREFIX="$HOME/opt/libimobiledevice-local"
export REPOS="$HOME/src/libimobiledevice-local"
mkdir -p "$LOCAL_PREFIX" "$REPOS"
```

Install the macOS prerequisites:

```shell
xcode-select --install
brew install autoconf automake libtool pkg-config \
  libplist libimobiledevice-glue libtatsu openssl@3
```

For CoreDevice/Xcode Wi-Fi discovery, install the full Xcode app and make sure
`xcrun devicectl` is available:

```shell
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
xcrun devicectl list devices
```

Build and install this modified `libusbmuxd`:

```shell
cd "$REPOS"
git clone https://github.com/Shadowsx3/libusbmuxd.git
cd libusbmuxd

export PKG_CONFIG_PATH="$LOCAL_PREFIX/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:/opt/homebrew/share/pkgconfig"
./autogen.sh --prefix="$LOCAL_PREFIX"
make -j"$(sysctl -n hw.ncpu)"
make install
```

Build and install `libimobiledevice` tools against the local `libusbmuxd`:

```shell
cd "$REPOS"
git clone https://github.com/libimobiledevice/libimobiledevice.git
cd libimobiledevice

export PKG_CONFIG_PATH="$LOCAL_PREFIX/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:/opt/homebrew/share/pkgconfig"
export CPPFLAGS="-I$LOCAL_PREFIX/include"
export LDFLAGS="-L$LOCAL_PREFIX/lib"

./autogen.sh --prefix="$LOCAL_PREFIX" --without-cython
make -j"$(sysctl -n hw.ncpu)"
make install
```

Add the local tools to your shell startup file:

```shell
cat >> ~/.zshrc <<EOF

# Local libimobiledevice tools using modified libusbmuxd.
export LIBIMOBILEDEVICE_LOCAL="$LOCAL_PREFIX"
export PATH="\$LIBIMOBILEDEVICE_LOCAL/bin:\$PATH"
export PKG_CONFIG_PATH="\$LIBIMOBILEDEVICE_LOCAL/lib/pkgconfig:/opt/homebrew/lib/pkgconfig:/opt/homebrew/share/pkgconfig:\${PKG_CONFIG_PATH:-}"
EOF
```

Open a new shell, then verify the selected binaries and linkage:

```shell
which idevice_id
otool -L "$(which idevice_id)" | grep libusbmuxd
```

The `otool` output should reference:

```text
$HOME/opt/libimobiledevice-local/lib/libusbmuxd-2.0.7.dylib
```

To verify Wi-Fi device discovery through CoreDevice/Xcode, make sure the iPhone
is paired with Xcode and reachable on the same network, then run:

```shell
idevice_id -n -l
ideviceinfo -n -u <UDID> -k DeviceName
idevicediagnostics -n -u <UDID> diagnostics GasGauge
```

This modified library can add paired physical devices to the network device
list when usbmuxd does not report them. The discovery order is:

1. If `LIBUSBMUXD_NETWORK_DEVICES` is set, use it as the authoritative mapping
   and skip automatic fallback discovery.
2. If Apple usbmuxd already reports a network device, use that fast existing
   list.
3. If no network device is reported, browse Bonjour `_apple-mobdev2._tcp`
   records and match the discovered iPhone hostname/IP to the paired Xcode
   UDID by normalized device name.
4. If Bonjour/Xcode matching cannot prove the UDID/endpoint pair, fall back to
   CoreDevice through `xcrun devicectl list devices`.

To disable CoreDevice fallback for a command, set:

```shell
LIBUSBMUXD_COREDEVICE_AUTODISCOVERY=0 idevice_id -n -l
```

To force CoreDevice discovery even when usbmuxd already reports network
devices, set:

```shell
LIBUSBMUXD_COREDEVICE_AUTODISCOVERY=force idevice_id -n -l
```

`LIBUSBMUXD_NETWORK_DEVICES` is authoritative. If Apple usbmuxd or CoreDevice
already reports a network record for the same UDID, the explicit mapping
replaces that record. This keeps `libimobiledevice` commands on the classic
Wi-Fi lockdown endpoint, for example `Basss-iPhone.local`, instead of a
`.coredevice.local` tunnel endpoint that can be created by Xcode, Instruments,
or `xctrace`.

Use this when discovery works but commands fail during lockdown startup, for
example when `idevice_id -n -l` lists the phone but `ideviceinfo -n` or
`idevicediagnostics -n` fails during `QueryType`:

```shell
export LIBUSBMUXD_NETWORK_DEVICES="<UDID>=<iPhone-hostname>.local"
idevicediagnostics -n -u <UDID> ioregentry AppleSmartBattery
```

## Usage

### iproxy

This utility allows binding local TCP ports so that a connection to one (or
more) of the local ports will be forwarded to the specified port (or ports) on
a usbmux device.

Bind local TCP port 2222 and forward to port 22 of the first device connected
via USB:
```shell
iproxy 2222:22
```

This would allow using ssh with `localhost:2222` to connect to the sshd daemon
on the device. Please mind that this is just an example and the sshd daemon is
only available for jailbroken devices that actually have it installed.

Please consult the usage information or manual page for a full documentation of
available command line options:
```shell
iproxy --help
man iproxy
```

### inetcat

This utility is a simple netcat-like tool that allows opening a read/write
interface to a TCP port on a usbmux device and expose it via STDIN/STDOUT.

Use ssh ProxyCommand to connect to a jailbroken iOS device via SSH:
```shell
ssh -oProxyCommand="inetcat 22" root@localhost
```

Please consult the usage information or manual page for a full documentation of
available command line options:
```shell
inetcat --help
man inetcat
```

### Environment

The environment variable `USBMUXD_SOCKET_ADDRESS` allows to change the location
of the usbmuxd socket away from the local default one.

An example of using an utility from the libimobiledevice project with an usbmuxd
socket exposed on a port of a remote host:
```shell
export USBMUXD_SOCKET_ADDRESS=192.168.179.1:27015
ideviceinfo
```

This sets the usbmuxd socket address to `192.168.179.1:27015` for applications
that use the libusbmuxd library.

## Contributing

We welcome contributions from anyone and are grateful for every pull request!

If you'd like to contribute, please fork the `master` branch, change, commit and
send a pull request for review. Once approved it can be merged into the main
code base.

If you plan to contribute larger changes or a major refactoring, please create a
ticket first to discuss the idea upfront to ensure less effort for everyone.

Please make sure your contribution adheres to:
* Try to follow the code style of the project
* Commit messages should describe the change well without being too short
* Try to split larger changes into individual commits of a common domain
* Use your real name and a valid email address for your commits

## Links

* Homepage: https://libimobiledevice.org/
* Repository: https://github.com/libimobiledevice/libusbmuxd.git
* Repository (Mirror): https://git.libimobiledevice.org/libusbmuxd.git
* Issue Tracker: https://github.com/libimobiledevice/libusbmuxd/issues
* Mailing List: https://lists.libimobiledevice.org/mailman/listinfo/libimobiledevice-devel
* Twitter: https://twitter.com/libimobiledev

## License

This library is licensed under the [GNU Lesser General Public License v2.1](https://www.gnu.org/licenses/lgpl-2.1.en.html),
also included in the repository in the `COPYING` file.

The utilities `iproxy` and `inetcat` are licensed under the [GNU General Public License v2.0](https://www.gnu.org/licenses/gpl-2.0.en.html).

## Credits

Apple, iPhone, iPad, iPod, iPod Touch, Apple TV, Apple Watch, Mac, iOS,
iPadOS, tvOS, watchOS, and macOS are trademarks of Apple Inc.

This project is an independent software library and has not been authorized,
sponsored, or otherwise approved by Apple Inc.

README Updated on: 2024-10-22
