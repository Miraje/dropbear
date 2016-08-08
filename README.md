# dropbear
Dropbear files compiled for the Juno-r2 board and to be used with OP-TEE OS.

## Compilation

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

## Instructions

1. Before executing the `make all` to compile the `OPTEE-OS` apply the patch that is in the end if it isn't already.

2. Now compile the `OPTEE-OS` with `make all`

3. Flash as described in the `OP-TEE` github page

3. Boot the board and change the `root` password

4. Launch `dropbear` in the backgroud

5. Connect by `ssh`


Patch:

```diff
project build/
diff --git a/juno.mk b/juno.mk
index a19df94..d89deb5 100644
--- a/juno.mk
+++ b/juno.mk
@@ -20,10 +20,13 @@ ARM_TF_PATH		?= $(ROOT)/arm-trusted-firmware
 U-BOOT_PATH		?= $(ROOT)/u-boot
 U-BOOT_BIN		?= $(U-BOOT_PATH)/u-boot.bin
 
+DROPBEAR_PATH	?= $(ROOT)/dropbear
+
 ################################################################################
 # Targets
 ################################################################################
-all: arm-tf u-boot linux optee-os optee-client xtest helloworld update_rootfs
+all: dropbear arm-tf u-boot linux optee-os optee-client xtest helloworld \
+	update_rootfs
 all-clean: arm-tf-clean busybox-clean u-boot-clean optee-os-clean \
 	optee-client-clean
 
@@ -31,6 +34,18 @@ all-clean: arm-tf-clean busybox-clean u-boot-clean optee-os-clean \
 -include toolchain.mk
 
 ################################################################################
+# Dropbear
+################################################################################
+dropbear:
+	#need to install: sudo apt install dropbear-bin
+	test -d "$(DROPBEAR_PATH)" || \
+	git clone https://github.com/Miraje/dropbear.git $(DROPBEAR_PATH)	
+	test -f "$(DROPBEAR_PATH)/dropbear_rsa_host_key" || \
+	(cd $(DROPBEAR_PATH) && exec dropbearkey -t rsa -s 1024 -f ./dropbear_rsa_host_key)
+	test -f "$(DROPBEAR_PATH)/dropbear_dss_host_key" || \
+	(cd $(DROPBEAR_PATH) && exec dropbearkey -t dss -f ./dropbear_dss_host_key)
+
+################################################################################
 # ARM Trusted Firmware
 ################################################################################
 ARM_TF_EXPORTS ?= \
@@ -163,8 +178,23 @@ filelist-tee:
 	@echo "slink /lib/libteec.so.1 libteec.so.1.0 755 0 0" >> $(GEN_ROOTFS_FILELIST)
 	@echo "slink /lib/libteec.so libteec.so.1 755 0 0" >> $(GEN_ROOTFS_FILELIST)
 
+.PHONY: filelist-dropbear
+filelist-dropbear:
+	@echo "# Changes for the dropbear files." >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /home 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /home/root 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /etc/dropbear 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /sbin/dropbear $(DROPBEAR_PATH)/dropbear 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dbclient $(DROPBEAR_PATH)/dbclient 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dropbearconvert $(DROPBEAR_PATH)/dropbearconvert 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dropbearkey $(DROPBEAR_PATH)/dropbearkey 755 0 0 >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/scp $(DROPBEAR_PATH)/scp 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /etc/dropbear/dropbear_dss_host_key $(DROPBEAR_PATH)/dropbear_dss_host_key 444 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /etc/dropbear/dropbear_rsa_host_key $(DROPBEAR_PATH)/dropbear_rsa_host_key 444 0 0 >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /var/log/wtmp $(DROPBEAR_PATH)/wtmp 755 0 0 >> $(GEN_ROOTFS_FILELIST)
+
 .PHONY: update_rootfs
-update_rootfs: u-boot busybox optee-client xtest helloworld filelist-tee
+update_rootfs: u-boot busybox optee-client xtest helloworld filelist-tee filelist-dropbear
 	cat $(GEN_ROOTFS_PATH)/filelist-final.txt $(GEN_ROOTFS_PATH)/filelist-tee.txt > $(GEN_ROOTFS_PATH)/filelist.tmp
 	cd $(GEN_ROOTFS_PATH) && \
 	        $(LINUX_PATH)/usr/gen_init_cpio $(GEN_ROOTFS_PATH)/filelist.tmp | gzip > $(GEN_ROOTFS_PATH)/filesystem.cpio.gz
```

