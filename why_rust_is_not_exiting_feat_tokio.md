---
title: Why my rust program is not exiting (feat. Tokio)
published_at: 2023-05-01
order: -1
---

# Why my rust program is not exiting (feat. Tokio)

I've been recently working on random [side project](https://github.com/nkitsaini/waper) and wanted to provide a REPL like interface in the program. I was using `tokio` all over my code and decided to use `tokio::io::Stdin` for building this REPL too. Everything was working except my program was not exiting. This is a story about how I figured out why my program was not exiting. 


## The code

The code was roughly like:
```rust
#[tokio::main]
fn main() -> anyhow::Result<()> {
	// ...

	// some other task that I wanted to run in background, while user interacts with repl
    let operation = orchestrator.start(...); 

    let mut cli = cli::Repl::new();

	// don't worry if you don't know this. It's not relevant
    tokio::pin!(operation); 

    loop {
		tokio::select! {
			x = &mut operation => break,
			_ = cli.reader.next_line() => { dbg!("User input") }
		}
	}

	// this is getting printed, but the program does not exit
    println!("I'm exiting"); 
    Ok(())
}
```
And `cli.reader` is of type `Lines<BufReader<Stdin>>` (all from `tokio::io`).

## The issue
Well, the code just prints "I'm exiting" but never exits. If it was some language like Python/JavaScript, I wouldn't have gone further and assumed that the library might be using some magic of the language to somehow block the program exit. But my encounters with rust has shown that rust language and libraries both have very few "Magic" behaviors, which makes this issue interesting. And as far as I knew, there was no construct in rust for a function to say "Well, I'm returning but don't let the caller return."

<aside>
A lot of people might have already got what is happening here, and if you are one of those I don't have anything interesting to show you. The answer screams in my face now that I know it.
</aside>

## The journey
The first step of journey was to determine if I should try to figure the answer out or just use some other way to implement REPL and call it done. Since I'd never looked at any `async` runtime implementation before, I thought debugging this would make me less afraid of what is behind the curtains of `tokio`.

As with any debugging journey, the goal was to remove as much code as possible which can still reproduce the issue.


1. I tried removing my `Orchestrator.start()` with a `tokio::time::sleep()` and the issue was still there, which means that it was not my code somehow keeping the process alive. **It does not exit.**

2. Next magic I could see was the `#[tokio::main]` proc macro. The only that I know about proc macros is that they are very powerful. So I removed the macro and manually created the runtime and made a single thread. **It does not exit.**

3. The `cli.reader` had three type layers `Lines`, `BufReader`, `Stdin`. I removed `Lines` and `BufReader` and the code was still not exiting. **It does not exit.**

4. Replace `Stdin` with `Timeout`. **It exits!**

If I remove `Stdin` the code was exiting as expected. So I was getting somewhere.

At this point code looked like this:
```rust
fn main() {
	tokio::runtime::Builder::new_current_thread()
		.enable_all()
		.build()
		.unwrap().block_on(async {
            let mut stdin = tokio::io::stdin();

            tokio::select! {
	            _ = tokio::time::sleep(Duration::from_secs(1)) => {
                    println!("Timeout")
                },
                _ = stdin.read_u8() => {
                    println!("Got a character")
                }
            };
			println!("Returning from future"); 
            return 1;
           });

	println!("Returning from main function"); 
}
```
The above code prints the following and does not exit:
```
Timeout
Returning from future
Returning from main function
```

Now there are no macros, everything is regular rust function.  `Orchestrator`, `Lines` and `BufReader` are not a suspect anymore.

Next task was to go into `block_on` and see if I can find something that might block the process. I specifically added logs all over in [block_on](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#L290), [exec.block_on](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#LL311C52-L311C60) and some more nested function. 

All of them were returning without any issue. Next I tried replacing `block_on` with a dummy implementation in `tokio` and called my implementation instead of original `block_on`.

```rust
// In https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/runtime.rs
impl Runtime {
	// ...

	pub fn block_on2<F: Future<Output = u8>>(&self, future: F) -> F::Output {
		return 7;
	}
}
```
The code was exiting after this change and I started removing stuff from the [block_on](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#L290) function and reduced it to the point where I knew that the [exec.block_on](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#LL311C52-L311C60) was doing something.

<aside>
At this point, I also copied the tokio source inside my project as I expected to make a lot of changes there, and did not want to pollute the shared tokio crate on my system.
</aside>

Next I tried `rust-gdb` to see if I can find where the program is stuck. I'm not very familiar with all the ways to use gdb, and all I could get was that program was stuck on some `SYSCALL`.

Next I noticed that program exists if I press `ENTER` (i.e. send newline to stdin), which meant that `Stdin` was somehow waiting for input even after the `select` was over. Now this was really strange to me as I knew that in rust `Future` don't do anything unless you explicitly wait on them. And as far as I knew `tokio::select!` just cancels the future which has not completed first.

So somehow the `stdin::read_u8` has a way to keep itself alive even after `tokio::select!` has ended.

Throughout the `tokio` code I was seeing a lot of `tokio::pin!()` calls, so just out of curiosity I went to check the docs. The `pin` was not doing anything special but one word caught my notice in the docs:

Hold you breaths... **DROP**

Of course, it was `Drop` all along. It all made sense.

<aside>
For those who don't know, you can implement Drop trait on any type in rust and execute some code before the type is Dropped by rust.
</aside>

Next thing was to find out which `Drop` implementation was blocking. I checked `Drop` for [Runtime](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#L419) and strangely it was returning fine without blocking. 

Defeated I referred to rust docs regarding [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html). Following part was of interest
```
This destructor consists of two components:

-   A call to `Drop::drop` for that value, if this special `Drop` trait is implemented for its type.
-   The automatically generated “drop glue” which recursively calls the destructors of all the fields of this value.
```
In our case call to `Drop::drop` was completed, so it was one of the [fields of Runtime](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/runtime.rs#L66) struct which was blocking the drop.
```rust
pub struct Runtime {
    scheduler: Scheduler,
    handle: Handle,
    blocking_pool: BlockingPool,
}
```
I searched for `impl Drop for ...` for all three types and found that only `BlockingPool` implements it and as expected the [drop](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/blocking/pool.rs#L276) of `BlockingPool` was not returning. The drop was calling [shutdown](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/blocking/pool.rs#L242) function which was in turn blocked on a `self.shutdown_rx.wait` [call](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/blocking/pool.rs#L261). And the `shutdown_rx.wait()` comments mention that it waits for all the `Sender` to drop.

Now I needed to find where this sender was getting created (of course it was somewhere in `Stdin`) and why was it not dropping.

At this point my assumption was that each task gets a `shutdown_tx`.

After a search for `shutdown_tx` I found that `shutdown_tx` was getting cloned in a `spawn_task` [function](https://github.com/tokio-rs/tokio/blob/f64a1a3dbde9af86dfdf1e9b919a43111f2d1c23/tokio/src/runtime/blocking/pool.rs#L413). I added a log to verify that this code was running, and it was. Next I replaced my `Stdin` with any empty never ending stream like this:

```rust
struct PendingStream {}
impl tokio::io::AsyncRead for PendingStream {
    fn poll_read(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
        buf: &mut io::ReadBuf<'_>,
    ) -> std::task::Poll<std::io::Result<()>> {
        std::task::Poll::Pending
    }
}
```

After replacing `Stdin` I found that the call to the mentioned `spawn_task` was not happening and thus the program was also exiting properly.

So I went to check the `Stdin` implementation and found this comment:
```rust
/// Constructs a new handle to the standard input of the current process.
///
/// This handle is best used for non-interactive uses, such as when a file
/// is piped into the application. For technical reasons, `stdin` is
/// implemented by using an ordinary blocking read on a separate thread, and
/// it is impossible to cancel that read. This can make shutdown of the
/// runtime hang until the user presses enter.
///
/// For interactive uses, it is recommended to spawn a thread dedicated to
/// user input and use blocking IO directly in that thread.
pub fn stdin() -> Stdin {
	let std = io::stdin();
	Stdin {
		std: Blocking::new(std),
	}
}
```
 Only if I would've [RTFM](https://en.wikipedia.org/wiki/RTFM). It clearly mentions that this will block and you should not use it for interactive stuff. Even though a little disappointed that it was me all along, I was glad that I figured the thing out instead of just hoping that it's some bug somewhere and let me just temporarily get around it.

## Ending notes
Even though the final answer is little disappointing, I did find a few things throughout the journey
1. `tokio` allows to create futures which can keep the `tokio::Runtime` alive even when the `Future` is not specifically being `awaited` on.
2. `tokio` is using `oneshot::Reciever/Sender` to detect if all children died. A clever trick. It shares the `Arc<oneshot::Sender>` with all the children. And on the receiver side instead of waiting for data it waits for an `Err` to occur, which signifies that no more `Sender` are alive.
3. Rust drop can sneakily do stuff. And can be used to indefinitely wait. (This I knew, but didn't knew).
4. `tokio::io::Stdin` should not be used for interactive applications directly.
5. `rust-gdb` is easy and is installed by default with most rust installations.
6. `tokio` code is no magic. It works as any other regular crate I might write.











