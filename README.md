
## Changes 

* Changed to use zramctl to initialise/manage the zram devices
* Changed the "stop" portion to *not* completely unload the zram kernel module as there might be other zram disks mounted - and unloading them could be catastrophic.
* Changed the logic to not use multiple individual zram devices per core; zramctl can handle multithreading natively (zramctl uses the term "streams" instead of cores)

### Important zram notes 

There is one slightly counterintuitive thing about zram: the *size of the disk* you specify at creation is indeed the maximum amount of RAM that the disk will use... *However*, this does not account for compression. The size specified to zramctl is the *size of the block device that is created and presented to the operating system*. The end result being that you could put 1GB of data into the device, it compresses down to say 256MB and now the disk is "full". 

If you'd like to see this for yourself, try something like this: 

```sh

# if available, you can play with other algorithms to see the speed 
zramctl -f -s 2GiB -a zstd 
#outputs name of zram device ... perhaps /dev/zram0 
mkfs.ext4 /dev/zram0 

mkdir /mnt/zramtest 

mount /dev/zram0 /mnt/zramtest 

zramctl --output-all 
#NAME DISKSIZE  DATA COMPR ALGORITHM STREAMS ZERO-PAGES TOTAL MEM-LIMIT MEM-USED MIGRATED MOUNTPOINT
#/dev/zram0
#           2G 75.4M 11.6M zstd            4       5369 13.1M        0B    13.1M       0B 

cp <some data> /mnt/zramtest/

#you might need to wait a few seconds 
zramctl --output-all 

# compare with what df indicates

df /mnt/zramtest 

```

You'll notice that the "DATA" column will increase and be *roughly* equivalent to the disk usage reported by `df` ... The "COMPR" column indicates the size of the compressed data - which if the data you copied in is highly compressable, should be pretty small. 

**HOWEVER**, the *available free space* reported by `df` will be *the actual size of the uncompressed data*. And when you have filled the volume with -in this case- 2GB of uncompressed data, the disk will be full and you'll get a pleasant error from the kernel telling you that you can't write any more data there. This will happen regardless of the actual size of the compressed data. 

#### An Alternative

The only alternative I've found isn't ideal but it does work (at least in the case of data volumes - I don't think it'd be a good idea at all for swap): There is a setting in `/sys/block/zram<#>/mem_limit` that sets an actual memory usage limit ... that is to say: a memory usage limit of the *compressed* data. So in that case, you can create the zram device with a much larger size and then set mem_limit so that it doesn't eat away all of your memory accidentally. 

You can set the mem limit by writing a value in **bytes** to that file... 1GB = 1073741824 bytes:  `echo 1073741824 > /sys/block/zram0/mem_limit`. You can check to see that it's set by using `zramctl --output-all` and looking at the "MEM-LIMIT" column.

From experience, this solution isn't 100% ideal: The filesystem behaves very strangely indeed if you hit this limit. The trick in my experience is to try to estimate how compressable your data is (test it!) and then use that as a guide on where to set the mem-limit.

For example, say that you want to use 2GB of memory maximum and that you're going to be storing log files in your zram volume. With testing, you find that you can get a compression ratio of about 4:1 (i.e. 1024MB of uncompressed data is compressed to 256MB). In that case, try creating the volume slightly smaller than that ratio - perhaps 6.5 or 7GB and then setting `mem_limit` to 2GB (*in bytes*). Monitor carefully. 

**Disclaimer:** Doing the above could result in some really weird results if you're storing mission-critical data on the zram volume ... I'd really test it to make sure that it's right ***for your use case***. (It's certainly not a universal solution.)

### Testing Notes

Tested on debian/sid with kernel 5.6 and 5.7; Haven't had a chance to test on say an RPi, but should work? 


### Fork Info

Forked from https://debian.salsa.org/jcc/zram-tools and merged with the
Debian package configuration files located in https://debian.salsa.org/debian/zram-tools


### Original README

zram-tools
==========

Scripts for managing zram devices, currently only for zramswap,
but more tools could be implemented in the future, such as managing
/tmp on zram.

zramswap start
--------------

Sets up zram devices and initializes swap. zram doesn't natively support
multiple processors, so by default a zram device is set up for every
core and then swap is initialized on those devices. This is configurable
in the zramswap config file.

zramswap stop
-------------

Removes all current zram swap spaces and devices.

zramswap status
---------------

shows information on data stored in zram swap.

/etc/default/zramswap
---------------------

Configuration file for zramswap.
