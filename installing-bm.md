Installing the p4.org Behavioural Model
=======================================

You can get started with P4 without having access to a physical switch a vendored SDKs. p4.org provides the "Behavioural Model" which is a standalone environment for compiling and running P4.

Notes:
------

* `sudo setcap cap_net_raw+eip` This capability means that raw sockets can be used and that the application is able to bind to any adddress
