# cjdns for Ubiquiti EdgeOS

### Introduction

This is a cjdns distributable package for Ubiquiti EdgeOS. It supports configuration through the standard configuration editor and should therefore be very straight-forward to deploy.

At this time this package is in very early stages of development, but the ultimate aim is to provide binary builds for some commonly-used platforms with pre-built cjdns binaries.

### Compatibility

|                       | Architecture | Compatible |                          Notes                             |
|-----------------------|:------------:|:----------:|:----------------------------------------------------------:|
|    EdgeRouter X (ERX) |    mipsel    |     Yes    | Builds with crossbuild-essential, see below                |
| EdgeRouter Lite (ERL) |    mips64    |     Yes    | Builds with Codescape SDK as mips32 which works, see below |

### Building for EdgeRouter X

On 64-bit Debian Jessie, start by installing the toolchain:
```
echo "deb http://emdebian.org/tools/debian/ jessie main" >> /etc/apt/sources.list

wget http://emdebian.org/tools/debian/emdebian-toolchain-archive.key
apt-key add emdebian-toolchain-archive.key

dpkg --add-architecture mipsel
apt-get update
apt-get install crossbuild-essential-mipsel
```
Compile the package then by cloning the repository and running 'make':
```
make
```
The package `vyatta-cjdns.deb` will be created in the parent directory.

### Building for EdgeRouter Lite

On 64-bit Debian Jessie, start by installing the build-essential package:
```
sudo apt-get update
sudo apt-get install -y build-essential
```
So far the only proven working toolchain is the Codescape SDK 2015.01.7 (and 2015.06.05).

Download the toolchain, assuming your homedir:
```
wget http://codescape-mips-sdk.imgtec.com/components/toolchain/2015.06-05/Codescape.GNU.Tools.Package.2015.06-05.for.MIPS.MTI.Linux.CentOS-5.x86_64.tar.gz
tar xf Codescape.GNU.Tools.Package.2015.06-05.for.MIPS.MTI.Linux.CentOS-5.x86_64.tar.gz
```
Add the bin dir to your `PATH` variable:
```
PATH=$HOME/mips-mti-linux-gnu/2015.06-05/bin:$PATH
```
Clone the repository, edit the file `vyatta-cjdns/debian/control` and change `Architecture: mipsel` to `Architecture: mips`, and then initiate the build by running:
```
PREFIX='mips-mti-linux-gnu-' make -e
```
The package `vyatta-cjdns.deb` will be created in the parent directory.

### Configuration

To establish a peering is straight-forward; replace `bind-address a.b.c.d:e` with the address you want cjdroute to listen on in `ip:port` format and replace `peers a.b.c.d:e` with the `ip:port` address of your peer:
```
configure
set interfaces cjdns tun0 udp-interface 0 bind-address a.b.c.d:e
set interfaces cjdns tun0 udp-interface 0 peers a.b.c.d:e password xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
set interfaces cjdns tun0 udp-interface 0 peers a.b.c.d:e publickey xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.k
commit
```
To configure beacons to automatically peer with other devices on your network using ethernet (assuming `switch0` is your internal interface):
```
configure
set interfaces cjdns tun0 ethernet-interface 0 bind-interface switch0
set interfaces cjdns tun0 ethernet-interface 0 beacon 2
commit
```
An IPv6 address and a keypair are automatically generated when you create a new cjdns interface. The `publickey`, `privatekey` and `ipv6` fields will be automatically populated with these. To manually configure your own IPv6 address and keypair (i.e. to bring in an existing keypair from another machine):
```
configure
set interfaces cjdns tun0 publickey xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.k
set interfaces cjdns tun0 privatekey xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
set interfaces cjdns tun0 ipv6 xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx
commit
```
To configure new authorized passwords for incoming connections:
```
configure
set interfaces cjdns tun0 authorized-password user1 password password1
set interfaces cjdns tun0 authorized-password user2 password password2
commit
```
An example stateful firewall configuration that will block unexpected incoming traffic on the cjdns interface, i.e. to prevent other people from reaching your ssh or Web UI ports:
```
set firewall ipv6-name CJD_LOCAL default-action drop
set firewall ipv6-name CJD_LOCAL rule 10 action accept
set firewall ipv6-name CJD_LOCAL rule 10 state established enable
set firewall ipv6-name CJD_LOCAL rule 10 state related enable
set firewall ipv6-name CJD_LOCAL rule 20 action drop
set firewall ipv6-name CJD_LOCAL rule 20 state invalid enable
set interfaces cjdns tun0 firewall local ipv6-name CJD_LOCAL
```
To see information about peerings, in operational view:
```
show interfaces cjdns tun0 peers
```
To see your IPv6 address and public/private keys, in operational view:
```
show interfaces cjdns tun0 identity
```
To restart a cjdns tunnel, in operational view:
```
restart cjdns tun0
```

### Footnotes

There is very little input validation right now on the configuration, so if you enter badly-formed config then `cjdroute` will simply fail to start.

You may also need to manually adjust your firewall to allow traffic on the `bind-address` that you specified.

The EdgeRouter X claims architecture `mipsel`, but the EdgeRouter Lite claims architecture `mips64`. Not currently sure how to cross-compile for `mips64` using crossbuild on Jessie. Any thoughts appreciated.
