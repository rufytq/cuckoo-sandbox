# Cuckoo Installation Guide

# Requirements
## Installing Python (2.7) libraries
    sudo apt-get install python python-pip python-dev libffi-dev libssl-dev
    sudo apt-get install python-virtualenv python-setuptools
    sudo apt-get install libjpeg-dev zlib1g-dev swig

MongoDB:

    sudo apt-get install mongodb

Or PostgreSQL:

    sudo apt-get install postgresql libpq-dev

KVM machinery module:

    sudo apt-get install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils python-libvirt

## Virtualization Software
    echo deb http://download.virtualbox.org/virtualbox/debian xenial contrib |  sudo tee -a /etc/apt/sources.list.d/virtualbox.list
    wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- |  sudo apt-key add -
    sudo apt-get update
    sudo apt-get install virtualbox-5.2

## Installing TCPDump
    sudo apt-get install tcpdump apparmor-utils
    sudo aa-disable /usr/sbin/tcpdump

Or

    sudo apt-get install tcpdump

Set privileges:
    
    sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

Verify result:

    getcap /usr/sbin/tcpdump

Result will be as follow:

    /usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip

Install *setcap* if you don't have:

    sudo apt-get install libcap2-bin

## Installing Volatility
    git clone https://github.com/volatilityfoundation/volatility.git
    cd volatility
    sudo python setup.py install

## Installing M2Crypto
Install [SWIG](https://www.swig.org) first then M2Crypto:

    sudo apt-get install swig
    sudo pip install m2crypto==0.24.0

## Installing guacd
    sudo apt install libguac-client-rdp0 libguac-client-vnc0 libguac-client-ssh0 guacd

If you only want RDP support you can skip the installation of the `libguac-client-vnc0` and `libguac-client-ssh0` packages.

# Installing Cuckoo

## Create user
    sudo adduser cuckoo

For VirtualBox:

    sudo usermod -a -G vboxusers cuckoo

For KVM:

    sudo usermod -a -G libvirtd cuckoo

## Install Cuckoo
*Global* installation:

    sudo pip install -U pip setuptools
    sudo pip install -U cuckoo

Or install Cuckoo in `virtualenv` (**recommended**):

    virtualenv venv
    . venv/bin/activate
    (venv)
        pip install -U pip setuptools
        pip install -U cuckoo

> **Note**: Reasons for using `virtualenv`:
>
> - Cuckoo’s dependencies may not be entirely up-to-date, but instead pin to a known-to-work-properly version.
> - The dependencies of other software installed on your system may conflict with those required by Cuckoo, due to incompatible version requirements (and yes, this is also possible when Cuckoo supports the latest version, simply because the other software may have pinned to an older version).
> - Using a virtualenv allows non-root users to install additional packages or upgrade Cuckoo at a later point in time.
> - And simply put, virtualenv is considered a best practice.

## Configuration
    cuckoo init
    cuckoo community

### cuckoo.conf
- `machinery` in `[cuckoo]`:
    
    `virtualbox` or `vmware`

- `ip` and `port` in `[resultserver]`:
    ```
    ip: 192.168.56.1
    port: 2042
    ```

- `connection` in `[database]`:
    ```
    connection = mysql://root@127.0.0.1/cuckoo
    ```

### auxiliary.conf
    [sniffer]
    enabled = yes

    tcpdump = /path/to/tcpdump

### <machinery\>.conf
- `mode` = `headless` / `gui`
- `path` = `/path/to/vm`
- `interface`

```
[cuckoo1]
label = Win7x64

ip = 192.168.56.101
```

### reporting.conf
    [mongodb]
    enabled = yes

## Network Routing
    sudo iptables -t nat -A POSTROUTING -o eth0 -s 192.168.56.0/24 -j MASQUERADE
    sudo iptables -P FORWARD DROP
    sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
    sudo iptables -A FORWARD -s 192.168.56.0/24 -d 192.168.56.0/24 -j ACCEPT
    sudo iptables -A FORWARD -j LOG

    echo 1 |  sudo tee -a /proc/sys/net/ipv4/ip_forward
    sudo sysctl -w net.ipv4.ip_forward=1

# Install guest OS Windows 7
    # install additional dependencies
    sudo apt-get install virtualbox-ext-pack virtualbox-guest-additions-iso
    # use a variable to store VM name as we have to reuse it several times
    VM='Windows7-64bit'
    # create a VM (valid ostype values from `VBoxManage list ostypes`)
    VBoxManage createvm --name "$VM" --ostype Windows7_64 --register
    # set usable memory to 4 GB
    VBoxManage modifyvm "$VM" --memory 4096
    # create a hard disk (40 GB)
    VBoxManage createhd --filename "$VM.vdi" --size 40960
    # Attach the hard disk to the VM
    VBoxManage storagectl "$VM" --name "SATA Controller" --add sata --controller IntelAHCI
    VBoxManage storageattach "$VM" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium "$VM.vdi"
    # Create a DVD drive with the installation media
    VBoxManage storagectl "$VM" --name "IDE Controller" --add ide
    VBoxManage storageattach "$VM" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium /path/to/windows_installer.iso
    # Enable VirtualBox Remote Display Protocol
    VBoxManage modifyvm "$VM" --vrde on
    VBoxManage modifyvm "$VM" --vrdeport 5555
    # List host's network interfaces
    ifconfig
    # Setup bridged network on VM ($host_iface must be a valid interface on the host as shown by ifconfig comand)
    VBoxManage modifyvm "$VM" --nic1 bridged --bridgeadapter1 $host_iface
    # Power on
    VBoxManage startvm "$VM" --type headless 

Install some RDP client to view console/GUI:
```
sudo apt-add-repository -y ppa:remmina-ppa-team/remmina-next
sudo apt-get install remmina remmina-plugin-rdp libfreerdp-plugins-standard
```

Remote connect to port 5555 and follow setup Windows 7.

Finally install VirtualBox Guest Additions:
```
VBoxManage storageattach "$VM" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium /usr/share/virtualbox/VBoxGuestAdditions.iso
```

Configure IP and turn off Firewall.

Install Python and Python Pillow:
```
pip install pillow
```

Copy agent.py file to Win7, execute it and take snapshot.
```
VBoxManage snapshot "<Name of VM>" take "<Name of snapshot>" --pause
```

Power off and restore:
```
VBoxManage controlvm "<Name of VM>" poweroff
VBoxManage snapshot "<Name of VM>" restorecurrent
```

# Usage
    cuckoo

Open new console window and run:
```
cuckoo web
```
for Web interface or using API:
```
cuckoo api
```

## Submit
```
cuckoo submit --help
Usage: cuckoo submit [OPTIONS] [TARGET]...

  Submit one or more files or URLs to Cuckoo.

Options:
  -u, --url           Submitting URLs instead of samples
  -o, --options TEXT  Options for these tasks
  --package TEXT      Analysis package to use
  --custom TEXT       Custom information to pass along this task
  --owner TEXT        Owner of this task
  --timeout INTEGER   Analysis time in seconds
  --priority INTEGER  Priority of this task
  --machine TEXT      Machine to analyze these tasks on
  --platform TEXT     Analysis platform
  --memory            Enable memory dumping
  --enforce-timeout   Don't terminate the analysis early
  --clock TEXT        Set the system clock
  --tags TEXT         Analysis tags
  --baseline          Create baseline task
  --remote TEXT       Submit to a remote Cuckoo instance
  --shuffle           Shuffle the submitted tasks
  --pattern TEXT      Provide a glob-pattern when submitting a
                      directory
  --max INTEGER       Submit up to X tasks at once
  --unique            Only submit samples that have not been
                      analyzed before
  -d, --debug         Enable verbose logging
  -q, --quiet         Only log warnings and critical messages
  --help              Show this message and exit.
```

Example:
```
    cuckoo submit /path/to/binary
    cuckoo submit --url http://www.example.com
```

## REST API

Resource | Description |
:-- | :-- |
`POST` [/tasks/create/file]()     | Adds a file to the list of pending tasks to be processed and analyzed.
`POST` [/tasks/create/url]()      | Adds an URL to the list of pending tasks to be processed and analyzed.
`POST`[/tasks/create/submit]()   | Adds one or more files and/or files embedded in archives to the list of pending tasks.
`GET` [/tasks/list]()             | Returns the list of tasks stored in the internal Cuckoo database. You can optionally specify a limit of entries to return.
`GET` [/tasks/sample]()           | Returns the list of tasks stored in the internal Cuckoo database for a given sample.
`GET` [/tasks/view]()             | Returns the details on the task assigned to the specified ID.
`GET` [/tasks/reschedule]()       | Reschedule a task assigned to the specified ID.
`GET` [/tasks/delete]()           | Removes the given task from the database and deletes the results.
`GET` [/tasks/report]()           | Returns the report generated out of the analysis of the task associated with the specified ID. You can optionally specify which report format to return, if none is specified the JSON report will be returned.
`GET` [/tasks/screenshots]()      | Retrieves one or all screenshots associated with a given analysis task ID.
`GET` [/tasks/rereport]()         | Re-run reporting for task associated with a given analysis task ID.
`GET` [/tasks/reboot]()           | Reboot a given analysis task ID.
`GET` [/memory/list]()            | Returns a list of memory dump files associated with a given analysis task ID.
`GET` [/memory/get]()             | Retrieves one memory dump file associated with a given analysis task ID.
`GET` [/files/view]()             | Search the analyzed binaries by MD5 hash, SHA256 hash or internal ID (referenced by the tasks details).
`GET` [/files/get]()              | Returns the content of the binary with the specified SHA256 hash.
`GET` [/pcap/get]()               | Returns the content of the PCAP associated with the given task.
`GET` [/machines/list]()          | Returns the list of analysis machines available to Cuckoo.
`GET` [/machines/view]()          | Returns details on the analysis machine associated with the specified name.
`GET` [/cuckoo/status]()          |  Returns the basic cuckoo status, including version and tasks overview.
`GET` [/vpn/status]()             | Returns VPN status.
`GET` [/exit]()                   | Shuts down the API server.


# Features

## Dashboard

  Submit interface

  System info

## Submit

  Drag/click to upload a file or enter a URL or file hash

## Import

## Analysis Summary
```
  Metadata
```
-  Summary

-  Static Analysis

-  Extracted Artifacts

-  Behavioral Analysis

-  Network Analysis

-  Dropped Files

-  Dropped Buffers

-  Process Memory

-  Compare Analysis

-  Export Analysis

-  Reboot Analysis

-  Options

-  Feedback
