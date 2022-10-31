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
* To lock the entire file, we set `l_start` and `l_whence` to point to the 

