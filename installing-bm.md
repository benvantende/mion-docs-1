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
    make
    sudo make install
    
p4c eBPF
--------
 
If this is required, building the backend needs to be done before `cmake ..` is run.
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
    

Compile a P4 Application for BMV
--------------------------------

    /* -*- P4_16 -*- */
    #include <core.p4>
    #include <v1model.p4>

    typedef bit<9>  egressSpec_t;
    typedef bit<48> macAddr_t;
    typedef bit<32> ip4Addr_t;

    header ethernet_t {
        macAddr_t dstAddr;
        macAddr_t srcAddr;
        bit<16>   etherType;
    }

    struct metadata {
    }

    struct headers {
        ethernet_t ethernet;
    }


    struct mac_digest {
        macAddr_t smac;
        egressSpec_t ingress_port;
    }


    /* Parser*/
    
    parser MyParser(packet_in packet,
                    out headers hdr,
                    inout metadata meta,
                    inout standard_metadata_t standard_metadata) {

        state start {
            transition parse_ethernet;
        }

        state parse_ethernet {
            packet.extract(hdr.ethernet);
            transition select(hdr.ethernet.etherType) {
                default : accept;
            }
        }

    }

    /* Checksum verification */

    control MyVerifyChecksum(inout headers hdr, inout metadata meta) {
        apply {  }
    }


    /* Ingress processing */

    control MyIngress(inout headers hdr,
                      inout metadata meta,
                      inout standard_metadata_t standard_metadata) {

        action drop() {
            mark_to_drop(standard_metadata);
        }

        action mac_forward(egressSpec_t port) {
            standard_metadata.egress_spec = port;
        }

        action mac_broadcast() {
            standard_metadata.mcast_grp = 33;
        }

        action noop () { /* Nothing to do */ }

        action send_digest() {
            digest<mac_digest>(1, {
                hdr.ethernet.srcAddr, 
                standard_metadata.ingress_port
            });
        }


        table smac_table {
            key = {
                hdr.ethernet.srcAddr: exact;
            }
            actions = {
                send_digest;
                noop;
            }
            default_action = send_digest();
            size = 4096;
            support_timeout = true;
        }


        table dmac_table {
            key = {
                hdr.ethernet.dstAddr: exact;
            }
            actions = {
                mac_forward;
                drop;
            }
            size = 1024;
            default_action = drop;
        }


        apply {
            if (hdr.ethernet.isValid()) {
                smac_table.apply();

                if (hdr.ethernet.dstAddr == 0xffffffffffff) {
                    mac_broadcast();
                } 
                else {
                    dmac_table.apply();	
                }
            }
        }
    }

    /* Egress processing */

    control MyEgress(inout headers hdr,
                     inout metadata meta,
                     inout standard_metadata_t standard_metadata) {
        action drop() {
            mark_to_drop(standard_metadata);
        }

        apply {
            // Prune multicast packet to ingress port to preventing loop
            if (standard_metadata.egress_port == standard_metadata.ingress_port) {
                drop();
            }
        }
    }


    /* Checksum computation */

    control MyComputeChecksum(inout headers hdr, inout metadata meta) {
        apply { }
    }


    /* Deparser */

    control MyDeparser(packet_out packet, in headers hdr) {
        apply {
            /* We can't use conditionals in a deparser. This means that we are
               unable to conditionally output headers directly. However, if the 
               frame is set as invalid, it won't be outputted, even if emit is
               called
            */
            packet.emit(hdr.ethernet);
        }
    }

    /* Switch pipeline definition */

    V1Switch(
        MyParser(),
        MyVerifyChecksum(),
        MyIngress(),
        MyEgress(),
        MyComputeChecksum(),
        MyDeparser()
    ) main;



Virtual Ethernet Ports
----------------------

The behavioral model supplies scripts to setup virtual ethernet ports. They create and teardown a fixed number of ports, each with fixed names.

    $ sudo behavioral-model/tools/veth_setup.sh
    $ sudo behavioral-model/tools/veth_teardown.sh
    
There should be no errors at this stage. To confirm that the ports have been added, you can run `ip link` and look for `veth0` through to `veth17`.

Virtual ethernet ports are created in pairs.


