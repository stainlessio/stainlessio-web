+++
title = "Dropping root privileges in rust"
date = 2019-01-07T00:00:01-08:00
tags = ["rust", "tutorial", "simple", "security"]
authors = [ "rhay" ]
+++
When I was writing a `fingerd` daemon in Rust (why? because I could), one thing that took me a little while to figure out was how to drop root privileges after I bound to port 79.

One of the principles of security is to use the least amount of privileges you need in order to accomplish the functionality you need.  In the case of fingerd and most network daemons that run on privileged ports (i.e. ports < 1024), you need to be root long enough to bind the socket to the port, and then you have no need for root.  It's called the **principle of least privilege**.

In the world of C and C++, you have the standard C library which provides `setuid(2)` to change the `uid` of the current running process, so that's what we need to use.  But from the usability standpoint, I wanted users to be able to specify the name of the user to run as, and `setuid(2)` takes a `uid` not a username, so we also need to use `getpwnam(3)` to use the password entry for the user by name.

To accomplish this, I created two functions, and then combined them together to create a third that drops from root to a named user. They are pretty straight forward once you figure out all of the ffi magic. This blog post will hopefully help you avoid having to worry about the ffi magic. I can't decide if I want to turn this into a crate or not, but maybe leave a comment to let me know.

First, let's get the `uid` from the username.

```rust
use std::io::Result;

fn get_uid(username : &str) -> Result<uid_t> {
    let p : &passwd = unsafe {
        let cstr = CString::new(username).expect("Unable to pass username to underlying C library");
        let p = getpwnam(cstr.as_ptr());
        if p.is_null() {
            return Err(Error::from_raw_os_error(ENOENT));
        }
        &*p
    };

    Ok(p.pw_uid)
}
```

So, there is some unsafe code to call `getpwname(3)`. We convert the `&str` username into a `CString` so we can pass it into `getpwnam(3)`.  Then we call the system call, check to make sure it's not null, and then return it as a reference outside of the unsafe block.  Once we have it, then we grab the `pw_uid` which is guaranteed to contain a value if we returned anything.

```rust
fn shed(uid : uid_t) -> Result<()> {
    match unsafe { setuid(uid) } {
        0 => Ok(()),
        -1 => Err(Error::last_os_error()),
        n => unreachable!("setuid returned {}", n)
    }
}
```

Next up, we wrap `setuid(2)` in an unsafe and convert the integer return code into a `std::io::Result`.

```rust
pub fn drop_root(username : &str) -> Result<()> {
    get_uid(username).and_then(shed)
}
```

Lastly, we chain them together via the `.and_then` combinator, and then to use it, we do `drop_root("rhay")?`. I do this immediately after `TcpListener::bind(&addr).unwrap();` in my main function. Since `main()` can now return a `Result`, `?` works great.

I hope this post saved you a little time. When I was working on this, I couldn't find any reference that explicitly called out this common security pattern in Rust.
