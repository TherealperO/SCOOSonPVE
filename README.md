I managed to get a VMware OpenServer 5.0.6a VM to run on Proxmox VE 7.

This guide isn't a step-by-step on how to export/import the VM from VMware to PVE. This guide picks up after all that's done.

**Note**: If your VMware VM uses IDE drives, I think this guide will still work if you substitute **wd** (IDE driver) where I used **blc** (Buslogic driver). In either case, make sure you mount the drives on the same IDs on both VMs.

In my case, the VMware VM was configured with SCSI controller (Buslogic). I configured the PVE VM with an LSI 53C895A SCSI controller and mounted the disks on the same IDs as on the VMware VM. I used Intel E1000 network adapter.

**Note 2:** ~~Set CPU type to 486 or log fills up with "i8254 timer period limited to 200000 ns" messages.~~ I think I've solved this by disabling short timers:

    vi /etc/conf/pack.d/clock/space.c
    
Change the disable_short_timers value to 1. You'll need to relink the kernel for this to take effect, but you can make the change before exporting from VMware, and when you relink the kernel later on in this guide, this change will be included as well. See https://www.scosales.com/ta/kb/127403.html for more info.

Although the disable_short_timers fix takes care of the log warnings, the VM seems to be about 10% faster on CPU performance alone when running it as 486. I've tried several different processor types, and 486 beats them all. This was tested running on Xeon X5670 and X5675 CPUs. YMMV.

You'll need the slha_btld drivers and the e1000 drivers:
    ftp://ftp.sco.com/pub/openserver5/507/drivers/slha_4.11.03/
    ftp://ftp.sco.com/pub/openserver5/507/drivers/eeG_5.0.7g/

I found the easiest way to get the drivers loaded without having to deal with floppy images is put them on the boot partition while still running on VMware. Download scoonpve.zip and extract to /stand. Make /stand writable first by issuing this command:

    /etc/btmnt -w    

Then extract the zip file before transferring the VM to PVE. Then you can remount read only with:

    /etc/btmnt -d

When booting on PVE you need to load the slha driver. Assuming your boot drive is SCSI Id 0, on the boot prompt enter:

    defbootstr link=slha Sdsk=slha(0,0,0)

If you boot drive is a different SCSI Id, change the last 0 to match it. Technically you're supposed to have 4th 0 (0,0,0,0) for lun, but in my experience it works fine without.. Probably defaults to 0. YMMV.
More info here: http://osr507doc.sco.com/en/HANDBOOK/SCSI_configuration.html

When prompted to insert the slha driver, just hit enter and it'll find it on the boot partition. That should get you booting into SCO. 

Start in maintenance mode and then reinstall slha driver and rebuild kernel (there may be a a better way of doing this, but you'll only do it once so no big deal):

    /etc/btldinstall /stand

Remove old SCSI drivers from kernel config:

    vi /etc/conf/cf.d/sdevice

Change the first column behind all the blc lines to N. Save and exit.

    vi /etc/conf/sdevice.d/blc

Change the first column behind all the blc lines to N. Save and exit.

    vi /etc/conf/cf.d/mscsi

Replace each instance of the old host adapter driver name (blc) with the new name (slha) in mscsi. Save and exit.

Relink kernel:

    /etc/conf/cf.d/link_unix -y

Reboot VM. System should boot normally.

Run 'custom' and select 'Software' → 'Install New'.
Install from your host machine using 'Media Images' in image directory /stand.
Install Intel Intel PRO/1000 drivers.
Exit Software Manager.

Run 'netconfig' and remove old adapter. Configure the new adapter.  Relink kernel. Reboot.

System should be mostly ready to go. 
