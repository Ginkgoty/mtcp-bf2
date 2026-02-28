# Developing Applications with mTCP

## Core Concepts

The mTCP programming model is very similar to Linux `epoll`. The corresponding API mapping is as follows:

| Linux System Call   | mTCP API                      |
| ------------------- | ----------------------------- |
| `socket()`          | `mtcp_socket()`               |
| `bind()`            | `mtcp_bind()`                 |
| `listen()`          | `mtcp_listen()`               |
| `accept()`          | `mtcp_accept()`               |
| `connect()`         | `mtcp_connect()`              |
| `read()` / `recv()` | `mtcp_read()` / `mtcp_recv()` |
| `write()`           | `mtcp_write()`                |
| `close()`           | `mtcp_close()`                |
| `epoll_create()`    | `mtcp_epoll_create()`         |
| `epoll_ctl()`       | `mtcp_epoll_ctl()`            |
| `epoll_wait()`      | `mtcp_epoll_wait()`           |
| `setsockopt()`      | `mtcp_setsockopt()`           |
| `getsockopt()`      | `mtcp_getsockopt()`           |

## Header Files

You only need to include two header files:

```c
#include <mtcp_api.h>    // Core socket APIs
#include <mtcp_epoll.h>  // epoll APIs
```

Optional helper headers (from `include`):

* `cpu.h` — `GetNumCPUs()`, `mtcp_core_affinitize()`
* `netlib.h` — `mystrtol()`, etc.
* `debug.h` — macros such as `TRACE_INFO`, `TRACE_ERROR`, etc.

## Minimal Application Skeleton

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <signal.h>
#include <mtcp_api.h>
#include <mtcp_epoll.h>

#define MAX_EVENTS 1024

static int done = 0;

void sig_handler(int signum) { done = 1; }

void *worker(void *arg) {
    int core = *(int *)arg;
    
    // 1. Pin the thread to a CPU core
    mtcp_core_affinitize(core);
    
    // 2. Create an mTCP context (one per thread)
    mctx_t mctx = mtcp_create_context(core);
    if (!mctx) return NULL;
    
    // 3. Create epoll instance
    int ep = mtcp_epoll_create(mctx, MAX_EVENTS);
    
    // 4. Create socket, bind, and listen
    int listener = mtcp_socket(mctx, AF_INET, SOCK_STREAM, 0);
    mtcp_setsock_nonblock(mctx, listener);
    
    struct sockaddr_in saddr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(8080),
    };
    mtcp_bind(mctx, listener, (struct sockaddr *)&saddr, sizeof(saddr));
    mtcp_listen(mctx, listener, 4096);
    
    struct mtcp_epoll_event ev = { .events = MTCP_EPOLLIN, .data.sockid = listener };
    mtcp_epoll_ctl(mctx, ep, MTCP_EPOLL_CTL_ADD, listener, &ev);
    
    // 5. Event loop
    struct mtcp_epoll_event events[MAX_EVENTS];
    while (!done) {
        int n = mtcp_epoll_wait(mctx, ep, events, MAX_EVENTS, -1);
        for (int i = 0; i < n; i++) {
            if (events[i].data.sockid == listener) {
                int c = mtcp_accept(mctx, listener, NULL, NULL);
                if (c >= 0) {
                    mtcp_setsock_nonblock(mctx, c);
                    ev.events = MTCP_EPOLLIN;
                    ev.data.sockid = c;
                    mtcp_epoll_ctl(mctx, ep, MTCP_EPOLL_CTL_ADD, c, &ev);
                }
            } else if (events[i].events & MTCP_EPOLLIN) {
                char buf[2048];
                int rd = mtcp_read(mctx, events[i].data.sockid, buf, sizeof(buf));
                if (rd <= 0) {
                    mtcp_epoll_ctl(mctx, ep, MTCP_EPOLL_CTL_DEL, events[i].data.sockid, NULL);
                    mtcp_close(mctx, events[i].data.sockid);
                } else {
                    // Process data...
                    mtcp_write(mctx, events[i].data.sockid, buf, rd); // echo
                }
            }
        }
    }
    
    mtcp_destroy_context(mctx);
    return NULL;
}

int main(int argc, char **argv) {
    // Initialize mTCP (pass the configuration file path)
    if (mtcp_init("my_app.conf")) {
        fprintf(stderr, "mtcp_init failed\n");
        return 1;
    }
    
    mtcp_register_signal(SIGINT, sig_handler);
    
    struct mtcp_conf mcfg;
    mtcp_getconf(&mcfg);
    mcfg.num_cores = 4;       // Use 4 cores
    mcfg.max_concurrency = 10000;
    mtcp_setconf(&mcfg);
    
    int cores[4];
    pthread_t threads[4];
    for (int i = 0; i < 4; i++) {
        cores[i] = i;
        pthread_create(&threads[i], NULL, worker, &cores[i]);
    }
    for (int i = 0; i < 4; i++)
        pthread_join(threads[i], NULL);
    
    mtcp_destroy();
    return 0;
}
```

## Makefile Template

If you are developing an application **outside** the mTCP source tree, you can use the following Makefile as a reference:

```makefile
# Path configuration
MTCP_DIR = /home/ubuntu/mtcp
DPDK_PKG_CONFIG = /opt/mellanox/dpdk/lib/aarch64-linux-gnu/pkgconfig

CC = gcc
CFLAGS = -g -O3 -Wall -Werror -fgnu89-inline

# Include paths
INC  = -I$(MTCP_DIR)/mtcp/include
INC += -I$(MTCP_DIR)/util/include

# DPDK flags (via pkg-config)
DPDK_CFLAGS  = $(shell PKG_CONFIG_PATH=$(DPDK_PKG_CONFIG) pkg-config --cflags libdpdk)
DPDK_LDFLAGS = $(shell PKG_CONFIG_PATH=$(DPDK_PKG_CONFIG) pkg-config --libs libdpdk)

CFLAGS += $(DPDK_CFLAGS)

# Link order matters
LIBS  = $(MTCP_DIR)/mtcp/lib/libmtcp.a
LIBS += $(MTCP_DIR)/util/http_parsing.o    # If HTTP parsing is required
LIBS += $(MTCP_DIR)/util/netlib.o          # If netlib is required
LIBS += -lpthread -lnuma -lrt -ldl -lgmp
LIBS += $(DPDK_LDFLAGS)

# Target
TARGET = my_app

all: $(TARGET)

$(TARGET): my_app.c
	$(CC) $(CFLAGS) $(INC) -o $@ $< $(LIBS)

clean:
	rm -f $(TARGET) *.o
```

## Key Notes and Caveats

1. **One `mctx_t` per thread:** `mtcp_create_context(core)` must be called from the thread bound to the corresponding core. Contexts must not be shared across threads.
2. **Call `mtcp_init()` before creating threads**, and call `mtcp_destroy()` only after all threads have been joined.
3. **The configuration file** (e.g., `epserver-sf.conf`) `num_cores` setting determines the maximum number of mTCP contexts that can be created.
4. **All I/O is non-blocking** and must be driven by `epoll`-style events.
5. **Do not mix** Linux native sockets with mTCP sockets — mTCP provides its own independent network stack.
6. **During linking,** `libmtcp.a` must appear before the DPDK libraries; otherwise, you may encounter undefined symbol errors.
