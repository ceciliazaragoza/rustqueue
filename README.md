# rustqueue — a bounded FIFO message queue as a Linux kernel module

## Demo

### Build output:

```
$ make clean && make
make -C /lib/modules/7.0.0-14-generic/build M=/home/ubuntu/rustqueue clean
make[1]: Entering directory '/usr/src/linux-headers-7.0.0-14-generic'
make[2]: Entering directory '/home/ubuntu/rustqueue'
make[2]: Leaving directory '/home/ubuntu/rustqueue'
make[1]: Leaving directory '/usr/src/linux-headers-7.0.0-14-generic'
make -C /lib/modules/7.0.0-14-generic/build M=/home/ubuntu/rustqueue modules
make[1]: Entering directory '/usr/src/linux-headers-7.0.0-14-generic'
make[2]: Entering directory '/home/ubuntu/rustqueue'
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: aarch64-linux-gnu-gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0
  You are using:           gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0
warning: pahole version differs from the one used to build the kernel
  The kernel was built with: 131
  You are using:             0
  RUSTC [M] rustqueue.o
  MODPOST Module.symvers
  CC [M]  rustqueue.mod.o
  CC [M]  .module-common.o
  LD [M]  rustqueue.ko
  BTF [M] rustqueue.ko
Skipping BTF generation for rustqueue.ko due to unavailability of vmlinux
make[2]: Leaving directory '/home/ubuntu/rustqueue'
make[1]: Leaving directory '/usr/src/linux-headers-7.0.0-14-generic'
```

### Running `rustqueue`:

#### Demo Part 1: Adding 3 messages to the queue

After adding 3 messages to the queue, we see them returned in a FIFO order. When there are no messages in the queue, there is no output.

```
$ sudo insmod rustqueue.ko
$ ls -la /dev/rustqueue
crw------- 1 root root 10, 263 May  8 03:59 /dev/rustqueue

$ echo "first message"  | sudo tee /dev/rustqueue > /dev/null
$ echo "second message" | sudo tee /dev/rustqueue > /dev/null
$ echo "third message"  | sudo tee /dev/rustqueue > /dev/null

$ sudo cat /dev/rustqueue
first message

$ sudo cat /dev/rustqueue
second message

$ sudo cat /dev/rustqueue
third message

$ sudo cat /dev/rustqueue
# queue is empty. EOF — no output
```

The kernel message buffer shows the `rustqueue` kernel module is inserted with a capacity of 16 messages. As each message is added to the queue, we can see the size in bytes and the size of the queue. As each message is removed or read from the queue, we see it dequeued as well.

```
$ sudo dmesg | tail -10
[ 7623.371990] rustecho: module unloaded
[19815.038829] rustqueue: module loaded
[19859.553943] rustqueue: module unloaded
[37681.663353] rustqueue: module loaded (capacity 16 messages)
[37693.096963] rustqueue: enqueued 14 bytes (1 in queue)
[37699.251100] rustqueue: enqueued 15 bytes (2 in queue)
[37703.514846] rustqueue: enqueued 14 bytes (3 in queue)
[37718.128903] rustqueue: dequeued (2 remaining)
[37722.964870] rustqueue: dequeued (1 remaining)
[37726.655151] rustqueue: dequeued (0 remaining)

$ sudo rmmod rustqueue
```

#### Demo Part 2: Attempt adding 20 messages to the queue

Since the rustqueue can only hold 16 messages, an error is returned after each attempt to add more than 16 messages.

```
$ sudo insmod rustqueue.ko
$ for i in {1..20}; do echo "message $i" | sudo tee /dev/rustqueue > /dev/null || echo "write $i FAILED"; done

/dev/rustqueue: No space left on device (os error 28)
write 17 FAILED
/dev/rustqueue: No space left on device (os error 28)
write 18 FAILED
/dev/rustqueue: No space left on device (os error 28)
write 19 FAILED
/dev/rustqueue: No space left on device (os error 28)
write 20 FAILED
```

The kernel message buffer shows that 16 messages are added to the queue After the queue is full, further writes to the queue are rejected. As each message is added to the queue, we can see the size in bytes and the size of the queue.

```
$ sudo dmesg | tail -20
[37763.989618] rustqueue: enqueued 10 bytes (1 in queue)
[37763.999792] rustqueue: enqueued 10 bytes (2 in queue)
[37764.009722] rustqueue: enqueued 10 bytes (3 in queue)
[37764.020513] rustqueue: enqueued 10 bytes (4 in queue)
[37764.026907] rustqueue: enqueued 10 bytes (5 in queue)
[37764.033613] rustqueue: enqueued 10 bytes (6 in queue)
[37764.039975] rustqueue: enqueued 10 bytes (7 in queue)
[37764.045751] rustqueue: enqueued 10 bytes (8 in queue)
[37764.050658] rustqueue: enqueued 10 bytes (9 in queue)
[37764.055241] rustqueue: enqueued 11 bytes (10 in queue)
[37764.059526] rustqueue: enqueued 11 bytes (11 in queue)
[37764.063808] rustqueue: enqueued 11 bytes (12 in queue)
[37764.068163] rustqueue: enqueued 11 bytes (13 in queue)
[37764.073323] rustqueue: enqueued 11 bytes (14 in queue)
[37764.078751] rustqueue: enqueued 11 bytes (15 in queue)
[37764.084542] rustqueue: enqueued 11 bytes (16 in queue)
[37764.088990] rustqueue: queue full, rejecting write
[37764.093913] rustqueue: queue full, rejecting write
[37764.098698] rustqueue: queue full, rejecting write
[37764.103474] rustqueue: queue full, rejecting write

$ sudo rmmod rustqueue
```

## What this is

rustqueue is a Linux kernel module that exposes a bounded FIFO message queue at /dev/rustqueue, written in safe Rust. It demonstrates how Rust's ownership and locking discipline make a small in-kernel IPC primitive easy to write and structurally free of buffer-handling bugs.

## Build & Run

### Requirements

‼️ Note: Develop inside a disposable VM to avoid issues resulting from kernel bugs, panic, or deadlock.


‼️ Note: Rust-enabled kernel required. Building a Rust kernel module needs a kernel built with `CONFIG_RUST=y`. Ubuntu 26.04 LTS already does this. The exact rustc version matters — using a different one will fail with "the compiler differs from the one used to build the kernel" errors.

- Rust-enabled Linux kernel headers (the C-side build infrastructure)
- The kernel-blessed Rust compiler (rustc-1.93 for Ubuntu 26.04 LTS)
- The matching Rust standard library source and `bindgen` for generating Rust bindings to the kernel’s C headers
- Standard `kmod` utilities for `insmod`, `rmmod`, and `lsmod`

#### Install requirements

```
$ sudo apt update
$ sudo apt install -y build-essential linux-headers-$(uname -r) kmod tree
$ sudo apt install -y rustc-1.93 rust-1.93-src bindgen
$ sudo update-alternatives --install /usr/bin/rustc rustc /usr/bin/rustc-1.93 100
```

Verify requirements installed

```
$ uname -r              # which kernel you're running
$ rustc --version       # should report 1.93.x
$ ls /lib/modules/$(uname -r)/build/rust  # Rust support files for this kernel exist
```

### Build & Run Commands

```
# build file
$ make clean && make
$ sudo insmod rustqueue.ko
$ ls -la /dev/rustqueue

# add messages to the queue
$ echo "first message"  | sudo tee /dev/rustqueue > /dev/null
$ echo "second message" | sudo tee /dev/rustqueue > /dev/null

# remove/read messages from the queue
$ sudo cat /dev/rustqueue
$ sudo cat /dev/rustqueue

# view last 10 messages in kernel message buffer
$ sudo dmesg | tail -10

# unload the module
$ sudo rmmod rustqueue
```

## Code Tour
`rustqueue.rs` is the main workhorse of the project.

### Create global FIFO message queue
Defines a global, thread-safe FIFO message queue for the kernel module. The `global_lock!` protects the queue with a mutex and requires all access to go through `QUEUE.lock()`. The lock is automatically released when the guard goes out of scope.
```
kernel::sync::global_lock! {
    unsafe(uninit) static QUEUE: Mutex<KVec<KVec<u8>>> = KVec::new();
}
```

### Register the device with `init`
Initializes the kernel module when loaded into the Linux. Logs that the module has been loaded, and the module becomes available for use by the device if this is successful.
```
fn init(_module: &'static ThisModule) -> impl PinInit<Self, Error>
```

### Add message to queue with `write_iter`
Checks whether there is space available to add the message, validates the message size, and adds the message to the queue. Then logs the queue activity and returns the number of bytes successfully written.
```
fn write_iter(mut kiocb: Kiocb<'_, Self::Ptr>, iov: &mut IovIterSource<'_>) -> Result<usize>
```

### Read message from queue with `read_iter`
When a file is read for the first time, it removes that message from the shared queue, stores as pending data, and logs the dequeue operation. Then copies the message contents into the buffer and returns the number of bytes read, or 0 if the queue is empty.
```
fn read_iter(mut kiocb: Kiocb<'_, Self::Ptr>, iov: &mut IovIterDest<'_>) -> Result<usize>
```

## Design Notes

- `Mutex<KVec<KVec<u8>>>` vs. a fixed array
  - The queue needs to be a dynamic FIFO buffer where messages can be added and removed at runtime up to the `MAX_MESSAGES` length. This approach has built-in push and pop support and works cleanly with kernel allocation rules. A fixed array would require manual indexing and extra handling for when it is empty or full, making it more complex and less suitable for this use case.
- `KVec` vs. `Vec`
  - `KVec` is designed for kernel space and allows for kernel-safe allocation instead of the standard Rust `Vec`. A standard `Vec` assumes a user-space allocator and runtime guarantees that don't exist in kernel space, so it cannot be used.
- `Mutex` vs. `RwLock`
  - A Mutex is used because the queue is a shared FIFO structure that is frequently mutated with enqueue and dequeue operations. Therefore, need to ensure exclusive access to preserve correctnes. An `RwLock` would not be useful because `read_iter` and `write_iter` modify the queue, so most operations still require locking.
- `const MAX_MESSAGES: usize = 16`
  - The maximum number of messages that the queue can hold which is 16 to ensure this kernel module is memory-safe and prevent unbounded allocations. If the user could keep writing indefinitely, then the queue could grow in size until it uses up all the kernel memory and potentially crash the system.
- `const MAX_MSG_SIZE: usize = 4096`
  - The maximum message size is 4096 bytes, which prevents a single write from consuming too much kernel memory. This prevents kernel memory from being exhausted and the system from crashing.

## Future Work

- Partial reads
  - Currently, each read consumes an entire message at once. If a user's buffer is smaller than the message size, then leave the remaining bytes in the queue instead of discarding or skipping them. This makes the module behave more like a standard streaming character device.
- Per-process queues
  - Currently, all processes share a single global queue. Can give each process or  file descriptor its own independent session by initializing a separate queue with the `open()` handler.
- Configurable capacity
  - Currently, `MAX_MESSAGES` is a hardcoded, constant value. This value could instead be exposed as a `module_param!` so that the user can set the value without recompiling.
- Non-blocking semantics
  - Adding a `poll()` implementation would allow user-space programs to wait for queue activity efficiently using `select()` or `epoll()`. This would eliminate busy-waiting, where a program repeatedly checks for data in a loop instead of sleeping until the kernel notifies it that the queue is ready.
- Persistence
  - Currently, all queued messages are lost when the module is unloaded. Can serialize queue contents to a file on module unload and restore them on load. This would allow the queue to survive module reloads or system restarts.

## License

Licensed GPL-2.0 to match the Linux kernel.
