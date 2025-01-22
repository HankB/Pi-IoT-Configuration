# Pi_IoT_configuration

Playbooks for setting up Raspberry Pis used for IoT stuff.

Playbooks to use after using `rpi-imager` to install the image and configure initial settings. Initial settings managed by `rpi-imager` include

* hostname
* WiFi credentials and country
* user
* timezone
* localization (keyboard)
* enable SSH

## main tasks

These are composed of two tasks, one nefore and one after the target is first booted.

* Pre boot - Following `rpi-imager` and before first boot, complete some configuration items.
* Post boot - Following the first boot in the target H/W include playbooks to
    * Perform functions common to all installations
    * Perform application specific functions
    * Configure R/O filesystems (or just use existing playbooks for this.)

### Pre boot

* Configure reboot pushbutton.
* Disable passwordless `sudo`.
* MAC address spoofing
* NTP server

`pre-boot.yml`

This playbook is invoked directly and will take the desired WiFi MAC address, NTP server IP and mount points as environment variable arguments. It is necessary to mount the boot and root partitions before invocation so the playbook can access them. Typical usage:

```text
ansible-playbook pre-boot.yml -b -K --extra-vars \
    "root_mount=/media/hbarta/rootfs \
    boot_mount=/media/hbarta/bootfs \
    spoof_mac=60:e3:27:1a:08:3e  \
    ntp_server=192.168.1.1"
```

### generic post boot

`post-boot.yml` will be invoked by the application specific playbooks.

* Update/upgrade (and report upgraded packages)
* Install checkmk agent
* Install desirable packages (e.g. `vim`, `tree` and so on.)

### Application install

These are expected to be specific for each installation. Some, such as installing DS18B20 or BME280 sensors can be abstracted and used by several hosts.

### Structure

Each host will have a main main playbook eponymously named, for example the playbook to configure `puyallup` will be `puyallup-install.yml`. Each will

* Invoke `post-boot.yml` to update/upgrade, install some apps ...
* Invoke or hard code any application specific playbooks.
* Enable the readonly filesystem. (Or not. More convenient to do after everything is working.)

*Before running any of these playbooks it is necessary to enable passwordless SSH to the target.*

## host specific notes

These playbooks invoke `post-boot.yml` and any further playbooks to complete the configuration required for the specified host.

### puyallup and dorman

And any others that operate with a single DS18B20 sensor.

```text
ansible-playbook puyallup-install.yml -i inventory -l rpios32 -b -K \
    --extra-vars "checkmk_agent=check-mk-agent_2.2.0p16-1_all.deb \
    location=lab measurement=10g_tank interval=3 locale=en_US.UTF-8"
```

### `sodus`

The BME280 binary needs to be built for a 32 bit RpiOS target (Pi Zero) and the easiest way to do this is build on `canby` and run the `sodus-install.yml` playbook there.

```text
ansible-playbook sodus-install.yml -i inventory -l sodus -b -K \
    --extra-vars "checkmk_agent=check-mk-agent_2.2.0p16-1_all.deb \
    location=bedroom measurement=temp_humidity=3 locale=en_US.UTF-8"
```

### `brandywine`

Development of the config files for this proceed on `canby` and with the ephemeral host `nbw` (new BrandyWine.) The supported sensors include

* 3x DS18B20
    * Freezer temperature
    * Outside temperature
    * Ambient (inside garage) temperature.
* AHT20 temp/humidity - outside conditions
* BME280 (BMP280) temp/humidity/pressure (temp/humidity) - outside conditions
* HC-SR04 distance sensor to determine garage door open/closed.

```text
ansible-playbook brandywine-install.yml -i inventory -l nbw -b -K \
    --extra-vars "checkmk_agent=check-mk-agent_2.2.0p16-1_all.deb \
     locale=en_US.UTF-8" \
     --tags AHT20
```

TODO: ID requirements.

## Errata

* `rpi-imager` 1.8.3 does not set the hostname when flashing a Lite image. Update: These things get set on first boot so the settings in the pre-boot image don't tell the whole story.
