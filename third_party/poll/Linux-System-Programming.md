### Multiplexed I/O
#### select()
```c
#include <sys/select.h>

int select (int n,
	        fd_set *readfds,
	        fd_set *writefds,
	        fd_set *exceptfds,
	        struct timeval *timeout);
```
The watched file descriptors are broken into three sets, each waiting for a different event. File descriptors listed in the `readfds` set are watched to see if data is available for reading.

The first parameter, `n`, is equal to the value of the highest-value descriptor in any set, plus one. Consequently, a caller to `select()` is responsible for checking which given file descriptor is the highest-valued and passing in that value plus one for the first parameter.

On successful return, each set is modified such that it contains only the file descriptors that are ready for I/O of the type delineated by that set. For example, assume two file descriptors, with the value 7 and 9, are placed in the `readfds` set. When the call returns, if 7 is still in the set, that file descriptor is ready to read without blocking. If 9 is no longer in the set, it is probably not readable without blocking. (I say *probably* here because it is possible that the data became available after the call completed. In that case, a subsequent call to `select()` will return the file descriptor as ready to read.)

#### poll()
The `poll()` system call is System V's multiplexed I/O solution. It solves several deficiencies in `select()`, although `select()` is still often used (most likely out of habit, or in the name of portability):
```c
#include <poll.h>

int poll (struct pollfd *fds, nfds_t nfds, int timeout);
```
Unlike `select()`, with its inefficient three bit-masked sets of file descriptors, `poll()` employs a single array of `nfds pollfd` structures, pointed to by `fds`. The structure is as follows:
```c
#include <poll.h>

struct pollfd{
	int fd;           /* file descriptor */
	short events;     /* requested events to watch */
	short revents;    /* returned events witnessed */
}
```
In addition, the following events may be returned in the `revents` field:

- POLLER

   Error on the given file descriptor

These events have no meaning in the `events` field and you should not pass them in that field because they are always returned if applicable. With `poll()`, unlike `select()`, you need not be explicitly ask for reporting of exceptions.

**poll() example**

Let's look at an example program that uses `poll()` to simultaneously check whether a read from *stdin* and a write to *stdout* will block:
```c
#include <stdio.h>
#include <unistd.h>
#include <poll.h>

#define TIMEOUT 5

int main(int argc, char const *argv[])
{
	struct pollfd fds[2];
	int ret;

	/* watch stdin for input */
	fds[0].fd = STDIN_FILENO;
	fds[0].events = POLLIN;

	/* watch a stdout for ability to write (almost always true) */
	fds[1].fd = STDOUT_FILENO;
	fds[1].events = POLLOUT;

	/* All set, block! */
	ret = poll (fds, 2, TIMEOUT * 1000);
	if (ret == -1) {
		perror("poll");
		return 1;
	}

	if (!ret) {
		printf("%d seconds elapsed.\n", TIMEOUT);
		return 0;
	}

	if (fds[0].revents & POLLIN)
		printf("stdin is readable\n");

	if (fds[1].revents & POLLOUT)
		printf("stdout is writable\n");

	return 0;
}
```
