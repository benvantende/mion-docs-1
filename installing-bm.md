Installing the p4.org Behavioural Model
=======================================

You can get started with P4 without having access to a physical switch a vendored SDKs. p4.org provides the "Behavioural Model", which is a standalone environment for running P4, and `p4c`, the reference compiler.

Notes:
------

* `sudo setcap cap_net_raw+eip` This capability means that raw sockets can be used and that the application is able to bind to any adddress
* 

Behavioural Model
-----------------

    git clone https://github.com/p4lang/behavioral-model.git
    cd behavioral-model
    ./install_deps.sh
    ./autogen.sh
    ./configure
    make -j `nproc`
    sudo make install
    
    
p4c
---

    sudo apt-get install git cmake g++ flex bison protobuf-compiler libprotobuf-dev libboost-dev libboost-iostreams-dev libgc-dev libgmp-dev libgtest-dev
    git clone https://github.com/p4lang/p4c.git
    cd p4c
    git submodule update --init --recursive
    mkdir build
    cd build
    cmake ..
    
p4c eBPF
--------
 
Install script is python3, so python3-pip will need to be installed. Python 3 is installed by default on 18.04.


    sudo apt-get install python3-pip libelf-dev
    pip3 install scapy
    
    sudo apt-get install llvm
    cd p4c/backends/ebpf
    chmod +x build_ebpf
    ./build_ebpf
    
p4c Graph
---------

    sudo apt-get install libboost-graph-dev

  
p4c other
---------

    pip3 install ipaddr
    sudo apt-get install doxygen graphviz
    
