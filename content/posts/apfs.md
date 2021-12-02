
---
title: "APFS"
date: 2021-06-22T19:04:24+08:00
draft: False
---


# APFS NOTES
This blog just records something when I port apfs(Apple file system) to Linux.
Please forgive my english ;)


## APFS maximum subvolume number limit

A complete APFS consists of one container super block (nx_superblcok) and multi
subvolumes.
Each subvolume has its own superblock, fs tree, backref tree.
And Thanks to Apple's good design, the maximum subvolume numer of one APFS is
100. It's defined in struct apfs_nx_superblock:

```
#define APFS_MAX_FILE_SYSTEMS 100

struct apfs_nx_superblock {
        ...
        /* This value must not be larger than the value of APFS_MAX_FILE_SYSTTEM */
        __le32 max_file_systems;
    
        __le64 fs_oid[APFS_MAX_FILE_SYSTEMS];
        ...
};
```

I have tried to create the 101th subvolume in macos, it failed as expected.
Unlike btrfs which treats snapshots as subvolumes, apfs create snapshot related
infomation inside subvolume. It means we can create snapshots of a subvolume 
as much as we want. However, the trade cost is only one snapshot/subvolume per
apfs_vol_superblock being mounted meantime.

## APFS superblock and its csum

nx_superblock and vol_superblock:
```
#define APFS_SUPER_INFO_SIZE 4096

struct apfs_nx_superblock {
        struct apfs_obj_header o;
        __le32 magic; // must be APFS_MAGIC
        
        __le32 block_size;
        __le64 block_count;
        __le64 features;
        __le64 readonly_compatible_features;
        __le64 incompatible_features;
        ...
};

/* Volume superblock */
struct apfs_vol_superblock {
        struct apfs_obj_header o;
        __le32 magic; /* Must be 'BSPA' */
        
        __le32 fs_index;

        __le64 features;
        __le64 readonly_compat_features;
        __le64 incompat_features;
        __le64 unmount_time;
        __le64 reserved;
        __le64 quota;
        __le64 allocated;
        struct apfs_wrapped_meta_crypto_state meta_crypto;
        ...
};
```

In APFS, every tree node, container and subvolume superblocks have their
obj_header members. Btrfs has similar designs for tree nodes and file extents.
Because apfs seems don't CoW its tree node, every object item has its csum field
in obj header.

Object header structure:

```
/* Header of all objects */
struct apfs_obj_header {
        u8 csum[APFS_CSUM_SIZE];
        __le64 oid;
        __le64 xid;
        /*
         * The low 16 bits value means object type.
         * The high 16 bits are mixed flags.
         */
        __le32 type;
        __le32 subtype;
};
```

And APFS generates csum using fletcher64:
```
/*
 * Note that this is not a generic implementation of fletcher64, as it assumes
 * a message length that doesn't overflow sum1 and sum2.  This constraint is ok
 * for apfs, though, since the block size is limited to 2^16.  For a more
 * generic optimized implementation, see Nakassis (1988).
 */
static u64 apfs_fletcher64(const void *addr, size_t len)
{
        const __le32 *buff = addr;
        u64 sum1 = 0;
        u64 sum2 = 0;
        u64 c1, c2;
        int i;

        for (i = 0; i < len / sizeof(u32); i++) {
                sum1 += le32_to_cpu(buff[i]);
                sum2 += sum1;
        }

        c1 = sum1 + sum2;
        c1 = 0xFFFFFFFF - c1 % (u64)0xFFFFFFFF;
        c2 = sum1 + c1;
        c2 = 0xFFFFFFFF - c2 % (u64)0xFFFFFFFF;

        return (c2 << 32) | c1;
}
```
@addr in APFS should be address of @apfs_obj_header and len should 
be (node_size - APFS_CSUMS_SIZE) or (APFS_SUPER_INFO_SIZE - APFS_CSUMS_SIZE).


## APFS COW

Unlike btrfs, nowdays xfs has data COW ability but no metadata COW.

After dumping APFS data structures, I realized APFS, like xfs does not do
metadata COW but data COW only. There are data's extent backrefs in every
subvolume but no metadata's.
And one more thing shocked me: apfs does metadata checksum indeed but no checksum
of data at all????!!!! Mabye it's Apple's dignity in protecting user data :)

{{< youtube wG8FUvSGROw >}}

## Credits:

* https://github.com/linux-apfs/linux-apfs-rw/
* https://github.com/sgan81/apfs-fuse/
* https://developer.apple.com/support/downloads/Apple-File-System-Reference.pdf