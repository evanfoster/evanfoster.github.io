<!--
.. title: Easy packet traces using remotecap
.. slug: easy-packet-traces-using-remotecap
.. date: 2018-05-05 17:23:59 UTC-06:00
.. tags: remotecap,tcpdump,capture
.. category: 
.. link: 
.. description: 
.. type: text
-->

I hate taking packet captures. I end up taking a ton of them at home and at work. Frankly, this workflow sucks:  
```bash
# Oh shoot, I need to take a packet capture.
dev-lappy: ssh root@somewhere.something.net
somewhere: tcpdump -i any -s 0 'not port 22' -w /tmp/foo.cap
# In another terminal after making the traffic I want...
dev-lappy: scp root@somewhere.something.net:/tmp/foo.cap captures/something-foo.cap
dev-lappy: wireshark captures/something-foo.cap
```

There's all kinds of garbage that can happen along the way:  
- `/tmp/` (or wherever you're capturing to) could fill up  
- Somebody else could clobber your capture file (that's a fun one)  
- You might forget to copy the new capture file over

And the whole affair just takes *forever*. I want live updates! I don't want to fill up precious disk space on my server! I want my capture file to be on my machine!

Enter `remotecap`, something I built because I was sick of this.

Here's the workflow with remotecap:  
```bash
dev-lappy: remotecap -w captures/something-foo.cap somewhere.something.net
# Screen is cleared and capture size and rate of growth is printed.
# This can be disabled by using -q/--quiet

# In another terminal
dev-lappy: wireshark captures/something-foo.cap
# Now, make the traffic you want to view and simply hit <C-r> in Wireshark
```

`remotecap` will `ssh` into the target server, run `tcpdump`, and then pipe the outback back over `ssh` to a file on your system. That's not all it can do, however!

Here's a more complicated example:
```bash
remotecap -w captures/some-weird-problem --user notroot --sudo \
          --command-path /stupid/path/tcpdump --filter 'port 80 and port 443 and port 1234'\
          --key ~/somerandomkey --packet-length 1234 --known-hosts None\
          10.0.0.5 1.2.3.4:2022 example.org:4561
```

This will run `tcpdump` on three machines simultaneously and put them in a folder named `some-weird-problem` as `$HOST.cap`, e.g. `some-weird-problem/10.0.0.5.cap`. 
It also logs in as a non-root user and then uses `sudo` to escalate privileges on these machines. The other options are pretty self-explanatory for anyone that has used `ssh` or `tcpdump`. My favorite thing is that you can specify the `ssh` listening port per host using `:` to separate the host from the port.

Internally, `remotecap` is built on top of `asyncio`, using `asyncssh` to do `ssh` connections and stream the output, and `aiofiles` to allow file I/O that won't block the event loop. 
Because of this, it performs pretty well. I've been able to take captures on systems that are recieving 500 Mb/s or more.

To use `remotecap`, you need to have `python >= 3.6` on your system. `remotecap` has been tested on Linux and should *probably* work on MacOS. I have no clue if this will work on Windows, so sorry there!

To install, simply do the following:  
```bash
# You could do this with system Python, but that's not really a great idea.
# All file paths below are just examples. You can put this virtualenv wherever you want.
dev-lappy: python3 -m venv ~/remotecap
dev-lappy: source ~/remotecap/bin/activate
# Make sure that you have libsodium installed. Install it using apt/dnf/pacman/zypper
# This also might need a compiler if the wheels don't work.
(remotecap) dev-lappy: pip install remotecap[recommends]
# Bunch of garbage prints
(remotecap) dev-lappy: remotecap --help
```

I may create packages for various distros using `fpm` at some point, but for now, just use `pip`.

If you run into any issues or just want to see the source, head on over to [`remotecap on Github`](https://github.com/evanfoster/remotecap).

Happy trails!
