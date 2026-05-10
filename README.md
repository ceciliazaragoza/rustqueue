# rustqueue — a bounded FIFO message queue as a Linux kernel module

TODO: A demo block, near the top. Either an asciinema recording, a 30-second GIF, a screenshot, or a copy-pasted terminal session. Something visual. A reader who scrolls only the first screen needs to see your module do its thing.

TODO: "What this is" paragraph. What problem does the module solve? What does it teach? Plain English, two or three sentences.

TODO: "Build & run" section. The exact commands, copy-pasteable. Tested. If your project requires a Rust-enabled kernel, say so explicitly with a link to the requirement.

TODO: "Code tour" section. Point readers at the interesting parts. "If you want to read the code, start at init (line 50) — that's where the device gets registered. Then look at write (line 80), which is where the lock is acquired and the queue is mutated." Hiring managers love this; it shows you understand your own code well enough to guide a stranger through it.

TODO: "Why this is interesting" or "Design notes" section. Why did you pick a Mutex over an RwLock? Why AtomicU64 instead of Mutex<u64>? Why a VecDeque instead of a fixed array? These are exactly the questions an interviewer would ask. Answer them in the README; you'll be answering them anyway, and writing them down forces clarity.

TODO: "Future work" section. Honest list of what's not yet there. This is engineering maturity — every real project has known limitations and an interesting roadmap.

TODO: License. GPL-2.0 to match the kernel. Mention it explicitly. (See 3.3.)