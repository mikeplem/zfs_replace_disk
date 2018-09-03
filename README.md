# Replace existing disks in mirror with larger size disks

The best way to test this is with file backed storage. The steps I am taking here match my physical NAS setup. Using file backed disks in this way lets me test my commands without the chance of destroying my actual disks.

Make the disk images - need to be at least 64 MB

disk 1-5 are 64 MB in size 

Disks used for the original mirror

```shell
# dd if=/dev/zero of=/tmp/disk1 bs=4k count=1 seek=16k
# dd if=/dev/zero of=/tmp/disk2 bs=4k count=1 seek=16k
# dd if=/dev/zero of=/tmp/disk3 bs=4k count=1 seek=16k
# dd if=/dev/zero of=/tmp/disk4 bs=4k count=1 seek=16k
# dd if=/dev/zero of=/tmp/disk5 bs=4k count=1 seek=16k
```

Disks used to replace the first two disks of the mirror

diskr1 and diskr1 are 128 MB in size

```shell
# dd if=/dev/zero of=/tmp/diskr1 bs=4k count=1 seek=32k
# dd if=/dev/zero of=/tmp/diskr2 bs=4k count=1 seek=32k
```

Make the zpool mirror with a hot spare

```shell
# sudo zpool create testpool mirror /tmp/disk1 /tmp/disk2 mirror /tmp/disk3 /tmp/disk4 spare /tmp/disk5

# zpool status

  pool: testpool
 state: ONLINE
  scan: none requested
config:

        NAME            STATE     READ WRITE CKSUM
        testpool        ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            /tmp/disk1  ONLINE       0     0     0
            /tmp/disk2  ONLINE       0     0     0
          mirror-1      ONLINE       0     0     0
            /tmp/disk3  ONLINE       0     0     0
            /tmp/disk4  ONLINE       0     0     0
        spares
          /tmp/disk5    AVAIL   

errors: No known data errors
```

Check the size of the pool

```shell
# df -h /testpool
Filesystem    Size    Used   Avail Capacity  Mounted on
testpool       48M    228K     48M     0%    /testpool
```

Set the pool to autoexpand so the disk size will grow automatically.

```shell
# sudo zpool set autoexpand=on testpool
```

Create a junk file to verify that the disk replacement will not ruin files

This is a 4 MB file

```shell
# sudo dd if=/dev/random of=/testpool/testfile bs=4k count=1 seek=1k
```

Get the MD5 checksum of the file

```shell
# md5 /testpool/testfile | sudo tee /testpool/md5sum.txt
MD5 (testfile) = 0cb3ec29519c5ac0762d7f8dc0098baf


nas:/testpool $ ls -l
total 141
-rw-r--r--  1 root  wheel       50 Sep  1 18:52 md5sum.txt
-rw-r--r--  1 root  wheel  4198400 Sep  1 18:51 testfile
```

Take the disk offline that you want to replace.

```shell
# sudo zpool offline testpool /tmp/disk1

# zpool status testpool
  pool: testpool
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 252K in 0h0m with 0 errors on Sat Sep  1 19:11:41 2018
config:

        NAME                      STATE     READ WRITE CKSUM
        testpool                  DEGRADED     0     0     0
          mirror-0                DEGRADED     0     0     0
            17951635017625177154  OFFLINE      0     0     0  was /tmp/disk1
            /tmp/disk2            ONLINE       0     0     0
          mirror-1                ONLINE       0     0     0
            /tmp/disk3            ONLINE       0     0     0
            /tmp/disk4            ONLINE       0     0     0
        spares
          /tmp/disk5              AVAIL   

errors: No known data errors
```

If this were a real disk, shutdown your computer and if you use an external drive bay where the disk is being replaced, turn it off as well. Then replace the disk. Once the disk is replaced, turn your devices back on.

Replace the old disk with the new one.

```shell
# sudo zpool replace testpool /tmp/disk1 /tmp/diskr1

# sudo zpool status
  pool: testpool
 state: ONLINE
  scan: resilvered 256K in 0h0m with 0 errors on Sat Sep  1 19:16:44 2018
 config:
        NAME             STATE     READ WRITE CKSUM
        testpool         ONLINE       0     0     0
          mirror-0       ONLINE       0     0     0
            /tmp/diskr1  ONLINE       0     0     0
            /tmp/disk2   ONLINE       0     0     0
          mirror-1       ONLINE       0     0     0
            /tmp/disk3   ONLINE       0     0     0
            /tmp/disk4   ONLINE       0     0     0
        spares
          /tmp/disk5     AVAIL   

errors: No known data errors
```

ZFS is now going to resilver the new drive.  This is going to take a while.  You **must** wait for this to complete before you perform the next disk replacement.

Verify the md5 checksum of the file in the pool.

```shell
# md5 /testpool/testfile
MD5 (testfile) = 0cb3ec29519c5ac0762d7f8dc0098baf
```

Replace the second disk in the mirror - for brevity I will not show output of commands since it is the same as the first disk

```shell
# sudo zpool offline testpool /tmp/disk2
```

Turn off your computer and drive bay, if you have one, and put the new disk in and then turn the devices back on.

```shell
# sudo zpool replace testpool /tmp/disk2 /tmp/diskr2
```

Remember that a resilver is going to take place now.

Lets see the size of the pool now

```shell
# df -h /testpool
Filesystem    Size    Used   Avail Capacity  Mounted on
testpool       80M    228K     79M     0%    /testpool
```

Replacing a hot spare - Create a new dummy replacement spare

```shell
# dd if=/dev/zero of=/tmp/disk6 bs=4k count=1 seek=16k
```

Remove the existing spare

```shell
# sudo zpool remove testpool /tmp/disk5
```

Add the new spare

```shell
# sudo zpool add testpool spare /tmp/disk6
```
