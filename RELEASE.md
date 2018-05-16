# Releasing a New Version

## Version Notes

### 3.2.0

#### Linux/Arm for Raspberry Pi

* Install packages:
```
sudo apt-get update && sudo apt-get install oracle-java8-jdk cmake ant
sudo apt-get install build-essential cmake pkg-config libpng12-0 libpng12-dev libpng++-dev libpng3 libpnglite-dev zlib1g-dbg zlib1g zlib1g-dev pngtools  libtiff4 libtiffxx0c2 libtiff-tools libjpeg8 libjpeg8-dev libjpeg8-dbg libjpeg-progs libavcodec-dev   libavformat-dev libgstreamer0.10-0-dbg libgstreamer0.10-0 libgstreamer0.10-dev  libunicap2 libunicap2-dev libdc1394-22-dev libdc1394-22 libdc1394-utils swig libv4l-0 libv4l-dev
```
* Change .bashrc to set JAVA_HOME, ANT_HOME and paths:

```
export ANT_HOME=/usr/share/ant/
export PATH=${PATH}:${ANT_HOME}/bin
export JAVA_HOME=/usr/lib/jvm/jdk-8-oracle-arm32-vfp-hflt/
export PATH=$PATH:$JAVA_HOME/bin
```

* Download opencv:

```
wget https://github.com/opencv/opencv/archive/3.2.0.zip
mv 3.2.0 opencv.zip
unzip opencv.zip 
cd opencv-3.2.0/
```

* Finally build using cmake:
```
mkdir build
cd build
cmake -D CMAKE_BUILD_TYPE=RELEASE -D WITH_OPENCL=OFF -D BUILD_PERF_TESTS=OFF -D BUILD_SHARED_LIBS=OFF -D JAVA_INCLUDE_PATH=$JAVA_HOME/include -D JAVA_AWT_LIBRARY=$JAVA_HOME/jre/lib/amd64/libawt.so -D JAVA_JVM_LIBRARY=$JAVA_HOME/jre/lib/arm/server/libjvm.so -D CMAKE_INSTALL_PREFIX=/usr/local ..
make
make install
```

* libopencv_java320.so can now be found in build/libs 


#### Linux
* Built on Ubuntu 16.04.1
* Was having problems with HGFS, solved with: http://askubuntu.com/questions/591664/files-missing-in-mnt-hgfs-on-ubuntu-vm
* sudo apt install default-jdk ant maven git cmake -y
* git clone https://github.com/rasa/vmware-tools-patches.git
* cd vmware-tools-patches
* ./patched-open-vm-tools.sh
* Getting seemingly random missing separator errors when making. Seems random because just running make again sometimes goes further. Was able to finally build by using `until make; do echo; done` which is absurd but it worked, so it's not absurd. Could be that this is due to unzipping on OS X, but the line endings should be the same.
* On 32 bit: `jason@ubuntu:/usr/lib/jvm/default-java/include$ sudo ln -s linux/jni_md.h jni_md.h` Seems that cmake didn't detect the include/linux directory for some reason.
* OpenCV built in tests would not pass in 32 bit. Failures seemed centered around the calib3d module. Releasing anyway as our library tests were passing, so perhaps this will still be useful to some people.

#### Ubuntu 14.04.5 Attempt
* Due to https://github.com/openpnp/opencv/issues/10, attempting to rebuild on Ubuntu 14.04.5 for better libstdc++ compatibility.
* HGFS seems to be just completely broken, so instead of switching back and forth between the VM and the host we're goign to build everything for the Linux parts in the VM and check it into Github and then finish the process in OS X.
* With a fresh 14.04.5 install:
```
sudo apt update -y
sudo apt full-upgrade -y
sudo reboot
sudo apt install default-jdk ant maven git cmake g++ -y
git clone https://github.com/openpnp/opencv.git
cd opencv; mkdir opencv; cd opencv
wget https://github.com/opencv/opencv/archive/3.2.0.zip
unzip opencv-3.2.0.zip
cd ..
./create-targets.sh 3.2.0
cd opencv/opencv-3.2.0/target/linux/x86_64
cmake -DBUILD_SHARED_LIBS=OFF ../../.. > cmake.log
make -j8
```



### 3.0.0

* OS X Tests are currently failing due to https://github.com/Itseez/opencv/issues/5030
* Linux 32 bit build failed due to IPP - seems to have been corrected after build 64 bit version.
* Linux 32 bit tests failed similar to OS X.
* Linux 64 bit tests crash the VM.
* Windows 64 bit tests fail in multiply.

## Prepare

1. Download OpenCV Linux/OSX source and Windows binary release to opencv/. Should be a ZIP and a EXE.
2. Unzip source ZIP to create opencv/opencv-VERSION.
3. Run ./create-targets.sh VERSION (e.g. ./create-targets 3.0.0) to create all of the target directories under opencv/opencv-VERSION/target.
4. Update pom.xml, OpenCV.java and README.md with new versions.

## Build OSX and Java Targets

1. Make sure Java is 1.7: `java -version`, `export JAVA_HOME=$(/usr/libexec/java_home -v1.7)` 
2. `cd opencv/opencv-VERSION/target/osx/x86_64`
3. `cmake -DBUILD_SHARED_LIBS=OFF -DEIGEN_INCLUDE_PATH=/usr/local/include/eigen3 ../../.. > cmake.log`
4. `make -j8`
5. `./copy-resources VERSION`. It won't finish, but it should get far enough.
6. `mvn clean test`.

## Build Linux Targets

### 32 bit

1. Start the 32 bit Linux VM. It needs default-jdk, maven and ant installed.
2. `cd /mnt/hgfs/opencv/opencv/opencv-VERSION/target/linux/x86_32`
3. `cmake -DBUILD_SHARED_LIBS=OFF ../../.. > cmake.log`
4. `make -j8`
5. In the host OS, `./copy-resources VERSION`
6. Back in the VM, `cd /mnt/hgfs/opencv`
7. `mvn clean test`

### 64 bit
 
1. Start the 64 bit Linux VM. It needs default-jdk, maven and ant installed.
2. `cd /mnt/hgfs/opencv/opencv/opencv-VERSION/target/linux/x86_64`
3. `cmake -DBUILD_SHARED_LIBS=OFF ../../.. > cmake.log`
4. `make -j8`
5. In the host OS, `./copy-resources VERSION`
6. Back in the VM, `cd /mnt/hgfs/opencv`
7. `mvn clean test`

## Unpack Windows Binaries

1. In a Windows VM, double click the Windows EXE downloaded previously. It will
ask where to extract. Choose opencv/opencv-VERSION/target/windows.
2. In the host OS, `./copy-resources VERSION`.
3. Back in the VM, `mvn clean test`

## Finish

1. Commit the updates to develop.
2. Merge develop into master.
3. Push to Sonatype:
	1. Read http://central.sonatype.org/pages/apache-maven.html
	2. `mvn clean deploy -P release-sign-artifacts`
	3. Check it's good at https://oss.sonatype.org/
	4. Release with `mvn nexus-staging:release`
	5. Or reset with `mvn nexus-staging:drop` 
4. `git tag vVERSION-RELEASE` (e.g. `git tag v3.0.0-1`)

