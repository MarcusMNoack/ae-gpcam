# ae-gpcam
Autonomous experiment at XPD using gpCAM.

This repository is largely a copy of https://github.com/bluesky/bluesky-pods.

The autonomous experiment uses four processes:
 * bluesky
 * xpdan
 * ROI reduction server
 * adaptive gpCAM server

These processes communicate using Zero MQ (0MQ) and Redis:
```
  RE ---> 0MQ "raw" ---> xpdan ---> 0MQ "an" --->---->|
  ^                                                   |
  |<--- redis <--- gpcam <--- 0MQ "rr" roi_reducer <--|
```

The script `not_xpdan_server.py` is provided to simulate xpdan by simply forwarding
documents from the RunEngine to the ROI reduction server.

### Run the demonstration in a virtual machine

An easy way to run all four processes is with a Vagrant virtual machine (VM).
A `Vagrantfile` is provided that installs podman and related prerequisites on CentOS 8.
Install [Vagrant](vagrantup.com) and [VirtualBox](https://www.virtualbox.org/), then clone this repository.
```
host:~ user$ git clone https://github.com/NSLS-II-XPD/ae-gpcam.git
host:~ user$ cd ae-gpcam
```

#### Install test data
Download TiCu_export.tar.gz and unpack it to the `ae-gpcam` directory (on the host, not the Vagrant VM).
```
host:ae-gpcam user$ ls -l
total 28160
-rw-r--r--   1 user  staff      1528 Oct 16 15:38 LICENSE
-rw-r--r--   1 user  staff      9550 Nov 12 13:56 README.md
drwxr-xr-x@ 93 user  staff      2976 Apr 16  2020 TiCu_export
-rw-r--r--@  1 user  staff  13379447 Nov  2 14:35 TiCu_export.tar.gz
-rw-r--r--   1 user  staff      3950 Nov 12 12:52 Vagrantfile
drwxr-xr-x  15 user  staff       480 Nov 12 13:23 ae_gpcam
```

#### Clone gpCAM
You will need read permission for the gpCAM repository on BitBucket (left as an exercise).
Clone the gpCAM repository to the host in the same directory as the Vagrantfile. The VM
will mount the gpCAM source as /vagrant/gpcamv4and5. When the VM is built gpCAM will be
installed from source in a conda environment called `gpcam`.
```
host:ae-gpcam user$ git clone git@bitbucket.org:MarcusMichaelNoack/gpcamv4and5.git
```

#### Build a Virtual Machine with Vagrant
Now build a VM:
```
host:ae-gpcam user$ vagrant up
```
Podman images will be created the first time the VM starts.

When you are done with this demonstration and want to shut down but not delete the VM use this:
```
host:ae-gpcam user$ vagrant halt
```
When you are ready to delete the Vagrant VM use this:
```
host:ae-gpcam user$ vagrant destroy
```

If when starting up for the first time you get errors implying you can
not vboxfs is not available, you may need to upgrade/install the
kernel headers so that the "Guest Extensions" will build in the guest.
This supports important functionality like mounting directories from
the host into the guest.

```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ sudo yum install kernel kernel-devel kernel-headers gcc make perl elfutils-libelf-devel bzip2 tar -y
host:ae-gpcam user$ vagrant halt
host:ae-gpcam user$ vagrant up
```

#### Start the demonstration processes
Open a terminal and SSH into the VM.
Change to the shared repository directory.
Then start the acquisition pod.
```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ cd /vagrant/ae_gpcam
[vagrant@localhost ae_gpcam]$ sudo bash start_acquisition_pod.sh
```

Start the not_xpdan server (analogous to xpdan) from inside the VM:
```
[vagrant@localhost ae_gpcam]$ sudo bash launchers/start_not_xpdan_server.sh
++ pwd
+ podman run --pod acquisition --rm -ti -v /vagrant/ae_gpcam/bluesky_config/scripts:/app -w /app bluesky python3 not_xpdan_consumer.py
ANALYSIS CONSUMER IS LISTENING ON b'from-RE'
```

Open a new terminal and start the ROI reduction server from inside the VM:
```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ cd /vagrant/ae_gpcam
[vagrant@localhost ae_gpcam]$ sudo bash launchers/start_roi_reduction_server.sh
++ pwd
+ podman run --pod acquisition --rm -ti -v /vagrant/ae_gpcam/bluesky_config/scripts:/app -w /app bluesky python3 roi_reduction_consumer.py
ADAPTIVE CONSUMER LISTENING ON b'from-analysis'
```

Open a new terminal and start the adaptive gpcam server from inside the VM:
```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ cd /vagrant/ae_gpcam
[vagrant@localhost ae_gpcam]$ sudo bash launchers/start_adaptive_gpcam_server.sh
++ pwd
+ podman run --pod acquisition --rm -ti -v /vagrant/ae_gpcam/bluesky_config/scripts:/app -w /app bluesky python3 adaptive_gpcam_consumer.py
ADAPTIVE CONSUMER LISTENING ON b'from-analysis'
```

Open a new terminal, start bluesky from inside the VM, and run an adaptive plan:
```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ cd /vagrant/ae_gpcam
[vagrant@localhost ae_gpcam]$ sudo bash launch_bluesky_headless.sh
+ '[' '' '!=' '' ']'
+ imagename=bluesky
++ pwd
+ podman run --pod acquisition -ti --rm -v /vagrant/ae_gpcam:/app -w /app -v ./bluesky_config/ipython:/usr/local/share/ipython -v ./bluesky_config/databroker:/usr/local/share/intake -v ./bluesky_config/happi:/usr/local/share/happi -e EPICS_CA_ADDR_LIST=10.0.2.255 -e EPICS_CA_AUTO_ADDR_LIST=no bluesky ipython3 --ipython-dir=/usr/local/share/ipython
Python 3.8.6 (default, Sep 25 2020, 00:00:00)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.12.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from ophyd.sim import *

In [2]: from bluesky_adaptive.per_start import adaptive_plan

In [3]: RE(adaptive_plan([det], {motor: 0}, to_recommender=None, from_recommender=from_recommender))
```

### Run sbu_sim
Open a new terminal, start bluesky from inside the VM, and run a simulated data acquisition run:
```
host:ae-gpcam user$ vagrant ssh
[vagrant@localhost ~]$ cd /vagrant/ae_gpcam
[vagrant@localhost ae_gpcam]$ sudo bash launch_bluesky_headless.sh
+ '[' '' '!=' '' ']'
+ imagename=bluesky
++ pwd
+ podman run --pod acquisition -ti --rm -v /vagrant/ae_gpcam:/app -w /app -v ./bluesky_config/ipython:/usr/local/share/ipython -v ./bluesky_config/databroker:/usr/local/share/intake -v ./bluesky_config/happi:/usr/local/share/happi -v /vagrant/TiCu_export:/usr/local/share/TiCu_export -e EPICS_CA_ADDR_LIST=10.0.2.255 -e EPICS_CA_AUTO_ADDR_LIST=no bluesky ipython3 --ipython-dir=/usr/local/share/ipython
Python 3.9.0 (default, Oct  6 2020, 00:00:00)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.18.1 -- An enhanced Interactive Python. Type '?' for help.

In [1]: import sbu_sim

In [2]: sbu_sim.initialize(get_ipython().user_ns, 'temp', 'xpd_auto_202003_msgpack');
Exception reporting mode: Minimal
Created Entry named '098f1dbd-5f22-48c0-9abe-62a68413ee49'
...
Entry cache found BlueskyEventStream named 'primary'

In [3]: RE(bp.scan([full, ctrl], ctrl.Ti, 0, 100, 25))


Transient Scan ID: 1     Time: 2020-11-12 18:47:00
Persistent Unique Scan ID: '22985648-3502-4c18-b0c8-bd6c4425827a'


Transient Scan ID: 1     Time: 2020-11-12 18:47:00
Persistent Unique Scan ID: '22985648-3502-4c18-b0c8-bd6c4425827a'
New stream: 'primary'
/usr/local/lib/python3.9/site-packages/bluesky/callbacks/best_effort.py:273: UserWarning: Omitting full_I from plot because dtype is array
  warn("Omitting {} from plot because dtype is {}"
/usr/local/lib/python3.9/site-packages/bluesky/callbacks/core.py:332: UserWarning: The key full_I will be skipped because LiveTable does not know how to display the dtype array
  warnings.warn("The key {} will be skipped because LiveTable "
+-----------+------------+------------+------------------+------------+
|   seq_num |       time |    ctrl_Ti | ctrl_anneal_time |  ctrl_temp |
+-----------+------------+------------+------------------+------------+
New stream: 'primary'
+-----------+------------+------------+------------------+------------+
|   seq_num |       time |    ctrl_Ti | ctrl_anneal_time |  ctrl_temp |
+-----------+------------+------------+------------------+------------+
|         1 | 18:47:00.7 |      0.000 |               30 |        400 |
|         1 | 18:47:00.7 |      0.000 |               30 |        400 |
|         2 | 18:47:00.9 |      4.167 |               30 |        400 |
|         2 | 18:47:00.9 |      4.167 |               30 |        400 |
|         3 | 18:47:01.1 |      8.333 |               30 |        400 |
|         3 | 18:47:01.1 |      8.333 |               30 |        400 |
|         4 | 18:47:01.3 |     12.500 |               30 |        400 |
|         4 | 18:47:01.3 |     12.500 |               30 |        400 |
|         5 | 18:47:01.6 |     16.667 |               30 |        400 |
|         5 | 18:47:01.6 |     16.667 |               30 |        400 |
|         6 | 18:47:01.8 |     20.833 |               30 |        400 |
|         6 | 18:47:01.8 |     20.833 |               30 |        400 |
|         7 | 18:47:02.0 |     25.000 |               30 |        400 |
|         7 | 18:47:02.0 |     25.000 |               30 |        400 |
|         8 | 18:47:02.3 |     29.167 |               30 |        400 |
|         8 | 18:47:02.3 |     29.167 |               30 |        400 |
|         9 | 18:47:02.5 |     33.333 |               30 |        400 |
|         9 | 18:47:02.5 |     33.333 |               30 |        400 |
|        10 | 18:47:02.8 |     37.500 |               30 |        400 |
|        10 | 18:47:02.8 |     37.500 |               30 |        400 |
|        11 | 18:47:03.0 |     41.667 |               30 |        400 |
|        11 | 18:47:03.0 |     41.667 |               30 |        400 |
|        12 | 18:47:03.2 |     45.833 |               30 |        400 |
|        12 | 18:47:03.2 |     45.833 |               30 |        400 |
|        13 | 18:47:03.4 |     50.000 |               30 |        400 |
|        13 | 18:47:03.4 |     50.000 |               30 |        400 |
|        14 | 18:47:03.7 |     54.167 |               30 |        400 |
|        14 | 18:47:03.7 |     54.167 |               30 |        400 |
|        15 | 18:47:03.9 |     58.333 |               30 |        400 |
|        15 | 18:47:03.9 |     58.333 |               30 |        400 |
|        16 | 18:47:04.1 |     62.500 |               30 |        400 |
|        16 | 18:47:04.1 |     62.500 |               30 |        400 |
|        17 | 18:47:04.4 |     66.667 |               30 |        400 |
|        17 | 18:47:04.4 |     66.667 |               30 |        400 |
|        18 | 18:47:04.6 |     70.833 |               30 |        400 |
|        18 | 18:47:04.6 |     70.833 |               30 |        400 |
|        19 | 18:47:04.8 |     75.000 |               30 |        400 |
|        19 | 18:47:04.8 |     75.000 |               30 |        400 |
|        20 | 18:47:05.1 |     79.167 |               30 |        400 |
|        20 | 18:47:05.1 |     79.167 |               30 |        400 |
|        21 | 18:47:05.3 |     83.333 |               30 |        400 |
|        21 | 18:47:05.3 |     83.333 |               30 |        400 |
|        22 | 18:47:05.6 |     87.500 |               30 |        400 |
|        22 | 18:47:05.6 |     87.500 |               30 |        400 |
|        23 | 18:47:05.8 |     91.667 |               30 |        400 |
|        23 | 18:47:05.8 |     91.667 |               30 |        400 |
|        24 | 18:47:06.0 |     95.833 |               30 |        400 |
|        24 | 18:47:06.0 |     95.833 |               30 |        400 |
|        25 | 18:47:06.2 |    100.000 |               30 |        400 |
|        25 | 18:47:06.2 |    100.000 |               30 |        400 |
+-----------+------------+------------+------------------+------------+
generator scan ['22985648'] (scan num: 1)

Created Entry named '22985648-3502-4c18-b0c8-bd6c4425827a'
+-----------+------------+------------+------------------+------------+
generator scan ['22985648'] (scan num: 1)

Out[3]: ('22985648-3502-4c18-b0c8-bd6c4425827a',)

In [4]:

```
