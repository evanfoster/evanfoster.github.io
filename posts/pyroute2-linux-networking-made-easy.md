<!-- 
.. title: pyroute2 - Linux networking made easy
.. slug: pyroute2-linux-networking-made-easy
.. date: 2016-11-05 16:05:17 UTC-06:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

### What is pyroute2?  
[pyroute2](https://github.com/svinota/pyroute2) is a Python library that uses [Netlink sockets](https://en.wikipedia.org/wiki/Netlink) to interact with the Linux networking stack. It is, in my opinion, the single best way to do anything networking related on Linux machines. Beyond simple things like replacing subprocess calls to `ifconfig` to get and change networking information, it offers a fantastic way to monitor networking related changes (interfaces up and down, new neighbors) and perform actions based upon those. I'll show you how to do that in this post. First, you need to understand some things about Netlink.
### Netlink basics
Netlink is super cool. With it, you can register a program to receive all Netlink messages, no polling required. There are quite a few different messages that Netlink can generate. I like to represent those as an enum in Python:

```python
from enum import Enum
class NetlinkEvents(Enum):
    # A new neighbor has appeared
    RTM_NEWNEIGH = 'RTM_NEWNEIGH'
    # We're no longer watching a certain neighbor
    RTM_DELNEIGH = 'RTM_DELNEIGH'
    # A new network interface has been created
    RTM_NEWLINK = 'RTM_NEWLINK'
    # A network interface has been deleted
    RTM_DELLINK = 'RTM_DELLINK'
    # An IP address has been added to a network interface
    RTM_NEWADDR = 'RTM_NEWADDR'
    # An IP address has been deleted off of a network interface
    RTM_DELADDR = 'RTM_DELADDR'
    # A route has been added to the routing table
    RTM_NEWROUTE = 'RTM_NEWROUTE'
    # A route has been removed from the routing table
    RTM_DELROUTE = 'RTM_DELROUTE'
```
This is not an all inclusive list of possible messages. If you want that, [this is a good place to start.](http://man7.org/linux/man-pages/man7/rtnetlink.7.html)

Now that we have some Netlink events defined, let's put them to use by watching for new addresses. This is where we'll use pyroute2.
### pyroute2 callbacks
First things first, let's import what we need from pyroute2. There are a couple of different ways to use pyroute2, but here I'm going to use IPDB.

```python
from pyroute2 import IPDB
ipdb = IPDB()
# We'll use this later
import pprint
pp = pprint.PrettyPrinter(indent=3)
```
Next, we'll make our callback. This callback needs to have 3 arguments:

```python
def new_address_callback(ipdb, netlink_message, action):
    if action == NetlinkMessages.RTM_NEWADDR.name:
        pp.pprint(netlink_message)
```
The first two arguments are fairly obvious, but the third one is a bit vague. action is basically the type of message, so we can use this to filter out the stuff we don't care about. Now, we'll fire things up!

```python
addr_callback = ipdb.register_callback(new_address_callback)
```
`register_callback` returns the ID of the callback which you'll need to cleanly stop it, so make sure you capture that! To test, we'll pop on over to a shell and add an address to `eth0`.

```bash
[evan@goliath ~]$ sudo ip addr add dev eth0 192.168.2.50/24
```
Back in Python, we should now see something like this:

```python
{  'attrs': [  ('IFA_ADDRESS', '192.168.2.50'),
               ('IFA_LOCAL', '192.168.2.50'),
               ('IFA_LABEL', 'eth0'),
               ('IFA_FLAGS', 129),
               ('IFA_CACHEINFO', {'ifa_valid': 4294967295, 'ifa_prefered': 4294967295, 'tstamp': 3663995, 'cstamp': 3663995})],
   'event': 'RTM_NEWADDR',
   'family': 2,
   'flags': 129,
   'header': {  'error': None,
                'flags': 0,
                'length': 84,
                'pid': 27963,
                'sequence_number': 1478380920,
                'type': 20},
   'index': 2,
   'prefixlen': 24,
   'scope': 0}
```

Once you're finished, you can stop the callback as such:

```python
ipdb.unregister_callback(addr_callback)
```
Nice and simple, right?
### Final notes
I've used pyroute2 both professionally and personally and have found it to be a necessity when dealing with any sort of Linux networking. I've barely scratched the surface of what it can do; [check out the docs if you want to see how powerful it is.](http://docs.pyroute2.org/general.html) It's a really great library and I wish it was used more widely.

If you find any mistakes in this post, please let me know about them by emailing me at the address in the footer. I've simplified things quite a bit here to make this post easier to digest, so if I've cut out something important I'd like to fix it.
