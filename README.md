# dropbear
Dropbear files compiled for the Juno-r2 board and to be used with OP-TEE OS.

These binaries were compiled with the following steps: 

  1. Download `buildroot-2016.02` to `Desktop`

	2. `cd Desktop/buildroot-2016.02`

	3. `make menuconfig`
	
	4. `Target options -> Target Architecture: ARM (little endian)`
	
	5. `Target options -> Target Architecture Variant: arm926t`
	
	6. `Toolchain -> C library: uClibc`
	
	7. `Compile and install uClibc utilities`
	
	8. Save configuration
	
	9. `make`
	
	10. Download and extract `dropbear-2016.73.tar.bz2` to `Desktop`
	
	11. `cd Desktop/dropbear-2016.73`
	
	12. `./configure --host=arm-buildroot-linux-uclibcgnueabi --disable-zlib --disable-syslog --disable-lastlog CC=/home/miraje/Desktop/buildroot-2016.02/output/host/usr/bin/arm-buildroot-linux-uclibcgnueabi-gcc LD=/home/miraje/Desktop/buildroot-2016.02/output/host/usr/bin/arm-buildroot-linux-uclibcgnueabi-ld`
	
	13. Edit `options.h` file and commento the line `153 (ecdsa host key)` and `245 (env password)`
	
	14. `make PROGRAMS="dropbear dbclient dropbearkey dropbearconvert scp" STATIC=1`
