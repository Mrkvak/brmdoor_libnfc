# Brmdoor via libnfc

This is an access-control system implementation via contactless ISO 14443A cards
and a PN53x-based reader. So you basically swipe your card, and if it's in
database, the door unlocks.

Info about authorized users and their cards and keys is stored in sqlite database.

This was originally designed for Raspberry (Raspbian), but it also runs on
desktop PC if you have the PN532 USB reader.

The daemon is implemented in `brmdoor_nfc_daemon.py`.

## NFC smartcard API

This project shows how to use libnfc from python to send APDUs to NFC
smartcards. Have a look at `test_nfc.py` for some examples, currently it
shows four interactions with NFC smartcards:

* read NDEF message from token (Mifare Desfire, Yubikey Neo)
* do HMAC-SHA1 authenthication (Yubikey Neo)
* read Track 2 Equivalent Data from Visa
* execute signature for payment on Mastercard

It is much more general in use than to use it as authenthicator to open door.

## Building and dependencies

You need just to run `make`. Additional dependencies:

- [libnfc](https://github.com/nfc-tools/libnfc/releases), in Debian and Ubuntu as libnfc-dev
- [libfreefare](https://github.com/nfc-tools/libfreefare), in Debian and Ubuntu install libfreefare-bin and libfreefare-dev
- [python-axolotl-curve25519](https://github.com/tgalal/python-axolotl-curve25519), in Ubuntu and Debian install python-axolotl-curve25519
- [SWIG version 2](http://www.swig.org/) - to generate Python-C++ bindings, SWIG 3 is known to cause segfaults sometimes
- [WiringPi2 pythonic binding](https://github.com/WiringPi/WiringPi2-Python) (for switching lock on Raspberry)
- [python-irc](https://pypi.python.org/pypi/irc) >= 16.0, use "pip install irc", the one in repos is old
- [pysftp](https://pypi.org/project/pysftp/) - for uploading SpaceAPI-formatted status to some host
  - optional runtime dependency, not needed unless you set SFTP SpaceAPI upload to true

All dependencies can be installed on Ubuntu or Debian/Raspbian via:

    apt install libnfc-dev libfreefare-bin libfreefare-dev python-axolotl-curve25519 swig2.0 python-dev
    pip install irc wiringpi2 pysftp

To build, just run make:

    make

## Howto

1. Create the database

        python create_authenticator_db.py authenthicator_db.sqlite

2. Copy sample config file, edit your pins, DB file location, timeouts

        cp brmdoor_nfc.config.sample brmdoor_nfc.config

3. Add some users

  - either authentication by UID, e.g.:

        ./brmdoor_adduser.py -c brmdoor_nfc.config -a uid 34795FCC SomeUserName

  - authentication by Yubikey's HMAC-SHA1 programmed on slot 2

        ./brmdoor_adduser.py -c brmdoor_nfc.config -a hmac 40795FCCAB0701 SomeUserName 000102030405060708090a0b0c0d0e0f31323334

  - to program Yubikey slot 2 to use HMAC with given key (requires package `yubikey-personalization`), use:

        ykpersonalize -2 -ochal-resp -ohmac-sha1 -ohmac-lt64 -oserial-api-visible
        
  - authentication using signed UID as NDEF message on Desfire:
  
        ./brmdoor_adduser.py -c brmdoor_nfc.config -a ndef 04631982cc2280 SomeUserName
  
  - you need to generate Ed25519 keypair, store the private key somewhere safe and put the public in config file
  
        ./generate_ed25519_keypair.py
        
  - you need to program the Desfire card to have the signature
       
        ./write_signed_ndef_on_desfire.py private_key_in_hex
        
Finally, run the daemon:

        sudo python brmdoor_nfc_daemon.py brmdoor_nfc.config

## Configuring libnfc devices

If you have PN532 device on other bus than USB (e.g. SPI), first search for it using:

    sudo nfc-scan-device -i

After that, create file `/etc/nfc/libnfc.conf` with line describing your device
from `nfc-scan-device` above, e.g. for SPI device:

    device.connstring = "pn532_spi:/dev/spidev0.0"

This daemon expects the library to be already configured to find the PN532 device.

If you installed libnfc from source, the default directory might be
`/usr/local/etc/nfc` instead of `/etc/nfc`.

## Conflicts with other running software - pcscd, pn533 kernel modules

If you have `pcscd` running, it will take over the reader and you can't use it. Kill/stop pcscd service/process if running.

Similarly, you have to blacklist `pn533` and `pn533_usb` kernel modules (usually in a file like `/etc/modprobe.d/blacklist.conf`).

## Startup with systemd and GNU screen

Example of startup unit for systemd, put in `/etc/systemd/system/brmdoor.service` and this repo cloned in `/root/brmdoor_libnfc`:

    [Unit]
    Description=brmdoor
     
    [Service]
    Type=forking
    User=root
    ExecStart=/usr/bin/screen -L -d -m -S brmdoor
    WorkingDirectory= /root/brmdoor_libnfc/
     
    [Install]
    WantedBy=multi-user.target


After adding the service file, run `systemctl daemon-reload` to notify systemd that unit was added. 
To enable automatic startup, use `systemctl enable brmdoor.service`.

A `/root/.screenrc` file that will run the daemon in detached screen:

    autodetach on
    startup_message off 

    screen -t brmdoor 0 /root/brmdoor_libnfc/brmdoor_start.sh


## Security considerations

Using SFTP for upload of status should be used with "internal-sftp" setting. This chroots the upload user's directory,
doesn't allow script or code execution. You need to chown the directory to root and make it not writable by non-root
users (requirement for internal-sftp). E.g. make `brmdoor-web` (used for sftp upload) user part of `sftp` group and have
  
    Subsystem sftp internal-sftp

    Match Group sftp
       ChrootDirectory %h
       ForceCommand internal-sftp
       AllowTcpForwarding no

For SFTP upload to work, target host needs to already to be in `~/.ssh/known_hosts` when making connection, otherwise
you'll get an exception. Simply connect via command-line sftp before running, check and accept the fingeprint beforehand.

## Known bugs (TODO)

* IRC disconnect is sometimes detected late, e.g. when trying to send message that door was open. This
  causes the message to be lost, but the reconnect will kick in
* Freenode loses packets (RST) seeming silent connection to be still alive when they are not.
* Periodic PING could theoretically solve this, but when I tried I got kicked out, so also you need to find the right
  interval
  
## Notes

You could use Android Host Card Emulation to emulate a Desfire - it actually just expects one application, D2760000850101.

See an [example of HCE NDEF emulation](https://github.com/TechBooster/C85-Android-4.4-Sample/blob/master/chapter08/NdefCard/src/com/example/ndefcard/NdefHostApduService.java).

You could just modify `write_signed_ndef_on_desfire.py` to write out the JSON into a file and then put the 
generated NDEF file into application so it will respond with it when
