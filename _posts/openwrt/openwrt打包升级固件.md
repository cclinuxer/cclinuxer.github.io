看下打印：

make image V=s -j1 --just-print

输出：

using cached model: v6_jd
rm -fr /home/jie/share/src/v6_jd_new/mesh/image/v6_jd
mkdir -p /home/jie/share/src/v6_jd_new/mesh/image/v6_jd
rm /home/jie/share/src/v6_jd_new/mesh/image/mkfw -f ; \
CONFIG_UPFW_FWID_LIMIT=`sed -n -e "/^CONFIG_UPFW_FWID_LIMIT=/{s/CONFIG_UPFW_FWID_LIMIT=//;s/\"//g;p}" /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/openwrt.config` ; \
make --no-print-directory "NOSQUASHFS=1" "UPFW_FWID_LIMIT=$CONFIG_UPFW_FWID_LIMIT" -C tools/mkfw O=/home/jie/share/src/v6_jd_new/mesh/image/
cc -I../../qihoo/upfw/src -Wall  -DNOSQUASHFS -DUPFW_FWID_LIMIT -o /home/jie/share/src/v6_jd_new/mesh/image/mkfw config.c ../../qihoo/upfw/src/firmware.c mkfw.c -lgcrypt -lz
:
do_ln() { echo "ln -s $1 $2" ; ln -s "$1" "$2" || exit 1 ; } ; \
        cd /home/jie/share/src/v6_jd_new/mesh/image/v6_jd && \
        for image in /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq/openwrt-ipq-ipq60xx-ubi-root.img /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/xbl_nand.elf /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/nand-system-partition-ipq6018.bin /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/bootconfig.bin /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/tz.mbn /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/devcfg.mbn /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/rpm.mbn /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/cdt-AP-CP03-C1_256M16_DDR3.bin /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/openwrt-ipq6018-u-boot.mbn /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/factory.bin ; do \
                do_ln "$image" "`basename $image`" ; \
        done
cd /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq && rm fwtemp -rf && mkdir fwtemp && cd fwtemp && tar xvf /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/dl/qca-wifi-fw-QCA6018_v1.0-WLAN.HK.2.1-01577-QCAHKSWPL_SILICONZ-1.tar.bz2 --strip-components=1
if [ -d "/home/jie/share/src/v6_jd_new/mesh/model/v6_jd/wlfw" ]; then \
        cd /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq/fwtemp; \
        cp /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/wlfw/* . -rf; \
fi
cd /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq && cd fwtemp && mkdir staging_dir && cp PIL_IMAGES/* staging_dir -rf && cp bdwlan* staging_dir && \
/home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/staging_dir/host/bin/mksquashfs4 staging_dir/ wifi_fw.squashfs -nopad -noappend -root-owned -comp \
        xz -Xpreset 9 -Xe -Xlc 0 -Xlp 2 -Xpb 2 -Xbcj arm -b 256k -processors 1 && \
dd if=wifi_fw.squashfs of=wifi_fw_squashfs.img bs=2k conv=sync && \
mv wifi_fw_squashfs.img /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq
cp /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/ubinize /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq
cp /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/ipq6018-ubinize.cfg /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq
#cp /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/wifi_fw_squashfs.img /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq
cd /home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq && ./ubinize -m 2048 -p 128KiB -o root.ubi ./ipq6018-ubinize.cfg
dd if=/home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq/root.ubi of=/home/jie/share/src/v6_jd_new/mesh/qca/ipq6018/bin/ipq/openwrt-ipq-ipq60xx-ubi-root.img bs=2k conv=sync 
cat /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/firmware_id
mv /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/firmware_id /home/jie/share/src/v6_jd_new/mesh/image/firmware_id
cat /home/jie/share/src/v6_jd_new/mesh/image/firmware_id
cd /home/jie/share/src/v6_jd_new/mesh/image/v6_jd/ && \
        /home/jie/share/src/v6_jd_new/mesh/image/mkfw -v `cat /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/softver` -c /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/firmware.config -k /home/jie/share/src/v6_jd_new/mesh/misc/fw_rsa.priv \
        -t "`git rev-parse HEAD`" \
        -u /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-upgrade.bin -f /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-flash.bin -p /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-partition-table && \
        [ -f /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/make_factory_flash.sh ] && \
        bash /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018/make_factory_flash.sh /home/jie/share/src/v6_jd_new/mesh/misc/ipq6018 /home/jie/share/src/v6_jd_new/mesh/image/v6_jd `ls .. -1 | grep '.[0-9]\-flash.bin'` && \
        cp -frL /home/jie/share/src/v6_jd_new/mesh/image/v6_jd/*-flash.bin /home/jie/share/src/v6_jd_new/mesh/image



打包的命令就是：

 /home/jie/share/src/v6_jd_new/mesh/image/mkfw -v `cat /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/softver` -c /home/jie/share/src/v6_jd_new/mesh/model/v6_jd/firmware.config -k /home/jie/share/src/v6_jd_new/mesh/misc/fw_rsa.priv \
        -t "`git rev-parse HEAD`" \
        -u /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-upgrade.bin -f /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-flash.bin -p /home/jie/share/src/v6_jd_new/mesh/image/%m-%v-partition-table && \



其中：firmware.config 中定义了每个分区的大小，我们也可以根据大小调整分区。如果想要更改分区，需要更改小大小。



model/m6/nand-partition.xml也要进行更改。这个是描述flash的。



调整分区：查看提交：git show d0c08da609a



打包流程请看如下流程：

https://www.cnblogs.com/sammei/p/3968916.html



makefile分析，可以用remake工具