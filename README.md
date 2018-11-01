# Scripts for Cluster
## setup folders
```bash
mkdir -p /home/cc/software
mkdir -p /home/cc/main/install
mkdir -p /home/cc/main/program
```

## setup nfs
```bash
mkdir -p /home/cc/nfs
sudo  apt-get install nfs-kernel-server
cp /etc/exports /home/cc/exports
echo -e "\n/home/cc/main       10.52.0.0/24(rw,fsid=0,insecure,no_subtree_check,async)\n" >> /home/cc/exports
cat /home/cc/exports
sudo mv /home/cc/exports /etc/exports
```


## bashrc
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

## orange fs with tcp
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

## orange fs with ib
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


## mpi
```bash
cd /home/cc/software
wget http://www.mpich.org/static/downloads/3.2.1/mpich-3.2.1.tar.gz
tar -xvf mpich-3.2.1.tar.gz
cd mpich-3.2.1
./configure --prefix=/home/cc/nfs/install --enable-shared --enable-fast=O3 
make -j48
make install
sudo updatedb
```

## mpi with ib
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

## netcdf
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

## boost
```bash
sudo apt-get install libboost-all-dev
sudo updatedb
```
## cmake
```bash
sudo apt install cmake
sudo updatedb
```

## cmake
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
