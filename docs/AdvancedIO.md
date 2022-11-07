# Advanced IO

This document covers numerous topics and functions that we lump under the term _advanced IO_: _non-blocking IO_, _record locking_, _IO multiplexing_ (the `select` and `poll` functions), _asynchronous IO_, the `readv` and `writev` functions, and memory-mapped IO (`mmap`).

## Nonblocking IO

System calls can loosely be divided into "slow" ones and all others. The slow system calls are those that can block forever. They include:

- Reads that can block the caller forever if data isn't present with certain file types (pipes, terminal devices, and network devices)
- Writes that can block the caller forever if the data can't be accepted immediately by the same file types (e.g. no room in the pipe, network flow control)
- Opens that block until some condition occurs on certain file types (such as an open of terminal device that waits until an attached modem answers the phone, or an open of a FIFO for writing only, when no other process has the FIFO open for reading)
- Reads and writes of files that have mandatory record locking enabled
- Certain `ioctl` operations
- Some of the interprocess communication functions

Note: system calls related to disk IO will not be considered slow, even though the read of write of a disk file can block the caller permanently.

Nonblocking IO lets us issue an IO operation such as `open`, `read`, or `write`, and not have it block forever. If the operaton cannot be completed, the call returns immediately with an error noting that the operation would have blocked.

There are two ways to specify nonblocking IO for a given descriptor:
1. If we call open to get the descriptor, we can specify the `O_NONBLOCK` flag
2. For a descriptor that is already open, we call `fcntl` to turn on the `O_NONBLOCK` file status flag.

Example:

Let us look at an example of nonblocking IO. The program below reads up to 500,000 bytes from the standard input and attempts to write it to the standard output. The standard output is first set to be nonblocking. The output is in a loop, with the results of each `write` being preinted on the standard error. The function `clr_fl` is similar to the function `set_fl`. This new function simply clears one or more of the flag bits.

```cpp
#include <stdio.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/uio.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <string.h>

void set_fl(int fd, int flags);
void clr_fl(int fd, int flags);

char buf[100000];

int main(void) {
  int ntowrite, nwrite;
  char *ptr;

  ntowrite = read(STDIN_FILENO, buf, sizeof(buf));
  fprintf(stderr, "read %d bytes\n", ntowrite);

  set_fl(STDOUT_FILENO, O_NONBLOCK);

  for (ptr = buf; ntowrite > 0; ) {
    errno = 0;
    nwrite = write(STDOUT_FILENO, ptr, ntowrite);
    fprintf(stderr, "nwrite = %d, errno = %d, err_message = '%s'\n", nwrite, errno, strerror(errno));
    if (nwrite > 0) {
      ptr += nwrite;
      ntowrite -= nwrite;
    }
  }

  clr_fl(STDOUT_FILENO, O_NONBLOCK);
  return 0;
}

void set_fl(int fd, int flags) {
  int val;

  if ((val = fcntl(fd, F_GETFL, 0)) < 0) {
    fprintf(stderr, "fcntl F_GETFL error");
    exit(1);
  }

  val |= flags; /* turn on flags */

  if (fcntl(fd, F_SETFL, val) < 0) {
    fprintf(stderr, "fcntl F_SETFL error");
    exit(1);
  }
}

void clr_fl(int fd, int flags) {
  int val;

  if ((val = fcntl(fd, F_GETFL, 0) < 0) {
    fprintf(stderr, "fcntl F_GETFL error");
    exit(1);
  }

  val &= ~flags; /* turn flags off */

  if (fcntl(fd, F_SETFL, val) < 0) {
    fprintf(stderr, "fcntl F_SETFL error");
    exit(1);
  }
}
```

In this example the program issues more than 9,000 `write` calls, even though only 500 are needed to output the data. The rest just return an error. This type of loop, called _polling_, is a waste of CPU time on a multiuser system. We will see that IO multiplexing with nonblocking descriptor is a more efficient way to do this.

Sometimes, we can avoid using nonblocking IO by designing our apps to use multiple threads. We can allow individual threads to block in IO calls if we can continue to make progress in other threads. This can sometimes  simplify the design; at other times, however, the overhead of synchronization can add more complexity than is saved from using threads.

## Record Locking

What happens when two people edit the same  file at the same time? In most UNIX systems, the final state of the file corresponds to the last process that wrote the file. In some applications, however, such a database system, a process needs to be certain that it alone is writing to a file. To provide this capability for processes that need it, Unix provides record locking.

_Record locking_ is the term normally used to describe the ability to a process to prevent other processes from modifying a region of a file while the first process is reading or modifying that portion of the file. "Record" really means byte range as the unix kernel does not have the notion of records in a file. Hence a better term would be _byte-range locking_, given that it is a range of a file (possibly entire file) that is locked. 

### `fcntl` Record Locking

Let us start with providing the `fnctl` declaration:

```cpp
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* struct flock *flockptr */ );
```
Return: depends on _cmd_ if OK, -1 on error

For record locking, _cmd_ is `F_GETLK`, `F_SETLK`, or `F_SETLKW`. The third argument _flockptr_ is a pointer to an `flock` structure:

```cpp
struct flock {
  short l_type;   /* F_RDLCK, F_WRLCK, or F_UNLCK */
  short l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END */
  off_t l_start;  /* offset in bytes, relative to l_whence */
  off_t l_len;    /* length, in bytes; 0 means lock to EOF */
  pid_t l_pid;    /* returned with F_GETLK */
}
```

This structure describes 

* the type of lock desired: `F_RDLCK` (a shared read lock), `F_WRLCK` (an exclusive write lock), or `F_UNLCK` (unlocking a region)
* the starting byte offset of the region being locked or unlocked (`l_start` and `l_whence`)
* the size of the region in nytes (`l_len`)
* the ID (`l_pid`) of the process holding the lock that can block the current process (returned by `F_GETLK` only)

Numerous rules apply to the specification of the region to be locked or unlocked.

* The two elements that specify the starting offset of the region are similar to the last two arguments of the `lseek` function. Indeed the `l_whence` member is specified as `SEEK_SET`, `SEEK_CUR`, or `SEEK_END`. 
* Locks can start and extend beyond the current end of file, but cannot start or extend before the beginning of the file.
* If `l_len` is 0, it means that the lock extends to the largest possible offset of the file. This allows us to lock a region starting anywhere in the file, up through and including any data that is appended to the file. (We do not have to try to guess how many bytes might be appended to the file)
* To lock the entire file, we set `l_start` and `l_whence` to point to the beginning of the file and specify a length (`l_len`) of 0. (There are several ways to specify the beginning of the file, but most applications specify `l_start` as 0 and `l_whence` as `SEEK_SET`).

We previously mentioned two types of locks: a shared read lock (`l_type` of `F_RDLCK`) and exclusive write lock (`F_WRLCK`). The basic rule is that any number of processes can have a shared read lock on a given byte, but only one process can have an exclusive write lock on a given byte, but only one process can have an exclusive write lock on a given byte. Furthermore, if there are one or more read locks on a byte, there can't be any write locks on that byte; if there is an exclusive write lock on a byte, there can't be any read locks on that byte. This formulation will be denoted as _compatibility rule_ for different lock types. 

The compatibility rule applies to lock requests made from different processes, not to multiple lock requests made by a single process. If a process has an existing lock on a range of a file, a subsequent attempt to place a lock on the same range by the same process will replace the existing lock with a new one. Thus, if a process has a write lock on bytes 16-32 of a file and then tries to place a read lock on bytes 16-32, the request will succeed, and the write lock will be replaced by a read lock.
To obtain a read lock, the descriptor must be open for reading; to obtain a write lock, the descriptor must be open for writing.

Details on the commands for the `fcntl` function:

`F_GETLK`  Determine whether the lock described by _flockptr_ is blocked by some other lock. If a lock exists that would prevent ours from being created, the information on that existing lock overwrites the information pointed to by _flockptr_. If no lock exists that would prevent ours from being created, the structure pointed to by _flockptr_ is left unchanged except for the `l_type` member, which is set to `F_UNLCK`.

`F_SETLK`  Set the lock described by _flockptr_. If we are trying to obtain a read lock (`l_type` of `F_RDLCK`) or a write lock (`l_type` of `F_WRLCK`) and the compatibility rule prevents the system from giving us the lock , `fcntl` returns immediately with `errno` set to either `EACCES` or `EAGAIN`. This command is also used to clear the lock described by _flockptr_ (`l_type` of `F_UNLCK`).

`F_SETLKW` This command is a blocking version of `F_SETLK`. (The `W` in the command name mean _wait_). If the requested read lock or write lock cannot be granted because another process currently has some part of the requested region locked, the calling process is put to sleep. The process wakes up either when the lock becomes available or when interrupted by a signal.

Be aware that testing for a lock with `F_GETLK` and then trying to obtain that lock with `F_SETLK` or `F_SETLKW` is not an atomic operation. We have no guarantee that, between the two `fcntl` calls, some other process won't come in and obtain the same lock. If we do not want to block while waiting for a lock to become  available to us, we must handle the possible error returns from `F_SETLK`. 

Note that POSIX.1 does not specify what happens when one process read locks a range of a file, a second process blocks while trying to get a write lock on the same range, and a third process then attempts to get another read lock on the range. If the third process is allowed to place a read lock on the range just because the range is already locked, then the implementation might starve processes with pending write locks. Thus, as additional requests to read lock the same range arrive, the time that the process with the pending write-lock has to wait is extended. If the read-lock requests arrive quickly enough without a lull in the arrival rate, then the writer could wait for a long time.

When setting or releasing a lock on a file , the system combines or splits adjacent areas as required. For example, if we lock bytes 100 through 199 and then unlock byte 150, the kernel still maintains the locks on bytes 100 through 149 and bytes 151 through 199. The schematic below illustrates the byte-range locks in this situation:

```
 ----------- ---------------------------------------------------------------- ------------
|           |                         locked range                           |            |
 ----------- ---------------------------------------------------------------- ------------
           |                                                                 |
          100                                                               199
```
File after locking bytes 100 through 199

```
 ---------- ------------------------- ---   --- ------------------------- --- ------------
|          |   first locked range    |   | |   |   second locked range   |   |            |
 ---------- ------------------------- ---   --- ------------------------- --- ------------
           |                         |         |                             |
          100                       149       151                           199
```
File after unlocking byte 150

If we were to lock byte 150, the system would coalesce the adjacent locked regions into a single region from byte 100 through 199. The resulting picture would be the first diagram above, the same as we started.

#### Example - Requesting and Releasing a Lock

To save ourselves from having to allocate an `flock` structure and fill in all the elements each time, the function `lock_reg` shown below handles all these details:

```cpp
#include <fcntl.h>

int lock_reg(int fd, int cmd, int type, off_t offset, int whence, off_t len)
{
   struct flock  lock;

   lock.l_type = type;      /* F_RDLCK, F_WRLCK, F_UNLCK */
   lock.l_start = offset;   /* byte offset, relative to l_whence */
   lock.l_whence = whence;  /* SEEK_SET, SEEK_CUR, SEEK_END */
   lock.l_len = len;        /* #bytes (0 means to EOF) */

   return (fcntl(fd, cmd, &lock));
}
```

Since most locking calls are to lock or unlock a region (the command `F_GETLK` is rarely used), we normally use one of the following five macros:

```cpp
#define read_lock(fd, offset, whence, len) \
     lock_reg((fd), F_SETLK, F_RDLK, (offset), (whence), (len))
#define readw_lock(fd, offset, whence, len) \
     lock_reg((fd), F_SETLKW, F_RDLCK, (offset), (whence), (len))
#define write_lock(fd, offset, whence, len) \
     lock_reg((fd), F_SETLK, F_WRLCK, (offset), (whence), (len))
#define writew_lock(fd, offset, whence, len) \
     lock_reg((fd), F_SETLKW, F_WRLCK, (offset), (whence), (len))
#define un_lock(fd, offset, whence, len) \
     lock_reg((fd), F_SETLK, F_UNLCK, (offset), (whence), (len))
```

#### Example - Testing for a Lock

The code excerpt below defines a function `lock_test` that we'll use to test for a lock.

Function to test for locking condition
```cpp
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>

pid_t lock_test(int fd, int type, off_t offset, int whence, off_t len)
{
   struct flock lock;

   lock.l_type = type;     /* F_RDLCK or F_WRLCK */
   lock.l_start = offset;  /* byte offset, relative to l_whence */
   lock.l_whence = whence; /* SEEK_SET, SEEK_CUR, SEEK_END */
   lock.l_len = len;       /* #bytes (0 means to EOF) */

   if (fcntl(fd, F_GETLK, &lock) < 0)
      err_sys("fcntl error");

   if (lock.l_type == F_UNLCK)
      return(0);        /* false, region isn't locked by another proc */
   return(lock.l_pid);  /* true, return pid of lock owner */

}
```

If a lock exists that would block the request specified by the arguments, this function returns the process ID of the process holding the lock. Otherwise, the function returns 0 (false). We define two helper macros invoking this function:

```cpp
#define is_read_lockable(fd, offset, whence, len) \
    (lock_test((fd), F_RDLCK, (offset), (whence), (len)) == 0)
#define is_write_lockable(fd, offset, whence, len) \
    (lock_test((fd), F_WRLCK, (offset), (whence), (len)) == 0)
```

Note that the `lock_test` function can't be used by a process to see whether it is currently holding a portion of a file locked. The defintion of the `F_GETLK` command states that the information returned applies to an existing lock that would prevent us from creating our own lock. Since the `F_SETLK` and `F_SETLKW` commands always replace a process's existing lock if it exists, we can never block on our own lock; thus, the `F_GETLK` command will never report our own lock.


#### Example - Deadlock

Deadlock occurs when two processes are each waiting for a resource that the other has locked. The potential for deadlock exists if a process that controls a locked region is put to sleep when it tries to lock another region that is controlled by a different process. 
The code below shows an example of deadlock. The child locks byte 0 and the parent locks byte 1. Each then tries to lock the other's already locked byte. We use parent-child synchronization routines (`TELL_xxx` and `WAIT_xxx`) so that each process can wait for the other to obtain its lock.



# Appendix: useful functions

```cpp
#include <errno.h>
#include <stdarg.h>
/*
 * Print a message and return to caller.
 * Caller specifies "errnoflag".
 */
static void
err_doit(int errnoflag, int error, const char *fmt, va_list ap)
{
  char buf[MAXLINE];
  vsnprintf(buf, MAXLINE, fmt, ap);
  if (errnoflag)
     snprintf(buf+strlen(buf), MAXLINE-strlen(buf), ": %s", strerror(error));
  strcat(buf, "\n");
  fflush(stdout);  /* in case stdout and strerr are the same */
  fputs(buf, stderr);
  fflush(NULL);    /* flushes all stdio output streams */
}

/*
 * Fatal error related to a system call.
 * Print a message and terminate.
 */
void err_sys(const char *fmt, ...)
{
   va_list ap;
   va_start(ap, fmt);
   err_doi(1, errno, fmt, ap);
   va_end(ap);
   exit(1);
}
```

# Useful links

https://notes.shichao.io/apue/ch8/
