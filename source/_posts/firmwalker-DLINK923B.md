---
title: D-Link 923B路由器敏感信息搜集
index_img: img/article/iot-1/image-20220914135931400.png 
excerpt: /root/iot/D-LINK923B/2k/root敏感信息
---

***Firmware Directory***
/root/iot/D-LINK923B/2k/root
***Search for password files***
##################################### passwd
t/bin/passwd
t/etc/passwd
t/var/lib/opkg/alternatives/passwd

##################################### shadow
t/etc/shadow

##################################### *.psk

***Search for Unix-MD5 hashes***


***Search for SSL related files***
##################################### *.crt

##################################### *.pem

##################################### *.cer

##################################### *.p7b

##################################### *.p12

##################################### *.key


***Search for SSH related files***
##################################### authorized_keys

##################################### *authorized_keys*

##################################### host_key

##################################### *host_key*

##################################### id_rsa

##################################### *id_rsa*

##################################### id_dsa

##################################### *id_dsa*

##################################### *.pub


***Search for files***
##################################### *.conf
t/etc/wpa_supplicant.conf
t/etc/avahi/avahi-daemon.conf
t/etc/pimd.conf
t/etc/mdev.conf
t/etc/AR6004_AP1_hostapd.conf
t/etc/inadyn-mt.conf
t/etc/dbus-1/system.d/avahi-dbus.conf
t/etc/dbus-1/session.conf
t/etc/dbus-1/system.conf
t/etc/hostapd.conf
t/etc/host.conf
t/etc/AR6003_hostapd.conf
t/etc/igd/miniupnpd.conf
t/etc/nsswitch.conf
t/etc/AR6004_hostapd.conf
t/etc/radvd.conf
t/etc/AR6003_AP1_hostapd.conf

##################################### *.cfg

##################################### *.ini


***Search for database related files***
##################################### *.db

##################################### *.sqlite

##################################### *.sqlite3


***Search for shell scripts***
##################################### shell scripts
t/etc/rc4.d/S99rmnologin.sh
t/etc/mdev/usb.sh
t/etc/mdev/automountsdcard.sh
t/etc/mdev/automounthdd.sh
t/etc/mdev/find-touchscreen.sh
t/etc/bash_completion.d/gdbus-bash-completion.sh
t/etc/bash_completion.d/gsettings-bash-completion.sh
t/etc/rc2.d/S99rmnologin.sh
t/etc/rcS.d/S04modutils.sh
t/etc/rcS.d/S02sysfs.sh
t/etc/rcS.d/S38devpts.sh
t/etc/rcS.d/S06alignment.sh
t/etc/rcS.d/S45mountnfs.sh
t/etc/rcS.d/S39hostname.sh
t/etc/rcS.d/S10checkroot.sh
t/etc/rcS.d/S02banner.sh
t/etc/rcS.d/S37populate-volatile.sh
t/etc/rcS.d/S35mountall.sh
t/etc/rcS.d/S55bootmisc.sh
t/etc/rcS.d/S01keymap.sh
t/etc/rc0.d/S25save-rtc.sh
t/etc/rc0.d/S31umountnfs.sh
t/etc/rc6.d/S25save-rtc.sh
t/etc/rc6.d/S31umountnfs.sh
t/etc/init.d/modutils.sh
t/etc/init.d/banner.sh
t/etc/init.d/devpts.sh
t/etc/init.d/checkroot.sh
t/etc/init.d/hostname.sh
t/etc/init.d/bootmisc.sh
t/etc/init.d/save-rtc.sh
t/etc/init.d/sysfs.sh
t/etc/init.d/set-hwver.sh
t/etc/init.d/mountnfs.sh
t/etc/init.d/keymap.sh
t/etc/init.d/hwclock.sh
t/etc/init.d/mountall.sh
t/etc/init.d/umountnfs.sh
t/etc/init.d/alignment.sh
t/etc/init.d/populate-volatile.sh
t/etc/init.d/rmnologin.sh
t/etc/rc5.d/S99rmnologin.sh
t/etc/qdt_rc0clean.sh
t/etc/rc3.d/S99rmnologin.sh

***Search for other .bin files***
##################################### bin files
t/lib/firmware/ath6k/AR6004/hw3.0/bdata.bin
t/lib/firmware/ath6k/AR6004/hw3.0/otp.bin
t/lib/firmware/ath6k/AR6004/hw3.0/fw-3.bin
t/lib/firmware/ath6k/AR6004/hw3.0/utf.bin
t/lib/firmware/ath6k/AR6003/hw2.1.1/otp.bin
t/lib/firmware/ath6k/AR6003/hw2.1.1/bdata.SD31.bin
t/lib/firmware/ath6k/AR6003/hw2.1.1/athtcmd_ram.bin
t/lib/firmware/ath6k/AR6003/hw2.1.1/data.patch.hw3_0.bin
t/lib/firmware/ath6k/AR6003/hw2.1.1/athwlan_router.bin

***Search for patterns in files***
-------------------- upgrade --------------------
t/sbin/fotad
t/bin/lcdd
t/bin/qmiweb
t/bin/tinyproxy
t/bin/agent
t/bin/lcdtest
t/var/lib/opkg/info/busybox-syslog.prerm

-------------------- admin --------------------
t/bin/amapi
t/bin/tr069
t/bin/qmiweb
t/bin/appmgr
t/bin/agent
t/etc/services
t/etc/passwd
t/etc/passwd-
t/lib/libamapi.so
t/WEBSERVER/www/cgi-bin/qcmap_auth
t/WEBSERVER/www/QCMAP_login.html
t/var/lib/opkg/info/base-passwd.preinst

-------------------- root --------------------
t/sbin/ip.iproute2
t/sbin/readprofile.util-linux
t/sbin/shutdown.sysvinit
t/sbin/fdisk.util-linux
t/sbin/dnsmasq_qmi
t/sbin/swapon.util-linux
t/sbin/dlna6_dms
t/sbin/adbd
t/sbin/tc
t/sbin/miniupnpd
t/sbin/ddns_qmi
t/sbin/ctrlaltdel
t/sbin/killall5
t/sbin/cfdisk
t/bin/tinylogin
t/bin/busybox
t/bin/qmiweb
t/bin/mount.util-linux
t/bin/radvd
t/bin/login.shadow
t/bin/tinyproxy
t/bin/ramond
t/bin/agent
t/bin/umount.util-linux
t/etc/login.defs
t/etc/default/rcS
t/etc/default/volatiles/00_core
t/etc/shadow
t/etc/securetty
t/etc/dbus-1/system.d/avahi-dbus.conf
t/etc/limits
t/etc/init.d/checkroot.sh
t/etc/init.d/bootmisc.sh
t/etc/init.d/usb
t/etc/init.d/hwclock.sh
t/etc/init.d/power_config
t/etc/init.d/avahi-daemon
t/etc/group
t/etc/udhcpc.d/udhcpc.script
t/etc/gshadow
t/etc/passwd
t/etc/group-
t/etc/passwd-
t/lib/libjansson.so
t/lib/libmount.so.1
t/lib/libqutils.so
t/lib/libblkid.so.1
t/var/lib/opkg/info/shadow.postinst
t/var/lib/opkg/info/dbus-1.preinst
t/var/lib/opkg/info/base-passwd.preinst
t/var/lib/opkg/info/base-passwd.control
t/var/lib/opkg/info/avahi-daemon.preinst
t/var/lib/opkg/info/base-files.list

-------------------- password --------------------
t/sbin/losetup.util-linux
t/sbin/ddns_qmi
t/bin/amapi
t/bin/tr069
t/bin/tinylogin
t/bin/busybox
t/bin/qmiweb
t/bin/mount.util-linux
t/bin/login.shadow
t/bin/appmgr
t/bin/agent
t/bin/qmodmd
t/bin/umount.util-linux
t/etc/login.defs
t/etc/tr069/param.xml
t/etc/tr069/igd_10.xml
t/etc/inadyn-mt.conf
t/lib/libc.so.6
t/lib/libqutils.so
t/WEBSERVER/www/QCMAP_Account.html
t/WEBSERVER/www/QCMAP_login.html
t/WEBSERVER/www/QCMAP_Account_Help.html
t/var/lib/opkg/info/base-passwd.control
t/var/lib/opkg/info/avahi-autoipd.postinst
t/var/lib/opkg/info/shadow.control
t/disk/internal-storage.iso

-------------------- passwd --------------------
t/sbin/vipw.shadow
t/bin/amapi
t/bin/tinylogin
t/bin/amgrevt
t/bin/busybox
t/bin/qmiweb
t/bin/login.shadow
t/bin/appmgr
t/bin/agent
t/bin/qmodmd
t/etc/login.defs
t/etc/services
t/etc/init.d/populate-volatile.sh
t/etc/nsswitch.conf
t/etc/busybox.links
t/lib/libc.so.6
t/lib/libnss_compat.so.2
t/lib/libqutils.so
t/lib/libnss_files.so.2
t/lib/libamapi.so
t/var/lib/opkg/alternatives/passwd
t/var/lib/opkg/info/shadow.postinst
t/var/lib/opkg/info/busybox-mountall.control
t/var/lib/opkg/info/busybox-mdev.control
t/var/lib/opkg/info/dbus-1.preinst
t/var/lib/opkg/info/base-passwd.preinst
t/var/lib/opkg/info/base-passwd.control
t/var/lib/opkg/info/avahi-daemon.preinst
t/var/lib/opkg/info/avahi-autoipd.postinst
t/var/lib/opkg/info/shadow.prerm
t/var/lib/opkg/info/avahi-daemon.control
t/var/lib/opkg/info/shadow.list
t/var/lib/opkg/info/router.list
t/var/lib/opkg/info/dbus-1.control
t/var/lib/opkg/info/busybox-syslog.control
t/var/lib/opkg/info/busybox.control
t/var/lib/opkg/info/tinylogin.list
t/var/lib/opkg/info/task-core-boot.control
t/var/lib/opkg/status

-------------------- pwd --------------------
t/bin/amapi
t/bin/tinylogin
t/bin/busybox
t/bin/qmiweb
t/bin/login.shadow
t/bin/appmgr
t/bin/ramond
t/bin/qmodmd
t/etc/busybox.links
t/lib/libc.so.6
t/WEBSERVER/www/QCMAP_login.html
t/var/lib/opkg/alternatives/pwd

-------------------- dropbear --------------------
t/etc/init.d/dropbear
t/var/lib/opkg/info/dropbear.postrm
t/var/lib/opkg/info/dropbear.control
t/var/lib/opkg/info/dropbear.prerm
t/var/lib/opkg/info/dropbear.list
t/var/lib/opkg/info/dropbear.postinst
t/var/lib/opkg/status

-------------------- ssl --------------------
t/bin/tinyproxy
t/bin/agent
t/etc/services
t/var/lib/opkg/info/libcrypto1.0.0.control
t/var/lib/opkg/info/libssl1.0.0.control
t/var/lib/opkg/info/openssl.control
t/var/lib/opkg/info/openssl.list

-------------------- private key --------------------
t/bin/agent

-------------------- telnet --------------------
t/bin/busybox
t/etc/services
t/etc/default/rcS
t/etc/busybox.links
t/var/lib/opkg/alternatives/telnet

-------------------- secret --------------------
t/bin/dhcp6s
t/bin/dhcp6c

-------------------- pgp --------------------
t/lib/libresolv.so.2

-------------------- gpg --------------------
t/disk/internal-storage.iso

-------------------- token --------------------
t/sbin/iwconfig
t/sbin/adbd
t/sbin/tc
t/bin/busybox
t/bin/qmiweb
t/bin/radvd
t/bin/ramond
t/lib/libjansson.so
t/lib/libc.so.6
t/lib/ld-linux.so.3
t/lib/libsqlite3.so.0
t/lib/libmount.so.1
t/lib/libqutils.so
t/lib/libblkid.so.1
t/lib/libpcap.so.1
t/WEBSERVER/www/QCMAP_Firewall.html
t/WEBSERVER/www/QCMAP_WLAN.html

-------------------- api key --------------------

-------------------- oauth --------------------


***Search for web servers***
##################################### search for web servers
##################################### apache

##################################### lighttpd

##################################### alphapd

##################################### httpd
t/sbin/httpd
t/var/lib/opkg/alternatives/httpd


***Search for important binaries***
##################################### important binaries
##################################### ssh
t/var/lib/opkg/alternatives/ssh

##################################### sshd

##################################### scp
t/var/lib/opkg/alternatives/scp

##################################### sftp

##################################### tftp
t/bin/tftp
t/var/lib/opkg/alternatives/tftp

##################################### dropbear
t/etc/dropbear
t/etc/init.d/dropbear

##################################### busybox
t/bin/busybox

##################################### telnet
t/bin/telnet
t/var/lib/opkg/alternatives/telnet

##################################### telnetd
t/sbin/telnetd
t/var/lib/opkg/alternatives/telnetd

##################################### openssl


***Search for ip addresses***
##################################### ip addresses
0.0.0.0
127.0.0.1
1.4.12.1
169.254.0.0
169.254.255.255
192.168.0.0
192.168.0.1
192.168.0.20
192.168.0.40
192.168.1.0
192.168.10.1
192.168.10.109
192.168.1.1
192.168.1.20
192.168.1.40
192.168.1.8
192.168.2.1
192.168.2.20
192.168.2.40
192.168.50.1
192.168.50.2
192.168.7.0
192.168.7.1
192.168.7.113
192.168.7.2
224.0.0.0
255.255.0.0
255.255.255.0
255.255.255.255
4.1.4.2
4.1.4.3
88.22.44.13

***Search for urls***
##################################### urls
http://0pointer.de
http://avahi.org
http://codeaurora.org
http://code.google.com
http://dbus.freedesktop.org
http://developer.android.com
http://download.savannah.gnu.org
http://downloads.sourceforge.net
http://ebtables.sourceforge.net
http://expat.sourceforge.net
http://ftp.gnome.org
http://kernel.org
http://linuxwireless.org
http://matt.ucc.asn.au
http://miniupnp.free.fr
http://netfilter.org
http://packages.debian.org
http://pkg-shadow.alioth.debian.org
http://savannah.nongnu.org
http://sites.google.com
http://sourceforge.net
http://support.cdmatech.com
http://theory.uwinnipeg.ca
http://tinylogin.busybox.net
http://wireless.kernel.org
http://www.angstrom-distribution.org
http://www.busybox.net
http://www.eglibc.org
http://www.freebsd.org
http://www.freedesktop.org
http://www.gnu.org
http://www.hpl.hp.com
http://www.iana.org
http://www.info-zip.org
http://www.infradead.org
http://www.linuxfoundation.org
http://www.linux-mtd.infradead.org
http://www.mylan
http://www.netfilter.org
http://www.openssl.org
http://www.quantatw.com
http://www.tcpdump.org
http://www.w3.org
http://www.xmlsoft.org
http://www.zlib.net
http://zlib.net

***Search for emails***
##################################### emails
ajt@debian.org
bdschuym@pandora.be
Bruce@Pixar.com
bruno.randolf@4g-systems.biz
dag@wieers.com
default@no-ip.com
miquels@cistron.nl
openembedded-core@lists.openembedded.org
rok.papez@arnes.si
rpurdie@openedhand.com
sandman@handhelds.org
sebastien.estienne@gmail.com
srb@cuci.nl
Tim@Rikers.org
walters@debian.org
