.. _my_awesome_server:

=================
My Awesome Server
=================

.. tip::

    This page is still under development. I'll be changing it a lot as I perfect how I'll setup my new server before I
    "put it in production" in my home.

I've had home servers since I was in high school in 2002. However I've never documented how I set them up before. Here
I'll be outlining the steps I took in setting up my current home Linux server. It's a general purpose server, acting as:

1. A file server for all of my media/backups/etc.
2. `Docker <https://www.docker.com/>`_ server.
3. Automated Bluray/DVD ripping (backups) station.
4. `Plex <https://www.plex.tv/>`_ media server.
5. Apple Time Machine backup server.
6. `Metrics <https://github.com/influxdata/chronograf>`_ collector and email alerting.
7. Tape backup server.
8. Automated video file transcoder.
9. Audio/video file ID3/metadata validator.

Hardware
========

My server will be going inside my `TV stand/cabinet`_. It'll share a case with my `pfSense <https://pfsense.org/>`_
custom router and be on a `UPS`_.

=============== ===========================================================================================
Case            `Travla T2241`_ dual mini-ITX with `Seasonic 250 watt`_ power supplies
Motherboard/CPU `Supermicro X10SDV-TLN4F-O`_ with Xeon D-1541
Memory          Kingston KVR24R17D8K4/64 (64GB)
M.2 SSD         Samsung 960 PRO 512GB
Storage HDDs    6x Seagate 10TB IronWolf Pro (ST10000NE0004)
SAS HBA         HighPoint RocketRAID 2721 4-Port Internal / 4 Port External
External Tape   *TBD*
=============== ===========================================================================================

Network
=======

While my server has 10GbE copper NICs and I've got a 16 port 10GbE copper managed switch at home, my stupid trash can
Mac Pro only has dual gigabit NICs. 10GbE copper thunderbolt NICs are too expensive as well.

To make the most of my Mac Pro I'll need to use VLANs. My server will use VLAN tagging and Samba will also listen on the
VLAN interface (separate subnet). The only other host on this VLAN will be my Mac Pro's second NIC. This way NIC1 will
be free to download data from my gigabit internet connection, whilst NIC2 will be dedicated to transfering files to my
server's Samba share. This lets me download files from the internet to my server via my Mac Pro at gigabit speeds.

VLANs
-----

.. describe:: VLAN1: Default

    The default VLAN. Used for managing the switch.

.. describe:: VLAN2: General

    The general VLAN for my home network. Every normal device communicates through it. Includes normal internet traffic.
    DHCP served by my pfSense box.

.. describe:: VLAN3: Guest

    Only used by the guest WiFi SSID. In case neighbors need to borrow my internet they can use this network, which will
    be separate from my general network. DHCP served by my pfSense box.

.. describe:: VLAN4: NAS

    Used for Samba. Only NIC2 on my Mac Pro and my server (VLAN tagging/trunking) will be on this VLAN.

.. describe:: VLAN5: ONT

    My gigabit "modem" will be on this VLAN, along with my pfSense box. Instead of plugging in the ONT directly to my
    pfSense box both will plug into my switch but be on their own VLAN. Easier cable management and if I ever want to
    get a second gigabit line I can plug in the second ONT into my switch.

Switchports
-----------

======= ============ ==================
Port    Device       VLAN
======= ============ ==================
1       ONT          5
2       pfSense WAN  5
3       pfSense LAN  2 (3 tagged)
4       Server       2 (4 tagged)
5       UPS          2
6       Chromecast   2
7       WiFi AP      2 (2+3 tagged)
8       Desk         2
9       Mac Pro NIC1 2
10      Mac Pro NIC2 4
11      *empty*      1
12      *empty*      3
13-16   *empty*      2
======= ============ ==================

Operating System
================

I'm using Fedora 25 Server installed on my M.2 SSD using `LUKS`_. I'll also be encryping all of my non-SSD hard drives
using their own LUKS key file (same file for all HDDs, but not SSD).

I follow https://gist.github.com/Robpol86/6226495 when setting up any Linux system, including my server. However I don't
setup my HDDs during setup, I leave them alone.

Sending Email
-------------

I want my server to send email alerts to me when events happen (e.g. a disk fails). I did write a guide about this at
:ref:`postfix_gmail_forwarding`. However instead of using Gmail to send email I'm going with https://www.sparkpost.com/
since they have a free tier and I can send emails in scripts using simple HTTP requests. Also email templates are a nice
feature.

To setup run:

.. code-block:: bash

    sudo dnf install postfix mailx cyrus-sasl{,-plain}

Then edit ``/etc/postfix/main.cf`` with the following. Replace ``<API-KEY>`` and ``<SENDING-DOMAIN>``.

.. code-block:: ini

    smtp_sasl_auth_enable = yes
    smtp_sasl_password_maps = static:SMTP_Injection:<API-KEY>
    relayhost = [smtp.sparkpostmail.com]:587
    smtp_sasl_security_options = noanonymous
    smtp_tls_security_level = encrypt
    header_size_limit = 4096000
    myorigin = <SENDING-DOMAIN>.com
    mydestination = <SENDING-DOMAIN>.com $myhostname localhost.$mydomain localhost

Then run:

.. code-block:: bash

    sudo tee /etc/aliases <<< 'root: <YOU>@gmail.com'
    sudo newaliases
    sudo systemctl start postfix.service
    sudo systemctl enable postfix.service
    mail -s "Test Email $(date)" <YOU>@gmail.com <<< "This is a test email."
    mail -s "Test Email for Root $(date)" root <<< "This is a test email."

You should receive both emails in your personal email account. If not make sure the numbers in your SparkPost's
dashboard's usage report have increased.

VLAN
----

.. code-block:: bash

    sudo nmcli con add type vlan ifname vlan4 dev eno3 id 4 ip4 10.168.192.4/24

That's it!

LUKS and Btrfs
==============

Here is where I format my storage HDDs. I want to use Btrfs since ZFS isn't first-class on Fedora and I want
Copy-On-Write with snapshots for backing up.

I also want to use Btrfs for RAID10 (RAID5 is a bad idea with 6x10TB and RAID6 still stresses all drives when one fails,
vs RAID10 stressing just one other drive). Since encryption isn't supported by Btrfs at this time I need to use LUKS.
Since I want to use LUKS with Btrfs my only option is to LUKS the drives first and then use Btrfs RAID ontop of them.

To avoid having to type in the same password six times on boot I'm instead using a random key file stored in /etc. It's
less safe but I'm encrypting my drives in case my server gets stolen. So since I'm using a traditional LUKS password on
my main SSD this key file will be encrypted anyway.

Run the following to set LUKS up:

.. code-block:: bash

    sudo dnf install cryptsetup btrfs-progs
    sudo sh -c 'umask 0277 && dd if=/dev/random of=/etc/hdd_key bs=1 count=128'
    (set -e; for d in /dev/sd[a-f]; do
        name=storage_$(lsblk -dno SERIAL $d)
        (! sudo grep -q "$name" /etc/crypttab)
        sudo fdisk -l $d |grep "Disk $d"
        sudo cryptsetup --key-file /etc/hdd_key --cipher aes-cbc-essiv:sha256 luksFormat $d
        uuid=$(lsblk -dno UUID $d)
        sudo tee -a /etc/crypttab <<< "$name UUID=$uuid /etc/hdd_key luks"
    done)
    sudo systemctl daemon-reload && sudo systemctl restart cryptsetup.target

Make sure all disks are in ``/dev/mapper``.

Btrfs
-----

Now it's time to create the Btrfs partition on top of LUKS as well as Btrfs subvolumes (for future snapshotting):

.. code-block:: bash

    # Create the Btrfs top volume.
    sudo mkfs.btrfs -L storage -m raid10 -d raid10 /dev/mapper/storage_*
    uuid=$(sudo btrfs filesystem show storage |grep -Po '(?<=uuid: )[0-9a-f-]+$')
    sudo tee -a /etc/fstab <<< "UUID=$uuid /storage btrfs autodefrag 0 2"
    sudo mkdir /storage; sudo mount -a
    # Create subvolumes.
    for n in Local Main Media Old Stuff Temporary TimeMachine; do
        sudo btrfs subvolume create /storage/$n
    done
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg

Reboot to make sure ``/storage`` is mounted.

Samba
=====

I'll have three Samba users on my server. Each user will have a separate password in Samba's database since things such
as printers may not store them 100% securely and I wouldn't want that to be an attack vector for my server (lifting the
password from the printer and then logging in and running sudo on my server).

======== ==========================================================================
User     Description
======== ==========================================================================
robpol86 The main user for my server. Will own everything besides "Stuff".
stuff    Separate user for "Stuff" in case I use it for malware testing/etc.
printer  Scanned documents will be put in "Temporary". Also writable by "robpol86".
======== ==========================================================================

Before installing anything I'll create additional users as per the table above and set permissions on the Btrfs
subvolumes (basically just directories from Samba's point of view).

.. code-block:: bash

    sudo useradd -p $(openssl rand 32 |openssl passwd -1 -stdin) -M -s /sbin/nologin stuff
    sudo useradd -p $(openssl rand 32 |openssl passwd -1 -stdin) -M -s /sbin/nologin printer
    sudo usermod -a -G printer robpol86
    sudo chown robpol86:robpol86 /storage/{Main,Media,Old,Temporary}
    sudo chown stuff:stuff /storage/Stuff
    sudo chmod 0750 /storage/{Main,Media,Old,Stuff}
    sudo chmod 0751 /storage/Temporary
    sudo setfacl -d -m u::rwx -m g::rx -m o::- /storage/{Main,Media,Old,Stuff,Temporary}
    mkdir -m 0770 /storage/Temporary/Printer; sudo chgrp printer $_  # Run as robpol86.
    sudo setfacl -d -m u::rwx -m g::rwx -m o::- /storage/Temporary/Printer

Next I'll install Samba, set Samba-specific passwords used by remote clients, and configure SELinux (other Samba guides
love to disable SELinux or set ``samba_export_all_rw`` which is basically the same as disabling SELinux).

.. code-block:: bash

    sudo dnf install samba policycoreutils-python-utils
    sudo smbpasswd -a stuff && sudo smbpasswd -e $_
    sudo smbpasswd -a printer && sudo smbpasswd -e $_
    sudo smbpasswd -a robpol86 && sudo smbpasswd -e $_
    sudo semanage fcontext -a -t samba_share_t /storage
    sudo semanage fcontext -a -t samba_share_t "/storage/(Main|Media|Old|Stuff|Temporary)(/.*)?"
    sudo restorecon -R -v /storage

Then write the following to ``/usr/local/bin/dfree_btrfs``:

.. literalinclude:: _static/dfree_btrfs.sh
    :language: bash

Now replace ``/etc/samba/smb.conf`` with:

.. literalinclude:: _static/smb.conf
    :language: ini

Finally run the following. Add firewall rules to force my OS X host to use the NAS VLAN for Samba.

.. code-block:: bash

    sudo chmod +x /usr/local/bin/dfree_btrfs
    sudo firewall-cmd --permanent --add-service=samba
    sudo firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=10.192.168.20 service name=samba drop"
    sudo systemctl restart firewalld.service
    sudo systemctl start smb.service
    sudo systemctl enable smb.service
    sudo systemctl start nmb.service
    sudo systemctl enable nmb.service

Alerting
========

I want everything I would normally check periodically on my server to be emailed to me instead. This will involve simple
cron jobs and more complicated emails derived from metrics.

.. code-block:: bash

    sudo dnf install smartmontools
    sudo systemctl start smartd

Add these to the **root** crontab. The email configuration from earlier in this document will take care of forwarding
root emails to my real email address.

.. code-block:: bash

    @hourly journalctl --since="1 hour ago" --priority=warning --quiet
    @monthly /usr/sbin/btrfs scrub start -Bd /storage

Setup InfluxDB and friends by following this guide (takes care of installing Docker too):
https://github.com/Robpol86/influxdb/wiki

References
==========

* http://nyeggen.com/post/2014-04-05-full-disk-encryption-with-btrfs-and-multiple-drives-in-ubuntu/

.. _TV stand/cabinet: https://www.standoutdesigns.com/products/media-console-solid-wood-majestic-ex-70-inch-wide
.. _Seasonic 250 watt: https://seasonic.com/product/ss-250-su-active-pfc-f0/
.. _UPS: http://www.apc.com/shop/us/en/products/APC-Smart-UPS-1500VA-LCD-RM-2U-120V/P-SMT1500RM2U
.. _Travla T2241: http://www.travla.com/business/index.php?id_product=49&controller=product
.. _Supermicro X10SDV-TLN4F-O: http://www.supermicro.com/products/motherboard/Xeon/D/X10SDV-TLN4F.cfm
.. _LUKS: https://fedoraproject.org/wiki/Disk_Encryption_User_Guide
