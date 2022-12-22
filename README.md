# NUT v2.8.0 Server Config
For Eaton UPS devices with the M2 network interface (SNMP)

Tested on Debian Bullseye 11 and Ubuntu 20.04

## Mandatory Disclaimer
This repository was created to help others set up their Eaton SNMP UPSes. Despite my best efforts, it should not in any way be considered a production ready implementation. My primary goal was to gain familiarity with NUT and Bash scripts while deploying the software to a handful of client machines. In conclusion, I'm not responsible for anything that goes wrong should you decide to use these deliverables and / or duplicate my efforts. You have been warned...

## Introduction
I started out with NUT v2.7.4, but realized that an upgrade was necessary. v2.8.0 has a number of stability / functionality improvements for the SNMP driver and the systemd configuration as a whole. Since I wasn't able to locate an installer package for Debian or Ubuntu (as of 12/2022), I compiled and installed the tarball. I'm pleased to say that everything was straightforward and only minor modifications were needed. The configuration files demonstrate how to gracefully shutdown the NUT host and any client machines that are connected to it, before powering down the UPS itself.

### systemd
NUT v2.8.0 comes with extensive scripts (upsdrvsvcctl and nut-driver-enumerator.sh) to manage the UPS drivers within systemd (v2.7.4 had a minimal implementation at best). This is especially important for those of us who run the snmp drivers, as a delayed network interface could cause the NUT driver to cycle indefinitely without ever establishing contact with the device. 

I recommend using the new systemd integration as it's a significant improvement. Any v2.8.0 binaries you compile will still function with an old v2.7.4 installation for testing purposes.

More information can be found here:

* [upsdrvsvcctl man page](https://networkupstools.org/networkupstools-master.github.io/docs/man/upsdrvsvcctl.html)
* [nut-driver-enumerator man page](https://networkupstools.org/networkupstools-master.github.io/docs/man/nut-driver-enumerator.html)

## Compiling NUT
Everything needed to compile under Debian can be found in these documents: 

* [NUT Config Prereqs](https://github.com/networkupstools/nut/blob/master/docs/config-prereqs.txt) - Install the dependencies outlined in this step first
* [NUT configuration script parameters](https://github.com/networkupstools/nut/wiki/Building-NUT-on-Debian,-Raspbian-and-Ubuntu) - Review and modify to suit your needs. The parameters I used are included below

### My configuration parameters
I used the arguments below for a barebones SNMP build. I removed all extraneous options. Defining the application paths is essential to ensure compatibility with your distribution. I identified a number of them by trial and error:

```
./configure --with-user=nut --with-group=nut --datadir=/usr/share/nut --prefix=/usr --sysconfdir=/etc/nut --with-statepath=/var/run/nut --with-altpidpath=/var/run/nut --with-drvpath=/usr/lib/nut --bindir=/usr/bin --sbindir=/usr/sbin --with-python3 --with-drivers=snmp-ups --without-serial --without-usb --without-neon --without-powerman --without-modbus --with-snmp --without-ipmi --without-freeipmi --with-doc=no --without-avahi --without-linux_i2c --with-ssl --with-nss --with-openssl --without-cgi --with-systemdsystemunitdir=/lib/systemd/system --with-pkgconfig-dir=/usr/lib/`gcc -print-multiarch`/pkgconfig
```

### Create the "nut" user and group
A group ("nut") and user ("nut") will need to be created for the service to run if you are bypassing the Debian / Ubuntu package and installing from scratch

Settings copied from user / group created by Debian nut-client package

#### Group
```
sudo addgroup --gid 136 nut
```

#### User
```
sudo adduser --home /var/lib/nut --no-create-home --shell /usr/sbin/nologin --uid=129 --ingroup nut--disabled-password --gecos "" nut
```

### Compile NUT
If you don't run `make install` as superuser, it will fail. If you are manually copying the files / binaries, you'll need to run this step to link / create the NUT binaries.
```
make all && make check && make install
```

If you choose to install NUT with `sudo make install`, you'll need to establish the state path (/var/run/nut) and enable the systemd services:

#### /usr/lib/tmpfiles.d/nut-server.conf
You can either run `sudo systemd-tmpfiles --create` or restart the system to establish this directory
```
d /run/nut 0770 root nut - -
```

#### Establish systemd services
```
sudo systemctl daemon-reload
sudo systemctl enable nut-driver-enumerator.path
sudo systemctl enable nut-driver-enumerator.service
sudo systemctl enable nut-driver@.service
sudo systemctl enable nut-monitor.service
sudo systemctl enable nut-server.service
sudo systemctl enable nut-driver.target
sudo systemctl enable nut.target
```

#### Invoke driver instances with /sbin/upsdrvsvcctl
Run this once you have device entries defined in /etc/nut/ups.conf
The configuration files and section provided with this repo should provide a good starting point

One or more entries will be configured for your devices. 
```
sudo /sbin/upsdrvsvcctl reconfigure
```

#### Start monitor and server
```
sudo systemctl start nut-server
sudo systemctl start nut-monitor

```
You should now be able to review the status of your UPS device(s) by running `upsc upsname`

## Configuring NUT
I included the notes I took for each section below

### /etc/nut/nut.conf
NUT server contains "MODE=netserver" if other systems will be depending on it for updates

### /etc/nut/ups.conf
The `pw` mibs had a couple more working features than `ietf`. For example, `pw` enables you to shut down outlet groups on the Eaton 5PX with `upscmd`. `ietf` enabled a beeper mute command and battery test through `upscmd`, but the latter didn't work. Shutdowns for `ietf` were performed through `upsrw`, while `pw` used `upscmd`. Neither appear to be capable of shutting down the UPS via `/sbin/upsmon -c fsd` or `/lib/nut/snmp-ups -a upsname -k`. `ietf` metrics ( from `upsc uspname`) had some minor inconsistencies. Otherwise, the two mibs are similar.

Note 1: You can get a list of available mibs by running `/lib/nut/snmp-ups -a upsname -x mibs=--list`

Note 2: If you're running the updated systemd scripts, changing this file will result in an automatic restart of the driver

### /etc/nut/upsd.conf
Enable the LISTEN command, use 0.0.0.0 to enable external addresses. Do not leave this exposed to public networks!

### /etc/nut/upsd.users
Define users with access to the NUT UPS server

The "slave" user will probably not need access to the `actions` or `instcmds` permissions. You can leave these fields out.

### /etc/nut/upsmon.conf
1. Define which UPS(es) you will be monitoring here
2. `MINSUPPLIES` will probably be 1
3. You can use a `SHUTDOWNCMD` similar to `SHUTDOWNCMD "/usr/bin/logger fake shutdown command"` to speed up your testing process
4. Define the `NOTIFYCMD` and `NOTIFYFLAG`s that will be used (varies by driver / UPS capabilities). I'm only using `ONBATT` and `ONLINE`.

### /etc/nut/upssched.conf
Determines what occurs at specific UPS events. Sets the shutdown timer that will be used to shut off the host and complement processes.

Be sure to uncomment the `CMDSCRIPT` and `PIPEFN` parameters and set appropriate paths

The timer you configure here determines when the server will shutdown. It's important to ensure that your shutdowns are synchronized. 

### /bin/upssched-cmd
May have world readable permissions by default. Mine is set to 750 with read access for the nut group since it contains NUT-specific passwords. 
```
sudo chmod 750 /bin/upssched-cmd
sudo chown root:nut /bin/upssched-cmd
```

In the `onbatt_shutdown` routine, 
* I call a script to shut down my router in 30s
* I shut down the primary UPS (Eaton 5PX) in 60s
* I shut down an unmonitored auxiliary UPS (also an Eaton 5PX w/ SNMP) in 60s
* I call `/sbin/upsmon -c fsd` to shutdown the host immediately

### /lib/systemd/system
These files are generated during the build process and represent the updated NUT systemd configuration:
* nut-driver-enumerator.path
* nut-driver-enumerator.service
* nut-driver\@.service
* nut-driver.target
* nut-monitor.service
* nut-server.service
* nut.target

### /lib/systemd/system-shutdow/nutshutdown
I commented out the single line in this file as it doesn't work with my configuration

### /usr/share/nut
The "cmdvartab" and "driver.list" files are included with the v2.8.0 tarball. These files are referenced by the NUT application. 

### Example installation scripts
These scripts were written to streamline the process of installing the minimum quantity of files needed by NUT. File ownership and permissions were assigned as needed. I'm not the most well-versed with Bash and wanted to gain some experience, so these scripts could probably use some improvement. If anything, they can be used to confirm where specific files are being installed. I wouldn't expect you to run them on your system!
