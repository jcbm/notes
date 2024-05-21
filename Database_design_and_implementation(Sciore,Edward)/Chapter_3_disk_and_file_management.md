
# Chapter 3 - Disk and file management

## 3.1 Persistent Data Storage 

### 3.1.1 Disk Drives

Disk drive consists of multiple platters.\
A platter consists of concentric tracks - many thousands.\
Pairs of platters are joined back-to-back - both sides are used for efficiency.\
A track is divided into sectors (see below).\
A track consists of a sequence of bytes.

A movable arm with a read/write head is used to read/write bytes to the platter 
- one head pr. platter
- all heads are always in the same position - connected to a single actuator that moves them simultaneously to the same track on each platter
- only one read/write head can be active at a time, because there is only one datapath to the computer 

The arm is positioned at the desired track. The head reads the bytes as they rotate under it.

Performance of a disk drive can be described by
- capacity
-- number of bytes that can be stored
-- depends on
--- number of platters
--- tracks per platter
--- bytes per track.
--  platters come standard sizes,, manufacturers primarily increase capacity by squeezing more tracks per platter and more bytes per track ("density")
--- Platter capacities > 40 GB are common
- rotation speed
-- rate at which the platters spin (RPM) 
-- Typically from 5400 rpm to 15,000 rpm
- transfer rate
-- speed at which bytes pass by the disk head, to be transferred to/from memory
-- a track’s worth of bytes can be transferred in the time it takes for the platter to make a single revolution 
-- determined by both the rotation speed and the number of bytes per track
-- 100 MB/s is common
- seek time
-- time it takes for the actuator to move the disk head from its current location to a requested track 
-- depends on how many tracks need to be traversed
--- 0 if the destination track is the same as the starting track
--- 15–20 ms if the destination and starting tracks are at different ends of the platter 
-- *Average* seek time provides a reasonable estimate of actuator speed
--- About 5 ms on modern disks 

### 3.1.2 Accessing a Disk Drive
Disk access - a request to read some bytes from the disk drive into memory or to write some bytes from memory to disk.\ 
*These bytes must be on a contiguous portion of a track on some platter.*

Stages
- Drive moves the disk head to the specified track (*seek time*)
- Waits for the platter to rotate until the first desired byte is beneath the disk head (*rotational delay*)
- As the platter continues to rotate, it reads/writes each byte that appears under the disk head, until the last desired byte appears (*transfer time*)

Disk access time is the sum of seek time, rotational delay, and transfer time.

Each step is constrained by the mechanical movement of the disk.

Significantly slower than electrical movement, which is why disk drives are so much slower than RAM. 

The seek time and rotational delay are nothing but overhead that every disk operation is forced to wait for, while transfer time is spent doing the actual reading/writing.

Calculating the exact seek time and rotational delay requires knowing the previous state of the disk. 
You can estimate these times by using their average: 
You already know about the average seek time. 
The average rotational delay is easily calculated - Can be as low as 0 (if the first byte just happens to be under the head) and as high as the
time for a complete rotation (if the first byte just passed by the head). 
On average, you will have to wait ½ rotation until the platter is positioned where you want it. 
I.e. the average rotational delay is half of the rotation time.

The transfer time can ne calculated from the transfer rate. 
If the transfer rate is r bytes/second and you are transferring b bytes, then the transfer time is b/r seconds.

The estimated access time for 1000 bytes is essentially the same as for 1 byte. 
*In other words, it makes no sense to access a few bytes from disk. *
Not even possible
- on modern disks each track is divided into fixed-length sectors; 
- a read/write must operate on an entire sector at a time. 
- Sector size may be determined by the manufacturer, or chosen when the disk is formatted - typically 512 bytes.

### 3.1.3 Improving Disk Access Time

#### Disk Caches
- memory bundled with the disk drive 
- usually large enough to store the contents of thousands of sectors 
- When a sector is read, the contents are stored in the cache
- if the cache is full, the new sector replaces an old sector 
- When a sector is requested, the disk drive checks the cache
-- If already in the cache, it can be returned immediately without an actual disk access

Suppose that an application requests the same sector more than once in a relatively short period. 
The first request will bring the sector into the cache and subsequent requests will retrieve it from the cache, thereby saving on disk accesses.

This feature is not particularly useful for a database engine, because it is already doing its own caching.
If a sector is requested multiple times, the engine will find the sector in its own cache and not even bother to go to the disk.

*The real value of a disk cache is its ability to pre-fetch sectors.*
Instead of reading just a requested sector, the disk drive can read a sector's entire track into the cache, hoping that other sectors of the track will be requested later. 
Reading an entire track is only a little more expensive than reading a single sector. 
In particular, *there is no rotational delay *. The disk can read the track starting from whatever sector happens to be under the read/write head and continue reading throughout the rotation.

I.e.
Time to read a sector = seek time + ½ rotation time + sector rotation time\
Time to read a track = seek time + rotation time

*The difference between reading a single sector and a track full of sectors is less than half the disk rotation time.* 
If the database engine needs just one other sector on the track, then reading the entire track into the cache will have saved time.

#### Cylinders
The database system can improve disk access time by storing related information in nearby sectors.
I.e. *the ideal way to store a file is to place its contents on the same track of a platter.* 

This works best if the disk does track-based caching, because the entire file will be read in a single disk access. 
But even without caching it eliminates seek time; each time another sector is read, the disk head will already be located at the proper track.

Suppose that a file occupies more than one track. 
It would make sense to store it in nearby tracks of the platter so that the seek time between tracks is as small as possible.
However, it's even better to store its contents on the *same* track of *other* platters. 
Since the read/write heads of each platter move together, all of the tracks having the same track number can be accessed without any additional seek time.

*The set of tracks having the same track number is called a cylinder, because if you look at those tracks from the top of the disk, they describe the outside of a cylinder.*
A cylinder can be treated as if it were a very large track, because all its sectors can be accessed with zero additional seeks.

#### Disk Striping
Disk access can be improved with multiple disk drives. 
Two small drives are faster than one large drive because they contain two independent actuators and thus can respond to two different sector requests simultaneously. 

Two 20 GB disks, working continuously, will be about twice as fast as a single 40 GB disk. 
This speedup scales well: in general, N disks will be about N times as fast as a single disk
Trade-off with cost: several smaller drives are more expensive than a single large drive.

However, the efficiency of multiple small disks will be lost if they cannot be kept busy. 
E.g. one disk contains frequently used files, while the other disks contains rarely used, archived files. 
The first disk would be doing all of the work, with the other disks standing idle most of the time. 
Same efficiency as a single disk.

To balance the workload among the multiple disks, the DBA could try to analyze file usage in order to best distribute the files on each disk.
Not practical: It is difficult to do, hard to guarantee, and would have to be continually reevaluated and revised over time.

*Disk striping* is better.  
Uses a controller to hide the smaller disks from the operating system, giving it the illusion of a single large disk. 
The controller maps sector requests on the virtual disk to sector requests on the actual disks.

Suppose there are N small disks, each having k sectors. 
The virtual disk will have N*k sectors; these sectors are assigned to sectors of the real disks in an alternating pattern. 
Disk 0 will contain virtual sectors 0, N, 2N, etc. 
Disk 1 will contain virtual sectors 1, N+1, 2N+1, etc., and so on. 

Most controllers allow a user to define a stripe to be of any size. 
A track makes a good stripe if the disk drives are also performing track-based disk caching. 
The optimal size depends on many factors, often determined by trial and error.

Disk striping is effective because it distributes the database equally among the small disks. 
If a request arrives for a random sector, then that request will be sent to one of the small disks with equal probability. 
If several requests arrive for contiguous sectors, they will be sent to different disks. 
Thus the disks are guaranteed to be working as uniformly as possible.

### 3.1.4  Improving Disk Reliability by Mirroring
Disk drives can fail
- The magnetic material on a platter can degenerate, causing sectors to become unreadable.
- a piece of dust or a jarring movement could cause a read/write head to scrape against a platter, ruining the affected sectors (a “head crash”).

One way to guard against disk failure is to keep a copy of the disk’s contents. 
- nightly backups of the disk; when a disk fails, you simply buy a new disk and copy the backup onto it. 
-- however, you lose all of the changes to the disk that occurred between the time that the disk was backed up and the time when it failed. 
-- The only way around this problem is to replicate every change to the disk at the moment it occurs. 
-- I.e. you need to keep two identical versions of the disk ("mirrors") 

As with striping, a controller is needed to manage the two mirrored disks. 
- On read, the controller can access the specified sector of either disk. 
- On write, the controller performs the same write to both disks. 

In theory, these two disk writes could be performed in parallel, which would require no additional time. 
In practice, however, it is important to write the mirrors sequentially to guard against a system crash. 
If the system crashes in the middle of a disk write, the contents of that sector are lost. 
If both mirrors are written in parallel, both copies of the sector could be lost
if the mirrors are written sequentially, then at least one of the mirrors will be uncorrupted.

Suppose that one disk from a mirrored pair fails. 
The DBA can recover the system by performing the following procedure:
- Shut down the system.
- Replace the failed disk with a new disk.
- Copy the data from the good disk onto the new disk.
- Restart the system.

Unfortunately, this procedure is not fool-proof. 
Data can still get lost if the good disk fails while it is in the middle of copying to the new disk.
The chance of both disks failing within a couple of hours of each other is small (1/60,000
with today’s disks).

If the database is important, this small risk might be unacceptable. 
You can reduce the risk by using three mirrored disks instead of two. 
Then the data would be lost only if all three disks failed within the same couple of hours; extremely unlikely to all fail. 

Mirroring can coexist with disk striping. 
A common strategy is to mirror the striped disks. 
- E.g., storing 40 GB of data on four 20 GB drives: 
- Two of the drives would be striped, the other two would be mirrors these. 
- Such a configuration is fast and reliable.

### 3.1.5 Improving Disk Reliability by Storing Parity
Mirroring requires twice as many disks to store the same amount of data. 
This burden is particularly noticeable when disk striping is used— To store 300 GB of data using 15 20GB drives, then you will need another 15 drives to be their mirrors. 
It is common for large database installations to create a huge virtual disk by striping many small disks, and the prospect of buying an equal number of disks just to be mirrors is unappealing. 

It would be nice to be able to recover from a failed disk without using so many mirror disks.
There is a clever way to use a single disk to back up any number of other disks. 
The strategy works by storing parity information on the backup disk. 

Parity is defined for a set S of bits as follows:
- 1 if it contains an odd number of 1s.
- 0 if it contains an even number of 1s.

I.e. if you add the parity bit to S, you will always have an even number of 1s.

Parity has the property that the value of any bit can be determined from the value of the other bits, as long as you know the parity.

For example, suppose that S={1, 0, 1}. 
The parity of S is 0 because it has an even number of 1s. 
Suppose you lose the value of the first bit. 
Because the parity is 0, the set {?, 0, 1} must have had an even number of 1s; thus, you can infer that the
missing bit must be a 1. 
Similar deductions can be made for each of the other bits (including the parity bit).

This use of parity extends to disks. 
Suppose you have N + 1 identically sized disks. 
You choose one of the disks to be the parity disk and let the other N disks hold the striped data. 
Each bit of the parity disk is computed by finding the parity of the corresponding bit of all the other disks. 
If any disk fails (including the parity disk), the contents of that disk can be reconstructed by looking, bit by bit, at the contents of the other disks. 

The disks are managed by a controller. 
Read and write requests are handled similarly to striping; the controller determines which disk holds the requested sector and performs that read/write operation. 
The difference is that write requests must also update the corresponding sector of the parity disk.
The controller can calculate the updated parity by determining which bits of the modified sector changed; the rule is that if a bit changes, then the corresponding parity bit must also
change. 
Thus, the controller requires four disk accesses to implement a sector-write operation: 
it must read the sector and the corresponding parity sector (in order to calculate the new parity bits), and it must write the new contents of both sectors.
This use of parity information is somewhat magical, in the sense that one disk is able to reliably back up any number of other disks. 

Drawbacks to to using parity 
- a sector-write operation is more time-consuming, as it requires both a read and a write from two disks. 
-- using parity reduces the efficiency of striping by a factor of about 20%.
- the database is more vulnerable to a non-recoverable multi-disk failure. 
-- When a disk fails all of the other disks are needed to reconstruct the failed disk, and the failure of any one of them is disastrous. 
-- If the database is comprised of many small disks (~100), then the possibility of a second failure becomes very real. 
--- To recover from a failed disk with mirroring we only need the other disk to not fail, which is much less likely.

## 3.1.6 RAID

Striping, mirroring and parity use a controller to hide the existence of the multiple disks from the OS and provide the illusion of a single, virtual disk. 
The controller maps each virtual read/write operation to one or more operations on the underlying disks. 
The controller can be implemented in software or hardware, although hardware controllers are more widespread.

These strategies are part of a larger collection of strategies known as RAID, which stands for Redundant Array of Inexpensive Disks. 

RAID levels.
- RAID-0 is striping
-- No guard against disk failure. 
--- If one of the disks fails, then the entire database is potentially ruined
- RAID-1 is mirrored striping.
- RAID-2 uses bit striping instead of sector striping 
-- has a redundancy mechanism based on error-correcting codes instead of parity 
-- Difficult to implement and has poor performance. It is no longer used.
- RAID-3 and RAID-4 use striping and parity. 
-- RAID-3 uses byte striping
-- RAID-4 uses sector striping. 
--In general, sector striping tends to be more efficient because it corresponds to the unit of disk access
- RAID-5 is similar to RAID-4, except that instead of storing all the parity information on a separate disk, the parity information is distributed among the data disks 
-- if there are N data disks, then every Nth sector of each disk holds parity information 
-- more efficient than RAID-4 because there is no longer a single parity disk to become a bottleneck
- RAID-6 is similar to RAID-5, except that it keeps two kinds of parity information
-- therefore able to handle two concurrent disk failures but needs another disk to hold the additional parity information

The two most popular RAID levels are RAID-1 and RAID-5.\ 
The choice between them is really one of mirroring vs. parity.\ 
Mirroring tends to be the more solid choice in a database installation
- speed and robustness 
- the cost of the additional disk drives has become so low

### 3.1.7 Flash Drives

Flash memory is a more recent technology with potential to replace disk drives. 
Uses semiconductor technology, similar to RAM, but does not require an uninterrupted power supply. 
Activity is entirely electrical 
- can access data much more quickly than disk drives
- no moving parts to get damaged

Flash drives currently have a seek time of around *50 microseconds* - about 100 times faster than disk drives. 
The transfer rate of current flash drives depends on the bus interface is it connected to. 
Flash drives connected by fast internal buses are comparable to those of disk drives; however, external USB flash drives are slower than disk drives.

Flash memory wears out. Each byte can be rewritten a fixed number of times; attempting to write to a byte that has hit its limit will cause the flash drive to fail.
Currently, this maximum is in the millions, which is reasonably high for most database applications. 
High-end drives employ “wear-leveling” techniques that automatically move frequently written bytes to less-written locations; allows the drive to operate until all bytes on the drive reach their rewrite limit.

A flash drive presents a sector-based interface to the operating system, which makes the flash drive look like a disk drive. 
It is possible to employ RAID techniques with flash drives, although striping is less important because the seek time of a flash drive is so low.

The main impediment to flash drive adoption is its price. 
Prices are currently
about 100 times the price of a comparable disk drive. 
Although the price of both flash and disk technology will continue to decrease, eventually flash drives will be
cheap enough to be treated as mainstream. At that point, disk drives may be relegated
to archival storage and the storage of extremely large databases.
Flash memory can also be used to enhance a disk drive by serving as a persistent
front end. If the database fits entirely in the flash memory, then the disk drive will never get used. But as the database gets larger, the less frequently used sectors will
migrate to disk.
As far as the database engine is concerned, a flash drive has the same properties as
a disk drive: it is persistent, slow, and accessed in sectors. (It just happens to be less
slow than a disk drive.) 

## 3.2 The Block-Level Interface to the Disk
Disks may have different hardware characteristics
- sector size varies 
- sectors can be addressed in different ways. 

The OS hides these details, providing applications with a simple interface for accessing disks.

The notion of a block is central to this interface. 
*A block is similar to a sector except that its size is determined by the OS. *
Each block has the same fixed size for all disks. 

The OS maintains a mapping between blocks and sectors. 
- assigns a block number to each block of a disk
-- given a block number, the OS determines the actual sector addresses.
- a block is  read and modified in memory, never on disk
-- On read, the sectors comprising the block are read into a memory page and then read from there
-. To modify block contents, the block is read into a page, the bytes are modified, and the page is written to the block on disk.

An OS typically provides several methods to access disk blocks, such as:
- readblock(n,p) reads the bytes at block n of the disk into page p of memory.
- writeblock(n,p) writes the bytes in page p of memory to block n of the disk.
- allocate(k,n) finds k contiguous unused blocks on disk, marks them as used, and returns the block number of the first one. 
-- The new blocks must be located as close to block n as possible.
- deallocate(k,n) marks the k contiguous blocks starting with block n as unused.

The OS track of which blocks on disk are available for allocation and which are not. 
Strategies
- *disk map*  
--  a sequence of bits, one bit for each block on the disk. 
--- 1 means the block is free, and a 0 means that the block is already allocated. 
-- The disk map is stored on the disk, usually in its first blocks. 
-- The OS can deallocate block n by simply changing bit n of the disk map to 1. 
-- It can allocate k contiguous blocks by searching the disk map for k bits in a row having the value 1 and then setting those bits to 0.
- *free list*.
- a chain of *chunks*
-- a chunk is a contiguous sequence of *unallocated* blocks. 
-- The first block of each chunk stores the length of the chunk and the block number of the next chunk on the chain.
-- The first block of the disk contains a pointer to the head of the chain. 
--- When OS needs to allocate k contiguous blocks
---- the free list is scanned for a sufficiently large chunk. 
---- When a large enough chunk is found it can either be allocated as a whole and removed from the list or a piece of length k of the chunk can be allocated
-- To deallocate a chunk of blocks, the OS simply inserts it into the free list.


The free list technique requires minimal extra space; all you need is to store an integer in block 0 to point to the first block in the list. 
Disk map requires space to hold the map; often several blocks may be required. 
The advantage of a disk map is that it gives the OS a better picture of where the “holes” in the disk are. 
Disk maps are often used if the OS needs to support the allocation of multiple blocks at a time.

## 3.3 The File-Level Interface to the Disk

another, higher-level interface to the disk
A client views a file as *a named sequence of bytes*. 
No notion of block at this level. 
Instead, a client can read (or write) any number of bytes starting at any position in the file.

The Java class RandomAccessFile provides a typical API to the file system.
- holds a file pointer that indicates the byte at which the next read/write operation will occur. 
- can be set explicitly by a call to *seek*. 
- readInt/writeInt moves the file pointer past the integer it read (or wrote).

Note that the calls to readInt and writeInt act as if the disk were being accessed directly, hiding the fact that disk blocks must be accessed through pages.
An OS typically reserves several pages (*I/O buffers*) of memory for its own use 
When a file is opened, the OS assigns an I/O buffer to the file, unbeknownst to the client.

The file-level interface enables a file to be thought of as a sequence of blocks. 
For example, if blocks are 4096 bytes long (i.e., 4K bytes), then byte 7992 is in block
1 of the file (i.e., its second block). 
Block references like “block 1 of the file” are called *logical* block references, because they tell us where the block is with respect to the file, but not where the block is on disk.
Given a particular file location, the seek method determines the actual disk block that holds that location. 

Seek performs two conversions:
- specified byte position to a logical block reference.
-- the logical block number is just the byte position divided by the block size.
-- E.g. assuming 4K-byte blocks, byte 7992 is in block 1 because 7992/4096=1 (integer division).
- logical block reference to a physical block reference.
-- depends on how a file system is implemented.
--- contiguous-, extent-based- or index -allocation


Each of these three strategies stores its information about file locations on disk, in a file system directory.
*seek* accesses the blocks of this directory when it converts logical block references to physical block references.
 You can think of these disk accesses as a hidden “overhead” imposed by the file system. 
OSs try to minimize this overhead, but they cannot eliminate it.

### Continuous Allocation
The simplest strategy, 
stores each file as a sequence of contiguous blocks. 

To implement contiguous allocation, the file system directory holds the length of each file and the location of its first block. 
Mapping logical to physical block references is easy—
if the file begins at disk block b, then block N of the file is in disk block b + N. 

Problems 
- a file cannot be extended if there is another file immediately following it. 
-- i.e. clients must create their files with the maximum number of blocks they might need, leading to wasted space when the file is not full (*internal fragmentation*)
- as the disk gets full, it may have lots of small-sized chunks of unallocated blocks, but no large chunks. 
-- Thus, it may not be possible to create a large file, even though the disk contains plenty of free space (*external fragmentation*)

I.e. *Internal* fragmentation is the wasted space inside a file. 
*External* fragmentation is the wasted space is outside all the files.

### Extent-Based Allocation
Variation of contiguous allocation that reduces both internal and external fragmentation. 
The OS stores a file as a sequence of fixed-length extents, where each extent is a contiguous chunk of blocks.
A file is extended one extent at a time.
 The file system directory for this strategy contains, for each file, a list of the first blocks of each extent of the file.

To find the disk block that holds block N of the file, the seek method searches the file system directory for the extent list for that file; it then searches the extent list to
determine the extent that contains block N, from which it can calculate the location of the block.

Extent-based allocation reduces internal fragmentation because a file can waste no more than an extent’s worth of space.
External fragmentation is eliminated because all extents are the same size.

Indexed Allocation
Doesn’t even try to allocate files in contiguous chunks. 
Each block of the file is allocated individually (in one- block-long extents, if you will). 
The OS implements this strategy by allocating a special index block with each file, which keeps track of the disk blocks allocated to that file. 
An index block ib can be thought of as an array of integers, where the value of ib[N] is the disk block that holds logical block N of the file.
To "calculate" the location of any logical block you just look it up in the index block.

This approach has the advantage that blocks are allocated one at a time, so there is no fragmentation. 
Its main problem is that files will have a maximum size, because they can have only as many blocks as there are values in an index block.
The UNIX file system addresses this problem by supporting multiple levels of index block, thereby allowing the maximum file size to be very large.

### 3.4 The Database System and the OS
The OS provides block-level and file-level support for disk access.  
Which level should the implementers of a database engine choose?

Block-level support gives the engine complete control over which disk blocks are used for what purposes. 
E.g. 
- frequently used blocks can be stored in the middle of the disk, where the seek time will be less. 
- blocks that tend to be accessed together can be stored near each other. 

The database engine is not constrained by OS limitations on files, allowing it to support tables that are larger than the OS limit or span multiple disk drives.

Disadvantages:
- complex to implement;
-- requires that the disk be formatted and mounted as a raw disk, that is, a disk whose blocks are not part of the file system
- requires that the DBA has extensive knowledge about block access patterns in order to fine-tune the system.

The other extreme is for the database engine to use the OS file system as much as
possible. 
For example, every table could be stored in a separate file, and the engine would access records using file-level operations. 
This strategy is much easier to implement, and it allows the OS to hide the actual disk accesses from the database system. 

This situation is unacceptable:
- the database system needs to know where the block boundaries are, so that it can organize and retrieve data efficiently
- the database system needs to manage its own pages, because the OS way of managing I/O buffers is inappropriate for database queries

Compromise strategy
The database system stores all of its data in one or more OS files, but treat the files as if they were raw disks. 
I.e. the database system accesses its “disk” using logical file blocks.

The OS is responsible for mapping each logical block reference to its corresponding physical block, via the seek method.
As seek may incur disk accesses when it examines the file system directory, the database system will not be in complete control of the disk.
However, these additional blocks are usually insignificant compared with the large number of blocks accessed by the database system. 
Thus the database system is able to use the high-level interface to the OS while maintaining significant control over disk accesses.

Used in many database systems.
Microsoft Access keeps everything in a single .mdb file, whereas Oracle, Derby, and SimpleDB use multiple files.

## 3.5 The SimpleDB File Manager
File manager: The part of the database engine that interacts with the operating system 

### 3.5.1 Using the File Manager
A SimpleDB database is stored in several files - one for each table and each index, as well as a log file and several catalog files.
 The SimpleDB file manager provides block-level access to these files, via the package simpledb.file. 
 This package exposes three classes: BlockId, Page, and FileMgr. 
 
A BlockId object identifies a specific block by its file name and logical block number. 

A Page object holds the contents of a disk block. 
Its first constructor creates a page that gets its memory from an operating system I/O buffer; used by the buffer manager. 
The second constructor creates a page that gets its memory from a Java array; used primarily by the log manager.

Various get/set methods enable clients to store or access values at specified locations of the page. 
A page can hold three value types: ints, strings, and “blobs” (i.e., arbitrary arrays of bytes). 
Corresponding methods for additional types can be added if desired

A client can store a value at any offset of the page but is responsible for knowing what values have been stored where.
An attempt to get a value from the wrong offset will have unpredictable results.

The FileMgr class handles the actual interaction with the OS file system. 
Its constructor takes two arguments: a string denoting the name of the database and an integer denoting the size of each block. 

The database name is used as the name of the folder that contains the files for the database; this folder is located in the engine’s current directory.  
If no such folder exists, then a folder is created for a new database.
isNew() returns true in this case and false otherwise - needed for the proper initialization of a new database.
 -*read* reads the contents of the specified block into the specified page. 
- *write* performs the inverse operation, writing the contents of a page to the specified block.
- *length* returns the number of blocks in the specified file.

The engine has one FileMgr object, which is created during system startup.
The class SimpleDB (in package simpledb.server) creates the object, and fileMgr() returns the created object.

### 3.5.2 Implementing the File Manager

The class BlockId
The code for class BlockId appears in Fig. 3.13. 

fileName and number, equals, hashCode, and toString.

The class Page
Each page is implemented using a Java ByteBuffer object. 
A ByteBuffer object wraps a byte array with methods to read and write values at arbitrary locations of the array. 
These values can be primitive values (such as integers) as well as smaller byte arrays. 

E.g.
setInt saves an integer in the page by calling the ByteBuffer’s putInt method. 

setBytes saves a blob as two values: first the number of bytes in the blob and then actual bytes of the blob. 
It calls ByteBuffer’s putInt method to write the integer and the method put to write the bytes.

The ByteBuffer class does not have methods to read and write strings, so string values are implemented to be written as blobs. 
Java's String.getBytes converts a string into a byte array; it also has a constructor that converts the byte array back to a string. 
Thus, Page’s setString method calls getBytes to convert the string to bytes and then writes those bytes as a blob.
Similarly, getString reads a blob from the byte buffer and then converts the bytes to a string.

The conversion between a string and its byte representation is determined by a character encoding. 
Several standard encodings exist, such as ASCII and Unicode-16. 
The Java Charset class contains objects that implement many of these encodings. 
The constructor for String takes a Charset.
Page uses the ASCII encoding.

A charset chooses how many bytes each character encodes to. 
ASCII - one byte per character
Unicode-16 - between 2 bytes and 4 bytes per character. Consequently, a database engine may not know exactly how many bytes a given string will encode to.
maxLength calculates the maximum size of the blob for a string having a specified number of characters. 
multiplies the number of characters by the max number of bytes per character and adding 4 bytes for the integer that is written with the bytes.

The byte array that underlies a ByteBuffer object can come either from an array or from the OS's I/O buffers. 
The Page class has a constructor for each.
Since I/O buffers are a valuable resource, the use of the first constructor is carefully controlled by the buffer manager. 
Other components of the database engine (such as the log manager) use the other constructor.

The class FileMgr
Primary job is to implement methods that read and write pages to disk blocks. 
read seeks to the appropriate position in the specified file and reads the contents of that block to the byte buffer of the specified page. 
write is similar. 
append seeks to the end of the file and writes an empty array of bytes to it, which causes the OS to automatically extend the file. 

The file manager always reads or writes a block-sized number of bytes from a file and always at a block boundary. 
In doing so, the file manager *ensures that each call to read, write, or append will incur exactly one disk access.*

Each RandomAccessFile object in the map openFiles corresponds to an open file. 
Files are opened in “rws” mode. 
“rw” specifies that the file is open for reading and writing. 
“s” specifies that the operating system should not delay disk I/O in order to optimize disk performance; every write operation must go immediately to the disk. 
This feature ensures that the database engine knows exactly when disk writes occur, which is important for implementing data recovery algorithms (ch. 5).

read, write, and append are synchronized, i.e. only one thread can execute them at a time. 
Synchronization is needed to maintain consistency when methods share updateable objects, such as the RandomAccessFile objects. 

The following scenario could occur if read were not synchronized: 
Suppose that two JDBC clients, each running in their own thread, are trying to read different blocks from the same file.
Thread A first starts to execute read but gets interrupted right after the call to f.seek, that is, it has set the file position but has not yet read from it. 
Thread B runs next and executes read to completion. 
When thread A resumes, the file position will have changed, but the thread will not notice it and incorrectly reads from the wrong block.

There is only one FileMgr object in SimpleDB, which is created by the SimpleDB constructor in package simpledb.server. 
The FileMgr constructor determines if the specified database folder exists and creates it if necessary. 
The constructor also removes any temporary files that might have been created by materialized operators (ch. 13)