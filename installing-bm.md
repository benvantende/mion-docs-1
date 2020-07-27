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
The following program tags all incoming packets with VLAN 102 and submits them to a specific port. Save the following as `vlan.p4`.

    /* -*- P4_16 -*- */
    #include <core.p4>
    #include <v1model.p4>


    /* Headers */

    typedef bit<9>  egressSpec_t;
    typedef bit<48> macAddr_t;
    typedef bit<32> ip4Addr_t;

    header ethernet_t {
        macAddr_t dstAddr;
        macAddr_t srcAddr;
        bit<16>   etherType;
    }

    header vlan_t {
        bit<3>  pcp; // Prority code point
        bit<1>  dei; // Drop eligibility indicator
        bit<12> vid; // VLAN identifier
        bit<16> etherType;
    }

    struct headers {
        ethernet_t ethernet;
        vlan_t vlan;
    }

    struct metadata {
        /* empty */
    }


    /* Parser */

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

    control MyVerifyChecksum(inout headers hdr, inout metadata meta)
    {
        apply {}
    }


    /* Ingress processing */

    control MyIngress(inout headers hdr,
        inout metadata meta,
        inout standard_metadata_t standard_metadata)
    {
        apply {
            standard_metadata.egress_spec = 2;

            if (hdr.ethernet.isValid()){
                bit<16> tmp_ether_type = hdr.ethernet.etherType;
                hdr.ethernet.etherType = 0x8100;
                hdr.vlan.setValid();
                hdr.vlan.vid = 102;
                hdr.vlan.etherType = tmp_ether_type;
            }
        }
    }


    /* Egress processing */

    control MyEgress(inout headers hdr,
        inout metadata meta,
        inout standard_metadata_t standard_metadata)
    {
        apply {
            if (standard_metadata.egress_port == standard_metadata.ingress_port) {
                mark_to_drop(standard_metadata);
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
            packet.emit(hdr.vlan);
        }
    }

    /* Switch definition */

    V1Switch(
        MyParser(),
        MyVerifyChecksum(),
        MyIngress(),
        MyEgress(),
        MyComputeChecksum(),
        MyDeparser()
    ) main; 


If you copy this file and run `p4c` against it (without arguments), it will be compiled for the BMv2 simple switch. This will produce a `vlan.json` file that can be passed to the simple switch for running.

Virtual Ethernet Ports
----------------------

The behavioral model supplies scripts to setup virtual ethernet ports. They create and teardown a fixed number of ports, each with fixed names.

    $ sudo behavioral-model/tools/veth_setup.sh
    $ sudo behavioral-model/tools/veth_teardown.sh
    
There should be no errors at this stage. To confirm that the ports have been added, you can run `ip link` and look for `veth0` through to `veth17`.

Virtual ethernet ports are created in pairs.


Running BMv2
------------

Make sure simple switch has correct capabilities:

     sudo setcap cap_net_raw+eip `which simple_switch`
     
The `simple_switch` application requires at a minimum the following:

    simple_switch $interfaces $program

An interface is added using the `-i` switch with the argument having the form `${idx}@${name}`. For our example, the complete command line would be:

    simple_switch -i 0@veth1 -i 1@veth3 -i 2@veth5 -i 3@veth7 -i 4@veth9 -i 5@veth11 -i 6@veth13 -i 7@veth15 -i 8@veth17 vlan.json
    

