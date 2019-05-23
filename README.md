netstar
========

This repository contains the netstar framework for building asynchronous network functions.

Installation
------------

* Currently, we have only tested netstar on Ubuntu16.04. So please use Ubuntu16.04 to install netstar.

* Please note, to build netstar, it is recommended that your machine has over 16GB memory. Otherwise the build process may fail.

* Clone this repository with `git clone https://github.com/netstar-project/netstar.git`

* `cd` to the `netstar` folder.

* Update three submodules including dpdk, fmt and c-ares. The three submodules are located at "https://github.com/duanjp8617/dpdk.git", "https://github.com/duanjp8617/fmt.git" and "https://github.com/duanjp8617/c-ares.git" respectively. You can use `git submodule update --init` to update the three submodules.

* Install required dependencies using `sudo ./install-dependencies.sh`.

* Configure netstar with `./configure.py --enable-dpdk --compiler=g++-5 --mode=release`

* Build netstar with `ninja`.

Structure of Source
--------------------

This repository is created based on seastar. The core of netstar is located in the `netstar/` folder. Several netstar applications are located in the `test/netstar/` folder.

* `netstar/asyncflow`: Contain the implementation of async-flow interface. The `netstar/asyncflow/sd_async_flow.hh` contains the implementation of the async-flow manager and the async-flow object.

* `netstar/hookpoint`: Contain the implementation of several hook points.

* `netstar/mica`: Contain the client interface for mica server.

* `netstar/preprocessor`: Contain the implementation of several preprocessors.

* `netstar/stack`: Contain an interface for using the Seastar TCP/IP stack within netstar.

* `port.hh` and `port_manager.hh`: The interface for creating a port abstraction for receiving and sending packets.

* `rte_packet.hh`: A wrapper around the DPDK `rte_mbuf`. This wrapper avoids the memory copy of `rte_mbuf`.

* `shard_container.hh`: Contains the definition of some utility functions.

* Note that we also add some patches to the original seastar source code. You can find out the patch that we created by searching for the `patch by djp` key word in the `net/` folder.

The Seastar Documentation and C++11/14 Tutorial
------------------------------------------------

netstar extensively uses seastar, especially the future/promise implementation provided by seastar. The following materials can be referred to help you understand how netstar works.

* A comprehensive tutorial of seastar, which is available at: "https://github.com/scylladb/seastar/blob/master/doc/tutorial.md"

* netstar and seastar are both implemented using C++11/14. A good tutorial about C+11/14 features can be found in the famous book `Effective Modern C++` by Scott Meyers.

A Tutorial of NetStar
----------------------

We give a tutorial based on a simple packet forwarder implemented using netstar. The source code is in `tests/netstar/simple_forwarder.cc`.

1. The `simple_forwarder.cc` implements a simple packet forwarder, which receives the packet from one port, and send the packet out from another port.

2. Initialize the basic seastar runtime system. This is done with line 132 `return app.run_deprecated(ac, av, [&app] {`.

3. Use the port_manager to add two ports for sending and receiving packets.
```cpp
port_manager::get().add_port(opts, 0, port_type::standard).then([&opts]{
    return port_manager::get().add_port(opts, 1, port_type::standard);
})
```
Line 159-161 add two ports. Port 0 (the first added port) is used to receive packets. Port 1 (the second added port) is used to send packets.

4. Add two hookpoints to each port with 161-165.
```cpp
.then([]{
    return hook_manager::get().add_hook_point(hook_type::sd_async_flow, 0);
}).then([]{
    return hook_manager::get().add_hook_point(hook_type::sd_async_flow, 1);
})
```

5. Start the forwarder in line 166.
```cpp
.then([]{
    return forwarders.start();
})
```
The definition of the forwarders is in line 42-43. forwarders is a special template class `distributed<forwarder>`.

By default, after a seastar program is initialized, the runtime creates one thread for each available CPU core and pins the thread to the corresponding CPU core. Different thread shares no information with each other. THe `distributed` class is a utility template class provided by seastar, which is used to create one instance of the `forwarder` class on each of the thread. The `forwarder` class contains the implementation of implements the core logic of the packet forwarder.

6. Line 167-171 tells the hookpoint to start to work.
```cpp
.then([]{
    return hook_manager::get().invoke_on_all(0, &hook::check_and_start);
}).then([]{
    return hook_manager::get().invoke_on_all(1, &hook::check_and_start);
})
```

7. Line 171-173 tells the forwarder to start to work.
```cpp
.then([]{
    return forwarders.invoke_on_all(&forwarder::run_udp_manager, 1);
})
```

8. Line 173-175 collects data from different `forwarder` instances running on different threads and prints the data on the terminal.

9. The details of the `forwarder` class below are covered below.

10. The constructor of the `forwarder` (line 51-63) configures the `forwarder` instance. It basically tells the hookpoint, which async-flow manager should it send packets to and receive packet from.
```cpp
forwarder() {
    hook_manager::get().hOok(0).send_to_sdaf_manager(_tcp_forward);
    hook_manager::get().hOok(0).receive_from_sdaf_manager(_tcp_reverse);

    hook_manager::get().hOok(1).receive_from_sdaf_manager(_tcp_forward);
    hook_manager::get().hOok(1).send_to_sdaf_manager(_tcp_reverse);

    hook_manager::get().hOok(0).send_to_sdaf_manager(_udp_forward);
    hook_manager::get().hOok(0).receive_from_sdaf_manager(_udp_reverse);

    hook_manager::get().hOok(1).receive_from_sdaf_manager(_udp_forward);
    hook_manager::get().hOok(1).send_to_sdaf_manager(_udp_reverse);
}
```

Here, hook point 0 will send TCP packets to the `_tcp_forward` async-flow manager. hook point 1 will receive TCP packets from the `_tcp_forward` async-flow manager.

In this way, the `_tcp_forward` async-flow manager forwards TCP packets between hook point 0 and hook point 1. Remember that hook point 0 is associated with port 0 and hook point 1 is associated with port 1 (step 4 of this tutorial). This means that the `tcp_forward` processes TCP packets between port 0 and port 1.

11. Line 71-89 implements the core processing logic of the packet forwarder. It is repeated using the `repeat` function call in line 70, so that it is called for each new flow.
```cpp
return _udp_forward.on_new_initial_context().then([this]() mutable {
        auto ic = _udp_forward.get_initial_context();

        do_with(ic.get_sd_async_flow(), [](sd_async_flow<udp_ppr>& ac){
            ac.register_events(udp_events::pkt_in);
            return ac.run_async_loop([&ac](){
                // printf("client async loop runs!\n");
                if(ac.cur_event().on_close_event()) {
                    return make_ready_future<af_action>(af_action::close_forward);
                }
                return make_ready_future<af_action>(af_action::forward);
            });
        }).then([](){
            // printf("client async flow is closed.\n");
        });

        return stop_iteration::no;
    });
});
```


12. In line 71, when a new flow arrives, a flow context is constructed. The continuation function defined in that line will be called to process the flow context.
```cpp
return _udp_forward.on_new_initial_context().then([this]() mutable {
    ...
});
```

13. In line 72, the flow context is retrieved, and an async-flow object is obtained from the flow context using `ic.get_sd_async_flow()`, and a continuation function is defined in line 74 to initialize the flow context.
```cpp
auto ic = _udp_forward.get_initial_context();

do_with(ic.get_sd_async_flow(), [](sd_async_flow<udp_ppr>& ac){
    ...
})
```

14. We only register a `udp_events::pkt_in` for the async-flow object, so that it only processes newly received udp packets. A continuation function is defined on line 76, which is invoked on by each newly received udp packet.
```cpp
ac.register_events(udp_events::pkt_in);
return ac.run_async_loop([&ac](){
    ...
});
```

15. After a packet is received, it is directly forwarded to another port (line 81).
```cpp
[&ac](){
    ...
    return make_ready_future<af_action>(af_action::forward);
}
```

Perform Asynchronous Operations
--------------------------------

Arbitrary asynchronous operations can be performed in continuation function that we mentioned in step 14 of the previous section.
```cpp
ac.register_events(udp_events::pkt_in);
return ac.run_async_loop([&ac](){
    // Arbitrary asynchronous operations can be performed here.
});
```

In `/tests/netstar/playground.cc`, an example program is given, which reads from the mica database once for each receive network packet.

In line 135 of `playground.cc`, a continuation function is defined for the async-flow object to handle the received packet.

In line 143 of `playground.cc`, a read request to mica database is generated.
```cpp
return this->_mc.query(Operation::kGet, sizeof(src_ip), key_buf.get_temp_buffer(),
                       0, temporary_buffer<char>())
```

In line 225-240, the response of the query is handled using the following code:
```cpp
.then_wrapped([&ac, this](auto&& f){
    try{
        f.get();
        return af_action::forward;

    }
    catch(...){
        if(this->_mc.nr_request_descriptors() == 0){
            this->_insufficient_mica_rd_erorr += 1;
        }
        else{
            this->_mica_timeout_error += 1;
        }
        return af_action::drop;
    }
});
```

More asynchronous operations can be added to the continuation function in similar way.

More Examples
--------------

Besides the `simple_forwarder.cc` and `playground.cc`, there are other examples in the `tests/netstar/` folder. The following describes their functionalities.

`https_ids.cc`: It extracts payload from HTTP request and uses `aho-corasick` algorithms for intrusion detection.

`http_request_extraction.cc`: It extracts payload from the HTTP requests.

`microbench_dbrw.cc`: A benchmark program for reading from and writing to a remote mica database.

`tcp_payload_extraction.cc`: It extracts TCP payload from TCP connections.

`tcp_sctp_server.cc`: A test program which accepts TCP connections and echos the received payload.

`two_stack.cc`: In this program, two user-space TCP/IP stacks are created, for different IP addresses.

`with_multiple_hookpoints.cc`: A test program for defining multiple hook points.
