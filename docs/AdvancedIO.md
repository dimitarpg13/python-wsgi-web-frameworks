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
