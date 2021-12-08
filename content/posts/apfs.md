
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

## APFS superblock and checksum

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

## Xattr

APFS stores compressed metadat in xattr items.
The following inode "Acknowledgments.html" is located in '/Applications/Safari.app/Contents/Resources/es.lproj/CaptionAutoFillCellView.strings'.
The inode now is compresssed and have acls 'A' and 'bar'. Xattr name 'A' has value 'BBB' and name 'foo' has value 'bar'.
The names 'com.apple.ResourceFork' and 'com.apple.decmpfs' are describtion of the compressed extents.
compress header in 'com.apple.decmpfs' shows the inode extents was compressed by algorithm LZVN and data is in a resource fork.
And com.apple.ResourceFork contains a xattr_dstream which descirbes size info and real extent location objectid 'objectid 27373'.

NOTE: LZVN is implemented in lzfse and APFS uses zlib compression algorithm too.

```
        item 7 key (3458764513820568300[27372 3] INODE 0) keyoff 666 keysize 8 itemoff 3664 itemsize 124
                parent 26206 private 27372 nlink(nkids) 1 size 110546
                mode REG[100644] uid 0 gid 0 bsd flags 0x80020(UF_COMPRESSED|SF_RESTRICTED)
                flags 0x64000(HAS_RSRC_FORK|FAST_PROMOTE|HAS_UNCOMPRESSED_SIZE)
                atime 1637032878540021363 (2021-11-16 11:21:18)
                ctime 1638942408716033431 (2021-12-08 13:46:48)
                mtime 1634527838000000000 (2021-10-18 11:30:38)
                btime 1634527838000000000 (2021-10-18 11:30:38)
                xfields count 1 used 24
                        field type NAME flags DO_NOT_COPY size 21 offset 3764
                ext_name Acknowledgments.html
        item 8 key (4611686018427415276[27372 4] XATTR 2[hash 0 namelen 2] A) keyoff 2353 keysize 12 itemoff 2395 itemsize 8
                flags 2(DATA_EMBEDDED) len 4
                        BBBB
        item 9 key (4611686018427415276[27372 4] XATTR 23[hash 0 namelen 23] com.apple.ResourceFork) keyoff 674 keysize 33 itemoff 3612 itemsize 52
                flags 1(DATA_STREAM) len 48
                        xattr_dstream objectid 27373 size 26439 allocated 28672 cryptoid 0 written 26439 read 0
        item 10 key (4611686018427415276[27372 4] XATTR 18[hash 0 namelen 18] com.apple.decmpfs) keyoff 707 keysize 28 itemoff 3592 itemsize 20
                flags 2(DATA_EMBEDDED) len 16
                        compress header signature 1668116582 8(LZVN RSRC) 110546
        item 11 key (4611686018427415276[27372 4] XATTR 4[hash 0 namelen 4] foo) keyoff 2339 keysize 14 itemoff 2403 itemsize 7
                flags 2(DATA_EMBEDDED) len 3
                        bar
                                                                                                                                                                                                                                                                                                                                                        flags 2(DATA_EMBEDDED) len
```

File extent 'objectid 27373':
```
        item 10 key (4611686018427415276[27372 4] XATTR 18[hash 0 namelen 18] com.apple.decmpfs) keyoff 707 keysize 28 itemoff 3592 itemsize 20
                flags 2(DATA_EMBEDDED) len 16
                        compress header signature 1668116582 8(LZVN RSRC) 110546
        item 11 key (4611686018427415276[27372 4] XATTR 4[hash 0 namelen 4] foo) keyoff 2339 keysize 14 itemoff 2403 itemsize 7
                flags 2(DATA_EMBEDDED) len 3
                        bar
        item 12 key (9223372036854803181[27373 8] FILE EXTENT 0) keyoff 735 keysize 16 itemoff 3568 itemsize 24
                len 28672 flags 0 bytenr 13570891776 cryptoid 0
        item 13 key (3458764513820568302[27374 3] INODE 0) keyoff 751 keysize 8 itemoff 3452 itemsize 116
                parent 26206 private 27374 nlink(nkids) 33 size 0
```

And the standlone file extent is without inode item aboved.

## Credits:

* https://github.com/linux-apfs/linux-apfs-rw/
* https://github.com/sgan81/apfs-fuse/
* https://developer.apple.com/support/downloads/Apple-File-System-Reference.pdf
