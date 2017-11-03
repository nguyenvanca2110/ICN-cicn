# ICN-cicn
ICN project by Cisco

===============CCIN Simple VMs Demo Guide===============

======Requirement:
- Host machine need to support SSE4.2
  cat /proc/cpuinfo | grep "sse4_2 "
- Virtualbox support SSE4.3, to0 (>=virtualbox 5.x.x)
- vagrant 2.x.x install by *deb (https://releases.hashicorp.com/vagrant/2.0.0/vagrant_2.0.0_x86_64.deb?_ga=2.193204845.1711068333.1509552126-1045245658.1506328806)


======Installation
0. #Provision script to update ubuntu and install required libraries
  export DEBIAN_FRONTEND=noninteractive
   
  sudo apt-get update
  # For faster download
  (Option 1)
	## sudo sed -i 's/kr.archive.ubuntu.com/ftp.daumkakao.com/g' /etc/apt/sources.list 
	## sudo sed -i 's/kr.security.ubuntu.com/ftp.daumkakao.com/g' /etc/apt/sources.list
  (Option 2)
	## sudo sed -i 's/archive.ubuntu.com/ftp.daumkakao.com/g' /etc/apt/sources.list 
	## sudo sed -i 's/security.ubuntu.com/ftp.daumkakao.com/g' /etc/apt/sources.list
  # Base on
	# Get:1 http://security.ubuntu.com/ubuntu xenial-security InRelease [102 kB]
	# Hit:2 http://archive.ubuntu.com/ubuntu xenial InRelease
	# Get:3 http://archive.ubuntu.com/ubuntu xenial-updates InRelease [102 kB]
	# Get:4 http://security.ubuntu.com/ubuntu xenial-security/main Sources [99.1 kB]
	# Get:5 http://security.ubuntu.com/ubuntu xenial-security/restricted Sources [2600 B]


  sudo apt-get -y --allow-unauthenticated upgrade
  sudo apt-get install -y libssl-dev openssl flex bison expat \
                          libexpat1-dev libevent-dev g++ libpcre3-dev \
                          python-dev uncrustify doxygen git libpcap-dev \
                          cmake libboost-all-dev
  sudo apt-get install -y software-properties-common
  
  
 =============CICN2 - Forwarder================================
0. #Check hardware support VM
# VPP request: SSE4.2 
cat /proc/cpuinfo
"must be have:  sse4_2 "
# Active sse4_2 
VBoxManage setextradata "VM-name" VBoxInternal/CPUM/SSE4.2 1

# Check virtualbox version (>=5 to support SSE4.2)
virtualbox -h
Box Manager 4.3.36_Ubuntu
 
1. # Install VPP

#VPP 17.07 (Install by compile the source code)
echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.stable.1707.ubuntu.xenial.main/ ./" | sudo tee -a /etc/apt/sources.list.d/99fd.io.list

# (When install by package:  sudo apt-get install cicn-plugin)
# Add repositories file and check it in "/etc/apt/sources.list.d/99fd.io.list" 
echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.master.ubuntu.$(lsb_release -sc).main/ ./" \
         | sudo tee -a /etc/apt/sources.list.d/99fd.io.list

#Set 17.07 as the "priority version" for CICN: edit the "/etc/apt/preferences.d/99fd.io.pref" file with your favorite editor and write the following:

sudo vi  /etc/apt/preferences.d/99fd.io.pref

Package: vpp*
Pin: version 17.07.01-release
Pin-Priority: 1001

Package: vpp-dpdk*
Pin: version 17.05-vpp6
Pin-Priority: 1001

#Then update your repositories:
==============================

#VPP 17.04
		 
echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.stable.1704.ubuntu.xenial.main/ ./" | sudo tee -a /etc/apt/sources.list.d/99fd.io.list

echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.master.ubuntu.$(lsb_release -sc).main/ ./" \
         | sudo tee -a /etc/apt/sources.list.d/99fd.io.list


sudo vi  /etc/apt/preferences.d/99fd.io.pref

Package: vpp*
Pin: version 17.04.2-release
Pin-Priority: 1001

Package: vpp-dpdk*
Pin: version 17.02-vpp2
Pin-Priority: 1001

Package: cicn-plugin
Pin: version 17.04-10~gd7ad2a7~b197
Pin-Priority: 1001
====================================================

#Then update your repositories:

export DEBIAN_FRONTEND=noninteractive
sudo apt-get update
sudo apt-get install -y --allow-unauthenticated vpp-dbg vpp-dev vpp-lib vpp vpp-plugins vpp-dpdk-dev vpp-dpdk-dkms


# Check interface
$ lspci
00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:09.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)

# Configuration script for vpp
sudo vi /etc/vpp/startup.conf
==
unix {
  nodaemon
  log /tmp/vpp.log
  full-coredump
}

api-trace {
  on
}

api-segment {
  gid vpp
}

dpdk {
  socket-mem 1024
  dev 0000:00:08.0
  dev 0000:00:09.0
}
EOF
==
2. # Get and compile code

#Install from the packages
 sudo apt-get install cicn-plugin

 #Install by compile from source code
git clone -b cicn-plugin/master  https://gerrit.fd.io/r/cicn cicn-plugin
cd $HOME/cicn-plugin/cicn-plugin
mkdir build
cd build
cmake -DCICN_INSTALL_PREFIX=/usr/lib -DNO_UNIT_TEST=TRUE ..
make
sudo make install

3. # Configure Cicn-plugin

# Setup the host for VPP 
#Check VPP status and start VPP as a process in both 16.04 and 14.04
$ sudo vpp -c /etc/vpp/startup.conf

# If the DPDK interface we want to assign to VPP is up, we must bring it down: 
cd $HOME
sudo ifconfig enp0s8 down
sudo ifconfig enp0s9 down

# VPP requires the uio and igb_uio modules to be loaded in the kernel: 

sudo modprobe uio
sudo modprobe igb_uio
sudo sysctl vm.nr_hugepages=1024

# Stop VPP
sudo systemctl stop vpp

#
sudo dpdk-devbind -b igb_uio 00:08.0
sudo dpdk-devbind -b igb_uio 00:09.0

# Start VPP as a service in Ubuntu 16.04
sudo systemctl start vpp

# Configure VPP interfaces
# Set an IP address on the DPDK interface
sudo vppctl set int ip address GigabitEthernet0/8/0 10.0.1.22/24
sudo vppctl set int state GigabitEthernet0/8/0 up

#Set an IP address on the DPDK interface
sudo vppctl set int ip address GigabitEthernet0/9/0 10.0.3.22/24
sudo vppctl set int state GigabitEthernet0/9/0 up

#Configure and start CICN
sudo vppctl cicn enable
# Add face ID from Comsumer to Forwarder
sudo vppctl cicn cfg face add local 10.0.1.22:33302 remote 10.0.1.21:33302

# Add face ID from Forwarder to Croducer
sudo vppctl cicn cfg face add local 10.0.3.22:33302 remote 10.0.3.23:33302
sudo vppctl cicn cfg fib add prefix /helloworld face 2
=========================================================================
 
============CICN1-Consummer
  
1. #Install metis forwarder

#Git code to build

git clone -b cframework/master   https://gerrit.fd.io/r/cicn cframework
git clone -b ccnxlibs/master https://gerrit.fd.io/r/cicn ccnxlibs
git clone -b libicnet/master     https://gerrit.fd.io/r/cicn libicnet
git clone -b sb-forwarder/master https://gerrit.fd.io/r/cicn sb-forwarder

cd cframework/longbow
mkdir build 
cd build
cmake ..
make
make test #Check test
sudo make install
cd $HOME

cd $HOME/cframework/libparc
#{
#The following tests FAILED:
#         42 - test_parc_Network (Failed)
#         73 - test_parc_ScheduledThreadPool (SEGFAULT)
#         76 - test_parc_ThreadPool (SEGFAULT)
#Errors while running CTest
#Makefile:127: recipe for target 'test' failed
#make: *** [test] Error}

mkdir build
cd build
cmake ..
make
make test #Check test
sudo make install
cd $HOME


cd $HOME/ccnxlibs/libccnx-common
mkdir build
cd build
cmake ..
make
sudo make install
cd $HOME

# Compile libccnx-transport-rta
#{
 # If libccnx-transport-rta error, handle following:
 echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.master.ubuntu.$(lsb_release -sc).main/ ./"           | sudo tee -a /etc/apt/sources.list.d/99fd.io.list
 sudo apt-get update
 sudo apt-get install libccnx-transport-rta
}

cd $HOME/ccnxlibs/libccnx-transport-rta
mkdir build
cd build
cmake ..
make
make test #Check test
sudo make install
cd $HOME

cd $HOME/ccnxlibs/libccnx-portal
mkdir build
cd build
cmake ..
make
make test #Check test
sudo make install
cd $HOME
# Compile libicnet

cd $HOME/libicnet
mkdir build
cd build
cmake ..
make
sudo make install
cd $HOME

# Compile sb-forwader
cd $HOME/sb-forwarder/metis
mkdir build
cd build
cmake ..
make
make test #Check test
sudo make install
cd $HOME

mkdir -p ~/.ccnx
parc-publickey --create ~/.ccnx/.ccnx_keystore.p12 1234 username 2048 1000

2. Configure consumer
#
$configure_metis_consumer = <<SCRIPT
# Clean from previous setting
  rm ~/metis.cfg

# Kill metis if it is already running
pgrep "metis_daemon"
killall metis_daemon

echo "add listener tcp local0 127.0.0.1 9695" > ~/metis.cfg
echo "add listener udp local1 127.0.0.1 9695" >> ~/metis.cfg
echo "add listener udp remote0 10.0.1.21 33302" >> ~/metis.cfg
echo "add connection udp conn0 10.0.1.22 33302 10.0.1.21 33302" >> ~/metis.cfg
echo "add route conn0 ccnx:/cicn 1" >> ~/metis.cfg
metis_daemon --config metis.cfg &


============CICN3 - Producer

1.# Same with consumer 
2.# Configure producer

rm ~/metis.cfg

# Kill metis if it is already running
pgrep "metis_daemon" 
killall metis_daemon


echo "add listener tcp local0 127.0.0.1 9695" > ~/metis.cfg
echo "add listener udp local1 127.0.0.1 9695" >> ~/metis.cfg
echo "add listener udp remote0 10.0.3.23 33302" >> ~/metis.cfg
echo "add connection udp conn0 10.0.3.22 33302 10.0.3.23 33302" >> ~/metis.cfg
metis_daemon --config metis.cfg &
=========================OPERATION============================
1. Active forwarder
# Setup the host for VPP 

# If the DPDK interface we want to assign to VPP is up, we must bring it down: 
cd $HOME
sudo ifconfig enp0s8 down
sudo ifconfig enp0s9 down

# VPP requires the uio and igb_uio modules to be loaded in the kernel: 

sudo modprobe uio
sudo modprobe igb_uio
sudo sysctl vm.nr_hugepages=1024

# Stop VPP
sudo systemctl stop vpp

sudo dpdk-devbind -b igb_uio 00:08.0
sudo dpdk-devbind -b igb_uio 00:09.0

# Start VPP as a service in Ubuntu 16.04
sudo systemctl start vpp

# Check interface
$ lspci
00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)
00:09.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)

# Configure VPP interfaces
# Set an IP address on the DPDK interface
sudo vppctl set int ip address GigabitEthernet0/8/0 10.0.1.22/24
sudo vppctl set int state GigabitEthernet0/8/0 up

#Set an IP address on the DPDK interface
sudo vppctl set int ip address GigabitEthernet0/9/0 10.0.3.22/24
sudo vppctl set int state GigabitEthernet0/9/0 up

#Configure and start CICN
sudo vppctl cicn enable
sudo vppctl cicn cfg face add local 10.0.1.22:33302 remote 10.0.1.21:33302
sudo vppctl cicn cfg face add local 10.0.3.22:33302 remote 10.0.3.23:33302
sudo vppctl cicn cfg fib add prefix /cicn face 2


2. Produce create a content:
 - restart mestis 
pgrep "metis_daemon" 
killall metis_daemon
 - make new content
producer-test ccnx:/helloworld

3. Comsumer request content

 - restart mestis 
pgrep "metis_daemon" 
killall metis_daemon
 - request new content
comsumer-test ccnx:/helloworld

===============================ERROR=====================================


1. If libccnx-transport-rta error, handle following:
#Kill "metis" process first
 pgrep "metis_daemon"
 killall metis_daemon
 
 echo "deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.master.ubuntu.$(lsb_release -sc).main/ ./"           | sudo tee -a /etc/apt/sources.list.d/99fd.io.list
 sudo apt-get update
 sudo apt-get install libccnx-transport-rta
}

2. CICN statistics
sudo vppctl cicn show

3. Show DPDK interface list
lspci

4. VPP version

sudo apt-get install -y --allow-unauthenticated vpp-dbg vpp-dev vpp-dpdk-dev vpp-lib vpp-dpdk-dkms vpp vpp-plugins

Reading package lists... Done
Building dependency tree
Reading state information... Done
vpp-dpdk-dev is already the newest version (17.05-vpp6).
vpp-dpdk-dkms is already the newest version (17.05-vpp6).
vpp is already the newest version (17.07.01-release).
vpp-dbg is already the newest version (17.07.01-release).
vpp-lib is already the newest version (17.07.01-release).
vpp-dev is already the newest version (17.07.01-release).
vpp-plugins is already the newest version (17.07.01-release).
0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.

5. VPP interface check (https://wiki.fd.io/view/VPP/Build,_install,_and_test_images)
sudo vppctl show int
sudo vppctl show interface
# Check status
sudo service vpp status
ubuntu@cicn2-v-dcni7:~$ sudo service vpp status
● vpp.service - vector packet processing engine
   Loaded: loaded (/lib/systemd/system/vpp.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2017-10-29 13:21:06 UTC; 2 days ago
  Process: 30193 ExecStopPost=/bin/rm -f /dev/shm/db /dev/shm/global_vm /dev/shm/vpe-api (code=exited, status=0/SUCCESS)
  Process: 30241 ExecStartPre=/sbin/modprobe uio_pci_generic (code=exited, status=1/FAILURE)
  Process: 30238 ExecStartPre=/bin/rm -f /dev/shm/db /dev/shm/global_vm /dev/shm/vpe-api (code=exited, status=0/SUCCESS)
 Main PID: 30244 (vpp_main)
    Tasks: 3
   Memory: 96.6M
      CPU: 2d 12h 30min 47.356s
   CGroup: /system.slice/vpp.service
           └─30244 /usr/bin/vpp -c /etc/vpp/startup.conf

=======>>>>>>>>>>>>> OK


. /opt/stack/devstack/openrc admin admin



openstack port delete cicn_net0 cicn_net1
openstack port create --network public \
--disable-port-security \
--fixed-ip subnet=net0_sub,ip-address=10.0.1.220 \
cicn_net0

openstack port create --network public \
--disable-port-security \
--fixed-ip subnet=net1_sub,ip-address=10.0.3.220 \
cicn_net1

openstack server delete cicn-3port
openstack server create --flavor m1.medium \
--image cicn-3port \
--nic net-id=$(openstack network list | awk '/ public / {print $2}'),v4-fixed-ip=192.168.10.220 \
cicn-3port

openstack server add port cicn-3port $(openstack port list | awk '/ cicn_net0 / {print $2}')
openstack server add port cicn-3port $(openstack port list | awk '/ cicn_net1 / {print $2}')
openstack console url show --novnc cicn-3port
