# Scripts for Cluster
## Setup folders
```bash
mkdir -p /home/cc/software
mkdir -p /home/cc/main/install
mkdir -p /home/cc/main/program
```

## Setup nfs
```bash
mkdir -p /home/cc/nfs
sudo  apt-get install nfs-kernel-server
cp /etc/exports /home/cc/exports
echo -e "\n/home/cc/main       10.52.0.0/24(rw,fsid=0,insecure,no_subtree_check,async)\n" >> /home/cc/exports
cat /home/cc/exports
sudo mv /home/cc/exports /etc/exports
```

### Start nfs script
```bash
#!/bin/bash
NODES=$(cat ./nodes)
for server_node in $NODES
do    
ssh -t $server_node << EOF
sudo umount --force -l /home/cc/nfs
sudo mount ares:/ /home/cc/nfs
EOF
done
```


## .bashrc
```bash
export PATH=/home/cc/nfs/install/bin:/home/cc/nfs/install/sbin:$PATH
export LD_LIBRARY_PATH=/home/cc/nfs/install/lib
export CPPFLAGS="-I/home/cc/nfs/install/include"
export CFLAGS="-I/home/cc/nfs/install/include"
export CXXFLAGS="-I/home/cc/nfs/install/include"
export LDFLAGS="-L/home/cc/nfs/install/lib"
export PVFS2TAB_FILE=/home/cc/nfs/pvfs2tab
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
sudo ldconfig
sudo updatedb

```

## Orangefs
### Install with tcp
```bash
cd /home/cc/software
sudo apt-get install linux-headers-`uname -r` db5.3
wget https://s3.amazonaws.com/download.orangefs.org/current/source/orangefs-2.9.7.tar.gz
tar -xvf orangefs-2.9.7.tar.gz
cd orangefs-2.9.7
./configure --prefix=/home/cc/nfs/install --enable-fast --enable-shared --with-kernel=/lib/modules/`uname -r`/build
sed -i -e 's/attr\/xattr.h/linux\/xattr.h/g' src/apps/user/ofs_setdirhint.c
make -j96
make install
make kmod
make kmod_prefix=/home/cc/nfs/install kmod_install
sudo updatedb
```

### Install with ib
```bash
cd /home/cc/software
sudo apt-get install linux-headers-`uname -r` db5.3
wget https://s3.amazonaws.com/download.orangefs.org/current/source/orangefs-2.9.7.tar.gz
tar -xvf orangefs-2.9.7.tar.gz
cd orangefs-2.9.7
./configure --enable-shared --prefix=/home/cc/nfs/install --with-kernel=/lib/modules/`uname -r`/build --with-openib=/usr/lib --without-bmi-tcp --with-openib-libs=/usr/lib
sed -i -e 's/attr\/xattr.h/linux\/xattr.h/g' src/apps/user/ofs_setdirhint.c
make -j96
make install
make kmod
make kmod_prefix=/home/cc/nfs/install kmod_install
sudo updatedb
```

### Start pfs server script
```bash
#!/bin/bash
SERVER_NODES=$(cat ./servers)
INSTALL_DIR=/home/cc/nfs/install
CONF_DIR=/home/cc/nfs/program/ares/scripts/conf
for server_node in $SERVER_NODES
do
ssh $server_node "sudo rm -rf /home/cc/pfs/"
ssh $server_node "mkdir -p /home/cc/pfs/"
ssh $server_node "${INSTALL_DIR}/sbin/pvfs2-server -f -a ${server_node} ${CONF_DIR}/pfs.conf"
echo "$server_node is starting"
ssh $server_node "${INSTALL_DIR}/sbin/pvfs2-server -a ${server_node} ${CONF_DIR}/pfs.conf"
sleep 2
ssh $server_node "ps -aef | grep pvfs2"
echo "$server_node  has started"
done
```

### Stop pfs server script
```bash
#!/bin/bash
SERVER_NODES=$(cat servers)
for server_node in $SERVER_NODES
do
        echo "$server_node is stopping"
        ssh $server_node "ps -aef | grep pvfs | awk '{print $2}' | xargs kill -9"
	    ssh $server_node "killall pvfs2-server"
        ssh $server_node "ps -aef | grep pvfs"
        echo "$server_node has been stopped"
done
```

### Setup pfs client script
```bash
#!/bin/bash
INSTALL_DIR=/home/cc/nfs/install
SCRIPTS_DIR=/home/cc/nfs/program/ares/scripts
umount -l --force /mnt/pfs 
ps -aef | grep pvfs | awk '{print $2}' | xargs kill -9
killall pvfs2-client
rmmod ${INSTALL_DIR}/lib/modules/`uname -r`/kernel/fs/pvfs2/pvfs2.ko 
rm -rf /mnt/pfs
mkdir /mnt/pfs
chown cc:cc -R /mnt/pfs
echo "loading kernel module"
insmod ${INSTALL_DIR}/lib/modules/`uname -r`/kernel/fs/pvfs2/pvfs2.ko
echo "loading pvfs2-client"
cp /home/cc/nfs/pvfs2tab /etc/pvfs2tab
${INSTALL_DIR}/sbin/pvfs2-client -p ${INSTALL_DIR}/sbin/pvfs2-client-core
echo "mounting the pvfs2"
${INSTALL_DIR}/bin/pvfs2-ls /mnt/pfs
mount -t pvfs2 tcp://ares-17:3334/orangefs /mnt/pfs
mount | grep pvfs2
```

### Start pfs client script
```bash
#!/bin/bash
CLIENT_NODES=$(cat ./clients)
INSTALL_DIR=/home/cc/nfs/install
SCRIPTS_DIR=/home/cc/nfs/program/ares/scripts
for node in $CLIENT_NODES
do
	echo "$node is starting"
ssh $node << EOF1
sudo cp /home/cc/.bashrc /root/.bashrc
sudo su << EOF
umount -l --force /mnt/pfs 
ps -aef | grep pvfs | awk '{print $2}' | xargs kill -9
killall pvfs2-client
rmmod ${INSTALL_DIR}/lib/modules/`uname -r`/kernel/fs/pvfs2/pvfs2.ko 
rm -rf /mnt/pfs
mkdir /mnt/pfs
chown cc:cc -R /mnt/pfs
echo "loading kernel module"
insmod ${INSTALL_DIR}/lib/modules/`uname -r`/kernel/fs/pvfs2/pvfs2.ko
echo "loading pvfs2-client"
cp /home/cc/nfs/pvfs2tab /etc/pvfs2tab
LD_PRELOAD=/home/cc/nfs/install/lib/libpvfs2.so.2.9.7 ${INSTALL_DIR}/sbin/pvfs2-client -p ${INSTALL_DIR}/sbin/pvfs2-client-core
echo "mounting the pvfs2"
LD_PRELOAD=/home/cc/nfs/install/lib/libpvfs2.so.2.9.7 ${INSTALL_DIR}/bin/pvfs2-ls /mnt/pfs
mount -t pvfs2 tcp://ares-17:3334/orangefs /mnt/pfs
mount | grep pvfs2
EOF
EOF1
	echo "client $node is running"
	read a
done
```

### Unsetup pfs client script
```bash
#!/bin/bash
INSTALL_DIR=/home/cc/nfs/install
SCRIPTS_DIR=/home/cc/nfs/program/ares/scripts
umount -l --force /mnt/pfs 
ps -aef | grep pvfs | awk '{print $2}' | xargs kill -9
killall pvfs2-client
rmmod ${INSTALL_DIR}/lib/modules/`uname -r`/kernel/fs/pvfs2/pvfs2.ko 
rm -rf /mnt/pfs
mount | grep pvfs2
```

### Stop pfs client script
```bash
#!/bin/bash
INSTALL_DIR=/home/cc/nfs/install
SCRIPTS_DIR=/home/cc/nfs/program/ares/scripts
CLIENT_NODES=$(cat ./clients)
for node in $CLIENT_NODES
do
	echo "$node is starting"
    ssh $node "sudo sh ${SCRIPTS_DIR}/unsetup_clients.sh"
	echo "$node is running"
done
```


## MPI
```bash
cd /home/cc/software
wget http://www.mpich.org/static/downloads/3.2.1/mpich-3.2.1.tar.gz
tar -xvf mpich-3.2.1.tar.gz
cd mpich-3.2.1
./configure --prefix=/home/cc/nfs/install --enable-shared --enable-fast=O3 --with-pvfs2=/home/cc/nfs/install
make -j48
make install
sudo updatedb
```

## MPI with ib
```bash
cd /home/cc/software
wget http://www.mpich.org/static/downloads/3.2.1/mpich-3.2.1.tar.gz
tar -xvf mpich-3.2.1.tar.gz
cd mpich-3.2.1
./configure --prefix=/home/cc/nfs/install --enable-fast=03 --enable-shared --enable-romio --enable-threads --disable-fortran --disable-fc --with-pvfs2=/home/cc/nfs/install --with-mxm=/opt/mellanox/mxm/lib/ --with-mxm-include=/opt/mellanox/mxm/include 
make -j48
make install
sudo updatedb
```


## hdf5
```bash
cd /home/cc/software
git clone --single-branch -b 1.10/master https://bitbucket.hdfgroup.org/scm/hdffv/hdf5.git
cd hdf5
./configure --prefix=/home/cc/nfs/install --enable-shared --enable-parallel
make -j96
make install
sudo updatedb
```

## H5Part
```bash
cd /home/cc/software
wget https://code.lbl.gov/frs/download.php/file/387/H5Part-1.6.6.tar.gz
tar -xvf H5Part-1.6.6.tar.gz
cd H5Part-1.6.6
./configure --prefix=/home/cc/nfs/install --enable-shared --enable-parallel --with-hdf5=/home/cc/nfs/install
sed -i -e 's/H5Pset_fapl_mpiposix/H5Pset_fapl_mpio/g' ./src/H5Part.c
make -j96
make install
sudo updatedb
```

## Netcdf
```bash
cd /home/cc/software
wget https://www.unidata.ucar.edu/downloads/netcdf/ftp/netcdf-4.6.1.tar.gz
tar -xvf netcdf-4.6.1.tar.gz
cd netcdf-4.6.1
./configure --prefix=/home/cc/nfs/install --enable-shared --enable-parallel --with-hdf5=/home/cc/nfs/install
make -j96
make install
sudo updatedb
```

## Boost
```bash
sudo apt-get install libboost-all-dev
sudo updatedb
```
## Cmake
```bash
sudo apt install cmake
sudo updatedb
```

## Java
```bash
sudo apt install default-jdk
sudo updatedb
```

## Avro
```bash
cd /home/cc/software
wget http://mirror.cogentco.com/pub/apache/avro/stable/cpp/avro-cpp-1.8.2.tar.gz
tar -xvf avro-cpp-1.8.2.tar.gz
cd avro-cpp-1.8.2
sudo ./build.sh install
sudo updatedb
```

## Parquet
```bash
cd /home/cc/software
wget http://archive.apache.org/dist/arrow/arrow-0.11.0/apache-arrow-0.11.0.tar.gz
tar -xvf apache-arrow-0.11.0.tar.gz
cd apache-arrow-0.11.0/cpp
mkdir build
cd build
cmake ../ -DARROW_PARQUET=ON
make -j48
sudo make install
sudo updatedb
```

## Hadoop
```bash
cd /home/cc/software
wget http://mirror.cc.columbia.edu/pub/software/apache/hadoop/common/hadoop-2.8.5/hadoop-2.8.5.tar.gz
tar -xvf hadoop-2.8.5.tar.gz.1
cp -r hadoop-2.8.5/* /home/cc/main/install/
sudo updatedb
```

