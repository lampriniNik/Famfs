# Deploy

How to bring the setup up from scratch, or after a restart. VM1 is the famfs master, VM2 is the client.

Lines marked `TODO` still need the exact values/commands from our setup.

## 0. Before anything

Check disk space on the host. The kernel build needs several GB, and when `/mnt/sdd2` hit 100% both VMs paused with `io-error`.

```
df -h /mnt/sdd2
```

If it's above ~95%, ask the admins first. We can't free space owned by other users.

## 1. Backing file (once)

```
mkdir -p cxl-device
truncate -s <SIZE> cxl-device/cxl-pmem.raw   # TODO: actual size
```

Size should be a multiple of 2 MB. Don't delete or recreate it while the VMs are running.

## 2. Start the VMs

Both VMs must point to the same `mem-path`, with `share=on`. Without `share=on` each VM gets a private copy and nothing is actually shared.

Roughly (TODO: replace with the real commands from `scripts/`):

```
qemu-system-x86_64 \
  -enable-kvm -cpu host -smp <N> \
  -machine pc,nvdimm=on \
  -m <RAM>,slots=2,maxmem=<MAXMEM> \
  -object memory-backend-file,id=mem1,share=on,mem-path=cxl-device/cxl-pmem.raw,size=<SIZE>,align=2M \
  -device nvdimm,id=nvdimm1,memdev=mem1 \
  -drive file=<vm1.img>,if=virtio \
  -netdev user,id=net0,hostfwd=tcp::2224-:22 \
  -device virtio-net-pci,netdev=net0 \
  -nographic
```

VM2 is the same with its own disk image and port 2225.

```
ssh -p 2224 <user>@localhost   # vm1
ssh -p 2225 <user>@localhost   # vm2
```

## 3. devdax in the guests

```
sudo apt install ndctl daxctl
ndctl list -N
ls -l /dev/dax* /dev/pmem*
```

If `/dev/dax0.0` is already there, skip ahead. If you see `/dev/pmem0` instead, the namespace is in fsdax mode and needs converting:

```
sudo ndctl create-namespace -f -e namespace0.0 --mode=devdax --align=2M
```

This writes to the shared memory, so do it from one VM only, then check that the other one sees `/dev/dax0.0` (it may need a reboot). TODO: confirm this is how we did it.

Check alignment:

```
cat /sys/bus/dax/devices/dax0.0/align   # should be 2097152
```

## 4. Sanity check with dax_test

Before famfs, make sure both VMs really see the same memory.

```
gcc -O2 -Wall -o dax_test src/dax_test.c
sudo ./dax_test ...   # on vm1, write   (TODO: args)
sudo ./dax_test ...   # on vm2, read back
```

Once famfs is on the device, don't write to it with `dax_test` anymore. The start of the device holds the famfs superblock and log and you will corrupt it.

## 5. famfs kernel

On a VM:

```
git clone https://github.com/cxl-micron-reskit/famfs-linux.git
cd famfs-linux
git checkout famfs_standalonev3
git rev-parse HEAD                     # save this in kernel/KERNEL_VERSION.md

cp ../famfs-cxl-lab/kernel/famfs-config-6.14 .config
make olddefconfig
make -j$(nproc)

sudo make modules_install
sudo make install
sudo update-initramfs -c -k 6.14.11-famfs_standalone
sudo update-grub
sudo reboot
```

Don't skip `update-initramfs`. That's what broke networking on VM2 the first time.

After the reboot, on both VMs:

```
uname -r                         # 6.14.11-famfs_standalone
ip a
grep famfs /proc/filesystems
ls -l /dev/dax0.0
```

Both VMs need to run the same kernel. TODO: note whether we built twice or copied the build over to VM2.

## 6. famfs user space

On both VMs:

```
git clone https://github.com/cxl-micron-reskit/famfs.git
cd famfs
make            # TODO: check the repo README for the exact build steps
sudo make install
```

The user space has to match the kernel side. If mount fails with odd errors, check that first.

## 7. Master (VM1)

Run `mkfs.famfs` only on the master, and only once. Running it again wipes everything.

```
sudo mkfs.famfs /dev/dax0.0
sudo mkdir -p /mnt/famfs
sudo famfs mount /dev/dax0.0 /mnt/famfs

sudo famfs creat -s 64M /mnt/famfs/data.bin   # empty file, fixed size
sudo famfs cp somefile /mnt/famfs/            # copy an existing file in
sudo famfs cp -r somedir /mnt/famfs/
```

Files have a fixed size from the moment they're created and space is allocated in 2 MB chunks.

## 8. Client (VM2)

```
sudo mkdir -p /mnt/famfs
sudo famfs mount /dev/dax0.0 /mnt/famfs
ls -l /mnt/famfs
```

If the master creates files after the client has mounted, replay the log on the client:

```
sudo famfs logplay /mnt/famfs   # TODO: check syntax for our version
```

Never run `mkfs.famfs` on the client.

## 9. Cross-VM check

```
# vm1, on a file that already exists in famfs
echo "hello from vm1" | sudo tee /mnt/famfs/msg.txt
# vm2
cat /mnt/famfs/msg.txt
```

famfs only keeps its own metadata consistent. For the data, stick to one writer at a time.

## After a restart or power outage

After a power cut the device has to be brought back up by the admins, so ask before doing anything else. Then:

```
df -h /mnt/sdd2
ls -lh cxl-device/cxl-pmem.raw
# start VMs (step 2)
uname -r
ls -l /dev/dax0.0 && daxctl list
sudo famfs mount /dev/dax0.0 /mnt/famfs     # master first, then client. NOT mkfs
ls -l /mnt/famfs
```

famfs isn't meant to be persistent storage. If the mount fails or files are missing, re-run `mkfs.famfs` on the master and recreate the files.

Write down the date of every restart/outage in `docs/notes/`, so it's clear which measurements are valid.

## Troubleshooting

- **VMs show `paused (io-error)`**: host disk full. `df -h`, get space freed, then resume.
- **mmap on `/dev/dax0.0` fails**: size or offset not a multiple of 2 MB.
- **No `/dev/dax0.0`**: namespace still in fsdax mode (step 3), or the device isn't up after an outage.
- **VMs don't see each other's writes**: missing `share=on` or different `mem-path` in the QEMU command.
- **No network after the new kernel**: regenerate the initramfs for that version and reboot.
- **famfs not recognised**: you booted the stock kernel. Check `uname -r` and pick the famfs kernel in GRUB.
- **Client doesn't see new files**: run `famfs logplay`.
- **Numbers look wrong**: check that the device was properly brought back up after the last outage.
