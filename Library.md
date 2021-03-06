# Spindump Library

## Introduction

The Spindump software builds on the Spindump Library, a simple but extensibile packet analysis package. The makefile builds a library, libspindump.a that can be linked to a program, used to provide the same statistics as the spindump command does, or even extended to build more advanced functionality. The Spindump command itself is built using this library.

The library can also be used in other ways, e.g., by integrating it to other tools or network devices, to collect some information that is necessary to optimize the device or the network in some fashion, for instance.

![Tool output](https://raw.githubusercontent.com/EricssonResearch/spindump/master/images/architecture2s.jpg)

The library has a number of functions, and can be used to build complex functionality, for instance to send measurement data to a central collection point. But the basics of the library are very simple, feeding packets to a packet analyzer which recognise new sessions and calculates statistics for each session. Additional functions can be called to produce aggregate statistics (such as average RTT values) or to further process or communicate the information output from the analyzer.

## Library Usage Example

The API definition for this library can be found from spindump_analyze.h. But the main function are analyzer initialization, handing a packet to it, and de-initialization. First, you need to initialize the analyzer, like this:

    struct spindump_analyze* analyzer = spindump_analyze_initialize();
    if (analyzer == 0) { /* ... handle error */ }

Then, you probably want to feed packets to the analyzer in some kind of loop. The analyzer needs to know the actual Ethernet message frame  received as an octet string, but also a timestamp of when it was received, the length of the frame, and how much of the message was captured if the frame is not stored in its entirety. Here's an example implementation of a packet reception loop that feeds the analyzer:

    while (... /* some condition */) {
      struct spindump_packet packet;
      memset(&packet,0,sizeof(packet));
      packet.timestamp = ...; /* when did the packet arrive? */
      packet.contents = ...;  /* pointer to the beginning of a packet, starting from Ethernet frame */
      packet.etherlen = ...;  /* length of that entire frame */
      packet.caplen = ...;    /* how much of the frame was captured */
      struct spindump_connection* connection = 0;
      spindump_analyze_process(analyzer,&packet,&connection);
      if (connection != 0) { /* ... look inside connection for statistics, etc */ }
    }

Finally, you need to clean up the resources used by the analyzer. Like this:

    spindump_analyze_uninitialize(analyzer);

That's the basic usage of the analyzer, using the built-in functions of looking at TCP, QUIC, ICMP, UDP, and DNS connections and their roundtrips.

The software does not communicate the results in any fashion at this point; any use of the collected information about connections would be up to the program that calls the analyzer; the information is merely collected in an in-memory data structure about current connections. A simple use of the collected information would be to store the data for later statistical analysis, or to summarize in in some fashion, e.g., by looking at average round-trip times to popular destinations.

The analyzer can be integrated to any equipment or other software quite easily.

But beyond this basic functionality, the analyzer is also extensible. You can register a handler:

    spindump_analyze_register(analyzer,
                              // what event(s) to trigger on (OR the event bits together):
                              spindump_analyze_event_newrightrttmeasurement,
                              // whether this handler is specific for a particular connection or all (0):
                              0,
                              // function to call for this event:
                              myhandler,
                              // pass 0 as private data to myhandler later:
                              0);

This registration registers the function "myhandler" to be called when there's a new RTT measurement. Handlers can be added and removed dynamically, and even be registered for specific connections.

But in the end, when a handler has been registered, if the noted event occurs then a user-specified function gets called. In the case of our new RTT measurement handler, the function to be called can be implemented, for instance, like this:

    void myhandler(struct spindump_analyze* state,
                   void* handlerData,
                   void** handlerConnectionData,
                   spindump_analyze_event event,
                   struct spindump_packet* packet,
                   struct spindump_connection* connection) {
       
       if (connection->type == spindump_connection_transport_quic &&
           event == spindump_analyze_event_newrightrttmeasurement) {
       
           /* ... */
       
       }
       
    }

In the first part of the code above, a handler is registered to be called upon seeing a new RTT measurement being registered. The second part of the code is the implementation of that handler function. In this case, once a measurement has been made, the function "myhandler" is called. The packet that triggered the event (if any) is given by "packet" and the connection it is associated with is "connection". For the connection delete events (as they can come due to timeouts), the packet structure is otherwise empty except for the timestamp (packet->timestamp) of the deletion.

All RTT measurements and other data that may be useful is stored in the connection object. See spindump_connections_struct.h for more information. For instance, the type of the connection (TCP, UDP, QUIC, DNS, ICMP) can be determined by looking at the connection->type field.

The RTT data can be accessed also via the connection object. For instance, in the above "myhandler" function one could print an RTT measurement as follows:

    printf("observed last RTT is %.2f milliseconds",
           connection->rightRTT.lastRTT / 1000.0);

The function "myhandler" gets as argument the original data it passed to spindump_analyze_register (here simply "0").

If the handler function needs to store some information on a per-connection basis, it can also do so by using the "handlerConnectionData" pointer. This is a pointer to variable inside the connection object that is dedicated for this handler and this specific connection.

By setting and reading that variable, the handler could (for instance) store its own calculations, e.g., a smoothed RTT value if that's what the handler is there to do.

Typically, for such usage of connection-specific variables, one would likely use the field for a pointer to some new data structure that holds the necessary data. The void* for the connection-specific data is always initialized to zero upon the creation of a new connection, so this can be used to determine when the data structure needs to be allocated. E.g.,

    struct my_datastructure* perConnectionData =
       (struct my_datastructure*)*handlerConnectionData;
    
    if (perConnectionData == 0) {
      
      perConnectionData = (struct my_datastructure*)malloc(sizeof(struct my_datastructure));
      *handlerConnectionData = (void*)perConnectionData;
      
    }
    
    /* ... use the perConnectionData for whatever purpose is necessary ... */

Note that if the handlers allocate memory on a per-connection basis, they also need to de-allocate it. One way of doing this is to use the deletion events as triggers for that de-allocation.

## Library API

The following is a detailed description of the library functionality.

### API function spindump_analyze_initialize

This function creates an object to represent an analyzer. It allocates memory as needed. It returns a non-NULL object pointer if the creation was successful, and NULL otherwise.

The prototype:

    struct spindump_analyze*
    spindump_analyze_initialize(void);

### API function spindump_analyze_uninitialize

Destroy the analyzer resources and memory object.

The prototype:

    void
    spindump_analyze_uninitialize(struct spindump_analyze* state);

### API function spindump_analyze_registerhandler

This function should be called to register a handler as discussed above. 

The prototype: 

    void
    spindump_analyze_registerhandler(struct spindump_analyze* state,
    				 spindump_analyze_event eventmask,
    				 struct spindump_connection* connection,
    				 spindump_analyze_handler handler,
    				 void* handlerData);

### API function spindump_analyze_unregisterhandler

This function should be called to de-register a handler that was previously registered.

The prototype: 

    void
    spindump_analyze_unregisterhandler(struct spindump_analyze* state,
    				   spindump_analyze_event eventmask,
    				   struct spindump_connection* connection,
    				   spindump_analyze_handler handler,
    				   void* handlerData);

### API handler callback interface

The user's function gets called when the relevant event happens. The interface is:

    void myhandler(struct spindump_analyze* state,
                   void* handlerData,
                   void** handlerConnectionData,
                   spindump_analyze_event event,
                   struct spindump_packet* packet,
                   struct spindump_connection* connection);

Here "myhandler" is the user's function and it will get as parameters the analyzer object, the handler data pointer supplied upon registration, a pointer to a pointer that the handler can use to store some information relating to this handler for the specific connection in question, the event, a pointer to the packet, and a pointer to the connection object.

### API data structure struct spindump_connection

This object represents a single connection observed by the analyzer. The full description of that object needs to be added later, but here are some of the key fields that are relevant:

* connection->type indicates the type of the connection (TCP, ICMP, QUIC, etc)
* connection->creationTime indicates when the first packet for the connection was seen
* connection->packetsFromSide1 counts the number of packets sent from the initiator to the responder 
* connection->packetsFromSide2 counts the number of packets sent from the initiator to the initiator 
* connection->leftRTT is the number of microsends for the RTT part that is between the initiator (client) and the measurement point 
* connection->rightRTT is the number of microsends for the RTT part that is between the responder (server) and the measurement point 

The description is in src/spindump_connections_structs.h and the most relevant pare are reproduced below:

    struct spindump_connection {
      unsigned int id;                                  // sequentially allocated descriptive id for the connection
      enum spindump_connection_type type;               // the type of the connection (tcp, icmp, aggregate, etc)
      enum spindump_connection_state state;             // current state (establishing/established/etc)
      struct timeval creationTime;                      // when did we see the first packet?
      struct timeval latestPacketFromSide1;             // when did we see the last packet from side 1?
      struct timeval latestPacketFromSide2;             // when did we see the last packet from side 2?
      unsigned int packetsFromSide1;                    // packet counts
      unsigned int packetsFromSide2;                    // packet counts
      unsigned int bytesFromSide1;                      // byte counts
      unsigned int bytesFromSide2;                      // byte counts
      unsigned int ect0FromInitiator;                   // ECN ECT(0) counts
      unsigned int ect0FromResponder;                   // ECN ECT(0) counts
      unsigned int ect1FromInitiator;                   // ECN ECT(1) counts
      unsigned int ect1FromResponder;                   // ECN ECT(1) counts
      unsigned int ceFromInitiator;                     // ECN CE counts
      unsigned int ceFromResponder;                     // ECN CE counts
      struct spindump_rtt leftRTT;                      // left-side (side 1) RTT calculations
      struct spindump_rtt rightRTT;                     // right-side (side 2) RTT calculations
      struct spindump_rtt respToInitFullRTT;            // end-to-end RTT calculations observed from responder
      struct spindump_rtt initToRespFullRTT;            // end-to-end RTT calculations observed from initiator
      ...
      union {
    
        struct {
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
          spindump_port side1peerPort;                  // source port for the initial packe
          spindump_port side2peerPort;                  // destination port for the initial packet
          struct spindump_seqtracker side1Seqs;         // when did we see sequence numbers from side1?
          struct spindump_seqtracker side2Seqs;         // when did we see sequence numbers from side2?
    	  ...
        } tcp;
    
        struct {
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
          spindump_port side1peerPort;                  // source port for the initial packe
          spindump_port side2peerPort;                  // destination port for the initial packet
          ...
        } udp;
    
        struct {
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
          spindump_port side1peerPort;                  // source port for the initial packe
          spindump_port side2peerPort;                  // destination port for the initial packet
          struct spindump_messageidtracker side1MIDs;   // when did we see message IDs from side1?
          struct spindump_messageidtracker side2MIDs;   // when did we see message IDs from side2?
          ...
        } dns;
    
        struct {
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
          spindump_port side1peerPort;                  // source port for the initial packe
          spindump_port side2peerPort;                  // destination port for the initial packet
          struct spindump_messageidtracker side1MIDs;   // when did we see message IDs from side1?
          struct spindump_messageidtracker side2MIDs;   // when did we see message IDs from side2?
    	  ...
        } coap;
    
        struct {
          uint32_t version;                             // QUIC version
          struct
          spindump_quic_connectionid peer1ConnectionID; // source connection id of the initial packet
          struct
          spindump_quic_connectionid peer2ConnectionID; // source connection id of the initial response packet
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
          spindump_port side1peerPort;                  // source port for the initial packe
          spindump_port side2peerPort;                  // destination port for the initial packet
          unsigned long initialRightRTT;                // initial packet exchange RTT in us
          unsigned long initialLeftRTT;                 // initial packet exchange RTT in us (only available sometimes)
          ... 
        } quic;
    
        struct {
          spindump_address side1peerAddress;            // source address for the initial packet
          spindump_address side2peerAddress;            // destination address for the initial packet
    	  ... 
        } icmp;
    
        struct {
          spindump_address side1peerAddress;            // address of host on side 1
          spindump_address side2peerAddress;            // address of host on side 2
    	  ... 
        } aggregatehostpair;
    
        struct {
          spindump_address side1peerAddress;            // address of host on side 1
          spindump_network side2Network;                // network address on side 2
    	  ... 
        } aggregatehostnetwork;
    
        struct {
          spindump_network side1Network;                // network address on side 1
          spindump_network side2Network;                // network address on side 2
          ...
        } aggregatenetworknetwork;
    
        struct {
          spindump_address group;                       // multicast group address
    	  ... 
        } aggregatemulticastgroup;
    
      } u;
    
    };

### Events

The currently defined events that can be caught are:

    #define spindump_analyze_event_newconnection		             1
    #define spindump_analyze_event_changeconnection		             2
    #define spindump_analyze_event_connectiondelete		             4
    #define spindump_analyze_event_newleftrttmeasurement	             8
    #define spindump_analyze_event_newrightrttmeasurement	            16
    #define spindump_analyze_event_newinitrespfullrttmeasurement        32
    #define spindump_analyze_event_newrespinitfullrttmeasurement        64
    #define spindump_analyze_event_initiatorspinflip	           128
    #define spindump_analyze_event_responderspinflip	           256
    #define spindump_analyze_event_initiatorspinvalue	           512
    #define spindump_analyze_event_responderspinvalue                 1024
    #define spindump_analyze_event_newpacket                          2048
    #define spindump_analyze_event_firstresponsepacket                4096
    #define spindump_analyze_event_statechange                        8192
    #define spindump_analyze_event_initiatorecnce                    16384
    #define spindump_analyze_event_responderecnce                    32768

These can be mixed together in one handler by ORing them together. The pseudo-event "spindump_analyze_event_alllegal" represents all of the events.

The events are as follows:

#### spindump_analyze_event_newconnection

Called when there's a new connection.

#### spindump_analyze_event_changeconnection

Called when the 5-tuple or other session identifiers of a connection change.

#### spindump_analyze_event_connectiondelete

Called when a connection is terminated explicitly or when it is cleaned up by the analyzer due to lack of activity.

#### spindump_analyze_event_newleftrttmeasurement

Called when there's a new RTT measurement between the measurement point and the initiator (client) of a connection. 

#### spindump_analyze_event_newrightrttmeasurement

Called when there's a new RTT measurement between the measurement point and the responder (server) of a connection. 

#### spindump_analyze_event_newinitrespfullrttmeasurement

Called when there's a new RTT measurement for a QUIC connection, a full roundtrip RTT value is as measured from spin bit flips coming from the initiator (client). 

#### spindump_analyze_event_newrespinitfullrttmeasurement

Called when there's a new RTT measurement for a QUIC connection, a full roundtrip RTT value is as measured from spin bit flips coming from the responder (server).

#### spindump_analyze_event_initiatorspinflip

Called when there's a spin bit flip coming from the responder (server). 

#### spindump_analyze_event_responderspinflip

Called when there's a spin bit flip coming from the initiator (client). 

#### spindump_analyze_event_initiatorspinvalue

Called whenever there's a spin bit value in a QUIC connection from the initiator (client).

#### spindump_analyze_event_responderspinvalue

Called whenever there's a spin bit value in a QUIC connection from the responder (server).

#### spindump_analyze_event_newpacket

Called whenever there's a new packet.

#### spindump_analyze_event_firstresponsepacket

Called when a connection attempt has seen the first response packet from the responder.

#### spindump_analyze_event_statechange

Called when the state of a connection changes, typically from ESTABLISHING to ESTABLISHED. 

#### spindump_analyze_event_initiatorecnce

Called when there's an ECN congestion event from the initiator (client) of a connection. 

#### spindump_analyze_event_responderecnce

Called when there's an ECN congestion event from the responder (server) of a connection. 

### Memory allocation

The library allocates memory as needed using malloc and free, and upon calling the analyzer uninitialization function, no allocated memory remains. Some of the allocation sizes can be changed in the relevant header files or through -D flag settings in the makefiles. For instance, the default number of sequence numbers stored for tracking TCP ACKs and COAP requests is 50, as defined in src/spindump_seq.h:

    #ifndef spindump_seqtracker_nstored
    #define spindump_seqtracker_nstored		50
    #endif

A command line option for the compiler could set this to, say 10:

    -Dspindump_seqtracker_nstored=10

and this would affect the memory consumption for a connection object.

A listing of what parameters can be modified is a currently listed feature request (issue #81), as is some kind of parametrization to allow library user to specify what allocation/free functions to use (issue #82).

