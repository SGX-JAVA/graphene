## Specific Developer Build Environments
### Ubuntu
#### Ubuntu 9.04
After installing Ubuntu 9.04 you need to install several build dependencies. The simplest way to do it is to execute the following commands:
```
sudo aptitude build-dep openjdk-6

sudo aptitude install openjdk-6-jdk 
```
In addition, it's necessary to set a few environment variables for the build: 
```
export LANG=C ALT_BOOTDIR=/usr/lib/jvm/java-6-openjdk
```

## Source Directory Structure
The source code for the OpenJDK is delivered in a set of directories: `hotspot`, `langtools`, `corba`, `jaxws`, `jaxp`, and `jdk`. The `hotspot` directory contains the source code and make files for building the OpenJDK Hotspot Virtual Machine. The `jdk` directory contains the source code and make files for building the OpenJDK runtime libraries and misc files. The top level `Makefile` is used to build the entire OpenJDK.

## Build Information
### Build Dependencies
#### Bootstrap JDK
All OpenJDK builds require access to the previously released JDK 6, this is often called a bootstrap JDK. The JDK 6 binaries can be downloaded from Sun's JDK 6 download site. For build performance reasons is very important that this bootstrap JDK be made available on the local disk of the machine doing the build. You should always set `ALT_BOOTDIR` to point to the location of the bootstrap JDK installation, this is the directory pathname that contains a bin, lib, and include It's also a good idea to also place its bin directory in the PATH environment variable, although it's not required.
