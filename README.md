# Packer CentOS 7 KVM example

[Packer](https://packer.io "Packer website") is a tool to automate the installation and provisioning of virtual machines to generate images for various platforms. You can have, for example, images for your test environment created with QEMU/KVM or Docker and images for your production environment created as Amazon AMI or VMware VMX images.

Basically, **Packer starts a VM in a private environment, feeds an ISO to the VM to install the operating system** (using kickstart, preseed or various other automation mechanisms) **and then waits until the VM restarts and is available via SSH** or WinRM. When it is available, Packer **can run different provisioners** (from bash scripts to your favourite tool like Ansible, Chef or Puppet) to setup the system as required. Once it's done provisioning, it will shut down the VM and possibly apply post-processors that can, for example, pack a VMware image made by multiple files in a single file and so on.

In this article I'll show you the steps to **create a CentOS 7 image on KVM** and explain some important settings.

First thing, you'll need Packer. You can download it from https://www.packer.io/downloads.html

```bash
# curl -O https://releases.hashicorp.com/packer/0.11.0/packer_0.11.0_linux_amd64.zip
# curl -O https://releases.hashicorp.com/packer/0.11.0/packer_0.11.0_SHA256SUMS
# curl -O https://releases.hashicorp.com/packer/0.11.0/packer_0.11.0_SHA256SUMS.sig
# gpg --recv-keys 51852D87348FFC4C
# gpg --verify packer_0.11.0_SHA256SUMS.sig packer_0.11.0_SHA256SUMS
# sha256sum -c packer_0.11.0_SHA256SUMS 2>/dev/null | grep OK
# unzip packer*.zip ; rm -f packer*.zip
# chmod +x packer
# mv packer /usr/bin/packer.io
```

I already did something "different" from the official documentation, sorry about that, but **CentOS and Fedora already have a completely unrelated program named packer in /usr/sbin/, so to avoid confusion I named the Packer binary packer.io**. All my examples will use this syntax, so make sure to keep that in mind when you'll check other examples on the official website or other blogs.

Let's make sure we have all we need to run the example. On my CentOS 7 host, I had to install:

```bash
# yum -y install epel-release
# yum -y install --enablerepo=epel qemu-system-x86
```

If you're running this example on a remote host, you'll probably want to **setup X11 forwarding** to be able to see the QEMU console. You'll need to edit your server's `/etc/ssh/sshd_config` file and make sure you have these options enabled:

```
X11Forwarding yes
X11UseLocalhost no
```

Then you'll need to restart sshd and make sure you have at least `xauth` installed:

```bash
# service sshd restart
# yum -y install xauth
```

At this point by logging to your remote host with the `-X` option to `ssh`, you should be able to forward X to your local system and see the QEMU graphical console:

```bash
# ssh -X user@remotehost 'qemu-system-x86_64'
```

If you still have problems, [this article](http://www.cyberciti.biz/faq/how-to-fix-x11-forwarding-request-failed-on-channel-0/) helped me solve a few issues: http://www.cyberciti.biz/faq/how-to-fix-x11-forwarding-request-failed-on-channel-0/

**Now you'll need a work directory**. One important thing to note is that Packer will use this directory, and subdirectories, as a stage for the files, including the VM disk image, so I highly recommend to create this workdir on a *fast storage* (SSD works best). In my case, I created it on my RAID 10 array and assigned ownership to my unprivileged user:

```bash
# mkdir -p /storage/packer.io/centos7-base
# chown velenux:velenux -R /storage/packer.io
```

At this point you should not need the root console anymore. If you have problems starting qemu/kvm you'll probably need to add your unprivileged user to the appropriate groups and login again.

We're finally ready to start exploring Packer. Our work directory will contain 3 main components: a **packer configuration file**, a **kickstart file** to setup our CentOS installation automatically and a **provisioning script** that will take care of post-installation setup of the virtual machine.

To make things easier I created a **public github repo** with an example you can clone on https://github.com/stardata/packer-centos7-kvm-example

The first thing we're going to examine is the packer configuration file, `centos7-base.json`:

```json
{
  "builders":
  [
    {
      "type": "qemu",
      "accelerator": "kvm",
      "headless": false,
      "qemuargs": [
        [ "-m", "2048M" ],
        [ "-smp", "cpus=1,maxcpus=16,cores=4" ]
      ],
      "disk_interface": "virtio",
      "disk_size": 100000,
      "format": "qcow2",
      "net_device": "virtio-net",

      "iso_url": "http://centos.fastbull.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso",
      "iso_checksum": "88c0437f0a14c6e2c94426df9d43cd67",
      "iso_checksum_type": "md5",

      "vm_name": "centos7-base",
      "output_directory": "centos7-base-img",

      "http_directory": "docroot",
      "http_port_min": 10082,
      "http_port_max": 10089,

      "ssh_host_port_min": 2222,
      "ssh_host_port_max": 2229,

      "ssh_username": "root",
      "ssh_password": "CHANGEME",
      "ssh_port": 22,
      "ssh_wait_timeout": "1200s",

      "boot_wait": "40s",
      "boot_command": [
        "<up><wait><tab><wait> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/c7-kvm-ks.cfg<enter><wait>"
      ],

      "shutdown_command": "shutdown -P now"
    }
  ],

  "provisioners":
  [
    {
      "type": "shell-local",
      "command": "tar zcf stardata-install.tar.gz stardata-install/"
    },
    {
      "type": "file",
      "source": "stardata-install.tar.gz",
      "destination": "/root/stardata-install.tar.gz"
    },
    {
      "type": "shell",
      "pause_before": "5s",
      "inline": [
        "cd /root/",
        "tar zxf stardata-install.tar.gz",
        "cd stardata-install/",
        "./install.sh",
        "yum clean all"
      ]
    }
  ]
}
```

I tried to arrange the contents to make it easier to read for newcomers.

The first thing you should notice is the **general structure of the file**: we have two sections, `builders` and `provisioners`.

In our example, the first is a list of only one element (the QEMU/KVM builder), but you could easily add more builders after that, to create images using different plugins.

In the `provisioners` section we have 3 different provisioners that will be run in sequence: the first runs a **command on the host** system, the second **transfers a file** (created/updated by the first) **to the VM** and the third **runs a series of commands on the VM**. We'll talk a bit more about them later.

Now let's examine our first `builder`: based on this configuration, Packer will run QEMU with 1 CPU/4 cores and 2G of RAM, creating a qcow2 virt-io disk with 100000M of space available. Note that qcow2 is a sparse file, or *"thin provision disk"*: the disk image will only use the space required and grow when required. Please notice how I set `"headless"` to `false`. This is a **boolean value, not a string**, and when you finish testing and debugging your Packer configuration you'll probably want to set it back to `true`.

The next set of parameters inform Packer of the URI where to find the installation ISO for this image. This ISO will be downloaded and cached locally during the first build, and you will probably want to pick a better mirror from: http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1511.iso

`vm_name` is pretty self-explanatory and `output_directory` is where the final image will be, if the build completes correctly.

The `http_*` parameters are required to setup the HTTP server that Packer will start during the build to serve files (for example, the kickstart file) to the virtual machine.

The `ssh_host_*` parameters specify the ports that will be redirected from the Host to the VM during the build. Packer utilizes ranges because it can run multiple builds (for multiple platforms) in parallel and allocates different ports for different builds. You can read more about that on the [official documentation](https://www.packer.io/docs/builders/qemu.html).

The next set of parameters specifies the values to use when accessing the VM via SSH. Note that the password must be the same you set in your kickstart and the `wait_timeout` is the maximum time that Packer will wait for the VM to become accessible via SSH. Considering it will have to install the distribution first, I set this to `1200s` (20m), altho in my tests the whole build process - including provisioning that happens after the system is available via SSH - took about 13m.

The `boot_wait` parameter sets a fixed amount of time that Packer will wait before proceeding with the `boot_command`; it's important to specify a value that is **long enough to allow the system to reach the distribution boot prompt, but short enough so that the default installation won't start**.

The `boot_command` parameter allows to emulate various key-presses to interact with the bootscreen. In my specific case, I'm emulating pressing the `up` key (to skip the media check), then `tab` to autocomplete the boot parameters based on the selected item and then I add the parameters required for a kickstart installation and emulate the pression of the `enter` key.
Running the build you'll see this happen on your screen without any interaction on your part!

Lastly, the `shutdown_command` is the command that will be run after the provisioners.

Before talking about the provisioners, it's worth examining the kickstart file `docroot/c7-kvm-ks.cfg`.

```kickstart
# reference: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/sect-kickstart-syntax.html
# based on one of our production installs with some modifications
# and some integrations from https://raw.githubusercontent.com/geerlingguy/packer-centos-7/master/http/ks.cfg

# Run the installer
install

# Use CDROM installation media
cdrom

# System language
lang en_US.UTF-8

# Keyboard layouts
keyboard us

# Enable more hardware support
unsupported_hardware

# Network information
network --bootproto=dhcp --hostname=centos7-test.stardata.lan

# System authorization information
auth --enableshadow --passalgo=sha512

# Root password
rootpw CHANGEME

# Selinux in permissive mode (will be disabled by provisioners)
selinux --permissive

# System timezone
timezone UTC

# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=vda

# Run the text install
text

# Skip X config
skipx

# Only use /dev/vda
ignoredisk --only-use=vda

# Overwrite the MBR
zerombr

# Partition clearing information
clearpart --none --initlabel

# Disk partitioning information
part pv.305 --fstype="lvmpv" --ondisk=vda --size=98000
part /boot --fstype="ext4" --ondisk=vda --size=1024 --label=BOOT
volgroup VGsystem --pesize=4096 pv.305
logvol /opt  --fstype="ext4" --size=5120 --name=LVopt --vgname=VGsystem
logvol /usr  --fstype="ext4" --size=10240 --name=LVusr --vgname=VGsystem
logvol /var  --fstype="ext4" --size=10240 --name=LVvar --vgname=VGsystem
logvol swap  --fstype="swap" --size=4096 --name=LVswap --vgname=VGsystem
logvol /  --fstype="ext4" --size=10240 --label="ROOT" --name=LVroot --vgname=VGsystem
logvol /tmp  --fstype="ext4" --size=5120 --name=LVtmp --vgname=VGsystem
logvol /var/log  --fstype="ext4" --size=10240 --name=LVvarlog --vgname=VGsystem
logvol /home  --fstype="ext4" --size=5120 --name=LVhome --vgname=VGsystem


# Do not run the Setup Agent on first boot
firstboot --disabled

# Accept the EULA
eula --agreed

# System services
services --disabled="chronyd" --enabled="sshd"

# Reboot the system when the install is complete
reboot


# Packages

%packages --ignoremissing --excludedocs
@^minimal
@core
kexec-tools
# unnecessary firmware
-aic94xx-firmware
-atmel-firmware
-b43-openfwwf
-bfa-firmware
-ipw2100-firmware
-ipw2200-firmware
-ivtv-firmware
-iwl100-firmware
-iwl1000-firmware
-iwl3945-firmware
-iwl4965-firmware
-iwl5000-firmware
-iwl5150-firmware
-iwl6000-firmware
-iwl6000g2a-firmware
-iwl6050-firmware
-libertas-usb8388-firmware
-ql2100-firmware
-ql2200-firmware
-ql23xx-firmware
-ql2400-firmware
-ql2500-firmware
-rt61pci-firmware
-rt73usb-firmware
-xorg-x11-drv-ati-firmware
-zd1211-firmware

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%post
yum -y upgrade
yum clean all
%end
```

As you can see the file is commented, so I will not spend too much time on it, but it's important to note how **the password is the same we set in the Packer configuration** and the **network options are set on DHCP**, because Packer will run a private network for the build and provide an IP address to the VM.
The **partitioning scheme** is similar to what we use in production and provided as an example, but I highly recommend you use your own partitioning scheme that you can retrieve in the file `/root/anaconda-ks.cfg` after a _"normal"_ installation.

After the operating system is installed and restarted, SSH becomes available and Packer proceeds to run the providers.

In our example, **the first provider runs a shell on the Host system** to update the content of `stardata-install.tar.gz`, so if you modify `stardata-install/install.sh` you'll be uploading the updated version to the VM.

The second provider, as we mentioned, copies `stardata-install.tar.gz` to the `/root/` directory in the VM.

The third and last provider runs a few commands to enter `/root/`, extract the tar.gz, enter `stardata-install/` and run `./install.sh` and then runs a `yum clean all` to cleanup the `yum` cache so our image will be even smaller.

We're ready for our first build. We're going to clone the repository and run `packer.io` with `PACKER_LOG=1` so we can see all the debug messages.

```bash
cd /storage/centos7-base/
git clone https://github.com/stardata/packer-centos7-kvm-example.git
cd packer-centos7-kvm-example
PACKER_LOG=1 packer.io build centos7-base.json
```

If everything works correctly, at the end of the build you'll have your qcow2-format image in `centos7-base-img/`.

For more information, you can check:

* [the official documentation](https://www.packer.io/docs/ "Packer documentation")
* [Jeff Geerling's repo for Vagrant and Ansible](https://github.com/geerlingguy/packer-centos-7)
