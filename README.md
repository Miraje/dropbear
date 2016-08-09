# dropbear
Dropbear files compiled for the Juno-r2 board and to be used with OP-TEE OS.

Tested on Juno-r2 board at 15:39:32 WEST Tuesday, 9 August 2016.

Note: For this to work it is needed to install `dropbear-bin`: `$ sudo apt install dropbear-bin`

## Compilation
The binaries are already compiled. This were the steps followed:

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

1. Before executing the `make all` to compile the `OPTEE-OS` apply the patch to include the dropbear files into the rootfs.

2. Now compile the `OPTEE-OS` with `make all`

3. Flash the board as described in the `OP-TEE` github page

4. Boot the board and entrer into the terminal. 

5. Create a `wtmp` file into the `/var/log/` directory:

    `root@FVP:/ touch /var/log/wtmp`

6. Change the `root` password:
    ```
    root@FVP:/ passwd
    Changing password for root
    New password: [NEW_PASSWORD]
    Retype password: [NEW_PASSWORD]
    Password for root changed by root
    root@FVP:/
    ```
    Replace `[NEW_PASSWORD]` with you new password.
    
7. Check if you can reach the network:

   7.1. Ping the google dns: `root@FVP:/ ping 8.8.8.8`. 
   
   Note: confirm that you have the ethernet cable connected to the board in the back panel in the reserved port as described in https://static.docs.arm.com/den0928/f/DEN0928F_juno_arm_development_platform_gsg.pdf page 12.
   
   7.2. If you dont get: `ping: sendto: Network is unreachable` jump to step 8 otherwise continue.
   
   7.3. Now it is needed to set the the ip address of eth0 and route gateway
   
   After the boot of the board there was some information about the ip address that was available as the following example shows:
   ```
   Sending discover...
   Sending select for 149.198.57.245...
   Lease of 149.198.57.245 obtained, lease time 14400
   running rc.d services...
   ```
   Use that ip that was available for eth0: `root@FVP:/ ifconfig eth0 149.198.57.245`
   
   Now it is needed to set the default gateway and to get it you need to use your computer (with ethernet connection to the same network here the juno-r2 board is connected). For linux machines do the `route -n` and for windows machines `ipconfig` and copy the default gateway value.
   
   Example on a windows machine: 
   ```
   Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : xxxxxxxxxxxxxxx
   Link-local IPv6 Address . . . . . : xxxxxxxxxxxxxxx
   IPv4 Address. . . . . . . . . . . : xxxxxxxxxxxxxxx
   Subnet Mask . . . . . . . . . . . : xxxxxxxxxxxxxxx
   Default Gateway . . . . . . . . . : 149.198.57.1
   ```
  
   Add the route: `root@FVP:/ route add -net 0.0.0.0 netmask 0.0.0.0 gw 149.198.57.1`
 
   Now execute the ping command again and it should work.
 
8. Launch `dropbear` in the backgroud:

```
root@FVP:/ dropbear
[1052] Aug 09 14:27:48 Running in background
```

9. Connect by `ssh` or send files by `scp` form your computer to the `root@149.198.57.245`.

Patch:

```diff
project build/
diff --git a/juno.mk b/juno.mk
index a19df94..cb3bd82 100644
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
 
@@ -31,6 +34,17 @@ all-clean: arm-tf-clean busybox-clean u-boot-clean optee-os-clean \
 -include toolchain.mk
 
 ################################################################################
+# Dropbear
+################################################################################
+dropbear:
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
@@ -163,8 +177,22 @@ filelist-tee:
 	@echo "slink /lib/libteec.so.1 libteec.so.1.0 755 0 0" >> $(GEN_ROOTFS_FILELIST)
 	@echo "slink /lib/libteec.so libteec.so.1 755 0 0" >> $(GEN_ROOTFS_FILELIST)
 
+.PHONY: filelist-dropbear
+filelist-dropbear:
+	@echo "# Dropbear files." >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /home 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /home/root 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "dir /etc/dropbear 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /sbin/dropbear $(DROPBEAR_PATH)/dropbear 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dbclient $(DROPBEAR_PATH)/dbclient 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dropbearconvert $(DROPBEAR_PATH)/dropbearconvert 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/dropbearkey $(DROPBEAR_PATH)/dropbearkey 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /bin/scp $(DROPBEAR_PATH)/scp 755 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /etc/dropbear/dropbear_dss_host_key $(DROPBEAR_PATH)/dropbear_dss_host_key 444 0 0" >> $(GEN_ROOTFS_FILELIST)
+	@echo "file /etc/dropbear/dropbear_rsa_host_key $(DROPBEAR_PATH)/dropbear_rsa_host_key 444 0 0" >> $(GEN_ROOTFS_FILELIST)
+
 .PHONY: update_rootfs
-update_rootfs: u-boot busybox optee-client xtest helloworld filelist-tee
+update_rootfs: u-boot busybox optee-client xtest helloworld filelist-tee filelist-dropbear
 	cat $(GEN_ROOTFS_PATH)/filelist-final.txt $(GEN_ROOTFS_PATH)/filelist-tee.txt > $(GEN_ROOTFS_PATH)/filelist.tmp
 	cd $(GEN_ROOTFS_PATH) && \
 	        $(LINUX_PATH)/usr/gen_init_cpio $(GEN_ROOTFS_PATH)/filelist.tmp | gzip > $(GEN_ROOTFS_PATH)/filesystem.cpio.gz
```

