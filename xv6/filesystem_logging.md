# Rangkuman Logging

> Logging Layer

- crash recovery caused by many file system operations involve multiple writes to the disk.
- crash after a subset of the writes may leave the on-disk file system in an inconsistent state.
- crash may leave an inode with a reference to a content block marked free
- or it may leave an allocated but unreferenced content block
- inode that refers to a freed block is likely to cause serious problems after a reboot
- kernel might allocate that block to another file and will have two different files pointing unintentionally to the same block
-  If xv6 supported multiple  users,  this  situation  could  be  a  security problem,  since  the  old  file’s  owner  would  be  able  to  read  and  write  blocks  in  the  newfile, owned  by  a  different  user.
- xv6 solve problem of crash with a simple form of logging
- xv6 syscall does not directly write the on-disk file system data structures. it places a description of all the disk writes it wish to make in a log on a disk
- when has logged, it will writes a special commit record to the disk indicating complete
- syscall will copies the writes to on-disk file system data structure and after that process is complete,  syscall erases the log on disk
- if system reboot, the file system code recovers from the crash before running process
- if log marked as containing a complete operation, the recovery code copies the writes to where they belong in on-disk file system
- if log not marked as containing a complete operation, the recovery code ignores the log
- recovery code finish by erasing the log
- if the crash occurs before the operation commit, the log on disk will not be marked complete, recovery code will ignore it, the state of the disk will be as had not even started
- if the crash occurs after the operation commit, the recovery code will replay all operation's writes, perhaps repeating them if the operation had started to write them on the on-disk data structure
- in either case, the log makes operations atomic with respect to crashes
- after recovery, either all of the operations writes appear on the disk or none of them appear

> Log Design

- the log resides at a known fixed location, specified in the superblock
- it consists of a header block followed by a sequence of updated block copies ("logged blocks")
- the header block contains an array of sector numbers, one for each of the logged blocks
- header blocks contain the count of the logged blocks.
- xv6 writes the header block when a transaction commit and set the count to zero after copying the logged blocks to the file system
- crash in the middle of transaction will result zero count in the log header block
- crash after commit will result non-zero count (valueable)
- each syscall indicates the start and end of the sequence of writes that must be the same like crashes.
- the logging system accumulate the writes of multiple syscall into one transaction to allow concurrent execution of file system by different process
- single commit involve the writes of multiple system calls
- logging system only commit when no file system calls are underway to avoid splitting a syscall across transaction
- group commit -> commiting several transactions together
- group commit reduces the number of disk operations because it amortizes the fixed cost of a commit over multiple operations.
- group commit handle the disk system more concurrent writes at the same time, perhaps allowing the disk to write them all during a single disk rotation
- xv6 IDE doesnt support batching, but xv6 file system design allowed
- xv6 dedicates a fixed amount of space on the disk to hold the log
- no single system call can be allowed to write more distinct blocks than there is space in the log
- write and unlink can potentially write many blocks
- unlinking a large file might write many bitmap blocks and a inode
- xv6 write syscall breaks up large writes into multiple smaller writes that fit in the log
- unlink doesnt cause problems because xv6 uses only one bitmap block
- logging system cannot allow a syscall to tart unless it is certain that the syscall writes will fit in the space remaining in the log

> Code: logging

- begin_op waits until the logging system is not currently commiting and until there is enough free log space to hold the writes and executing syscall
- log.outstanding count number of calls
- the log increment reserve space and prevents a commit from occuring during syscall
- the code assume that each system call might write up to MAXOPBLOCKS distinct block
- log_write acts as proxy for bwrite. record the block sector number in memory, reserving it a slot in the log on disk and marks the buffer B_DIRTY to prevent the block cache from evicting it
- block stay in cache until commited
- until commited, the cached copy is the only record of the modification
- it cant write on it place on disk until after commit and other read in the same transaction must see the modifications
- log_write notices when a block is written multiple times during a single transaction and allocate that block the same slot in the log. this optimization called absorption
- by absorbing several disk writes into one, the file system can save log space and can achieve better performance
- end_op first decrement the count of outstanding system call
- if count is zero, it commit current transaction by calling commit()
- four stages in commit() :
-> write_log() copies  each  block  modified  in  the  transaction  from  the  buffer  cache  to  its  slot  in  the  log  on  disk
-> write_head() writes the header block to disk: this is the commit point, and a crash after the write will result in recovery replaying the transaction’s writes from the log
-> install_trans() reads each block from the log and writes it to the proper place in the file system
-> end_op() writes the log header with a count of zero
- commit() has to happen before the next transaction starts writing logged blocks so crash doesnt result in recovery using onne transaction header with the subsequent transaction logged blocks
- recover_from_log() is called from initlog(), that is called during boot before the first user process runs
- it will reads log header and mimics the action of end_op() if the header indicates that the log contains a commited transaction

- - example :
filewrite() {
begin_op();
ilock(f->ip);
r = writei(f->ip, ...);
iunlock(f->ip);
end_op();
}
- code wrapped in loop that breaks up large writes into individual transaction to avoid overflowing the log.
- writei() writes many blocks as part of the transaction

> Code: block allocator

- file and directory content is stored in disk blocks, which allocated from a free pool
- xv6 block allocator maintains a free bitmap on disk, with one bit per block.
- a zero bit indicates that the corresponding block is free
- the program mkfs sets the bits corresponding to the boot sector, superblock, log blocks, inode blocks, and bitmap blocks.

- the block allocator have two function :
-> balloc() allocate a new disk block,
-> bfree() frees a block

- the loop in balloc considers every block, starting at block 0 up to sb.size, the number of blocks in the file system.
- bitmap bit is zero is indicating that it is free
- if balloc finds such a block, it update the bitmap and return the block.
- the loop is split into two pieces for efficiency
- the outer loop reads each block of bitmap bits
- the inner loop checks all BPB bits in a single bitmap block
- Bfree() finds the right bitmap block and clears the right bit
- the exclusive use implied by bread() and brelse() avoids the need for explicit locking
- balloc() and bfree() must be called inside a transactions