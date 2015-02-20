Quagga FIB Push Interface
=========================
Introduction
------------
This document describes an interface that allows an external component
to learn the forwarding information computed by the Quagga routing suite.

In Quagga, the Routing Information Base (RIB) is maintained in the
'zebra' infrastructure daemon. Routing protocols communicate their best
routes to zebra, and zebra computes the best route across protocols for
each prefix. This latter information makes up the Forwarding Information
Base (FIB). Zebra feeds the FIB to the kernel, which allows the IP stack
in the kernel to forward packets according to the routes computed by
Quagga. The kernel FIB is updated in an OS-specific way. For example,
the 'netlink’ interface is used on Linux, and route sockets are used on
FreeBSD.

The purpose of this work is to better support scenarios where the router
has a forwarding path that is distinct from the kernel, commonly a
hardware-based fast path. In these cases, the FIB needs to be maintained
reliably in the fast path as well. We will refer to the component that
programs the forwarding plane (directly or indirectly) as the Forwarding
Plane Manager or FPM.

As things stand today, the FPM component has to pick up the FIB from the
kernel. For example, on Linux one can use the netlink interface to learn
about route changes. Netlink, however, is not a reliable protocol --
messages can be dropped under stress, which then requires the FPM to do
a full resynchronization with the kernel.

The goals of this work are to:

* Provide a reliable way for the FPM component to obtain the FIB from 
  zebra.
* Move towards providing the FIB in a platform-independent manner. That 
  is, the RIB to FPM interface should eventually be independent of the 
  underlying OS.

Overview
--------
This document proposes a new point-to-point interface from zebra to the
FPM. The communication takes place over a stream socket (TCP). The
connection is initiated by zebra -- that is, the FPM would be the TCP
server.

The relevant zebra code kicks in when zebra is configured with the
`--enable-fpm` flag. Zebra periodically attempts to connect to the FPM
port. Once the connection is up, zebra starts sending messages
containing routes over the socket to the FPM.

In the first cut, a route message comprises of a short 'FPM header'
followed by a netlink message. The code may, therefore, work only on
Linux at the moment, but it will allow quick conversion of existing FPM
components to the new interface. In the future, we plan to move to a
platform-independent format. The netlink-based encoding will continue be
supported as a transition mechanism as long as it is required.

Zebra sends a complete copy of the IP forwarding table to the FPM,
including routes that it may have picked up from the kernel.

The existing interaction of zebra with the kernel remains unchanged --
that is, the kernel continues to receive FIB updates as before.

Message format
--------------
All messages exchanged with the FPM have the following header:

    typedef struct fpm_msg_hdr_t_
    {
      /*
       * Protocol version.
       */
      uint8_t version;

      /*
       * Type of message, see below.
       */
      uint8_t msg_type;

      /*
       * Length of entire message, including the header, in network
       * byte order. 
       */
      uint16_t msg_len;
    } fpm_msg_hdr_t;


The current protocol version is 1, which only defines one message 
type (below).

    typedef enum fpm_msg_type_e_ {
      FPM_MSG_TYPE_NONE = 0,

      /*
       * Indicates that the payload is a completely formed netlink
       * message.
       */
      FPM_MSG_TYPE_NETLINK = 1,
    } fpm_msg_type_e;

Data structures
---------------
The zebra daemon is single-threaded. It currently sends FIB updates to
the kernel synchronously (`rib_install_kernel`) as it is processing the
rib work queue (which contains changed routes).

The FPM will be a user-space process, and zebra communicates with it
over a stream socket. The socket could be back-pressured -- zebra needs
to write to it asynchronously, whenever writes are possible. The RIB
thus needs to efficiently keep track of the set of prefixes that need to
be sent to the FPM.

The existing zebra code did not have a per-prefix structure -- the
`info` pointer of the `struct route_node` pointed to a list of routes
(`struct rib`). This change introduces a new per-prefix structure
(`rib_dest_t`) in zebra, which becomes the `info` pointer for
route_nodes. The `rib_dest_t`, in turn, has a list of `struct rib`s
hanging off of it.

Here are some annotated excerpts from the new code:

        /*
     * Structure that represents a single destination (prefix).
     */
    typedef struct rib_dest_t_ {

      /*
       * Back pointer to the route node for this destination. This
       * helps us get to the prefix that this structure is for.
       */
      struct route_node *rnode;

      /*
       * Doubly-linked list of routes for this prefix.
       */
      struct rib *routes;

      /*
       * Flags, see below.
       */
      u_int32_t flags;

      /*
       * Linkage to put dest on the FPM processing queue.
       */
      TAILQ_ENTRY(rib_dest_t_) fpm_q_entries;

    } rib_dest_t;

    /*
     * This flag indicates that a given prefix has been 'advertised' to
     * the FPM to be installed in the forwarding plane.
     */
    #define RIB_DEST_SENT_TO_FPM   (1 << (ZEBRA_MAX_QINDEX + 1))

    /*
     * Zebra FPM globals.
     */
    typedef struct zfpm_glob_t_ {

    ...

      /*
       * Stream socket to the FPM.
       */
      int sock;

      /*
       * List of rib_dest_t structures to be processed
       */
      TAILQ_HEAD (zfpm_dest_q, rib_dest_t_) dest_q;

    } zfpm_glob_t;

    /*
     * Check if the given rib_dest structure can be garbage collected,
     * and if yes, delete it.
     *
     *  The following must hold true for a rib_dest_t to be garbage 
     *  collected.
     *
     *    - The 'routes' list must be NULL.
     *    - The dest must not be on the FPM dest_q (above).
     */
     extern int rib_gc_dest (struct route_node *rn);

    /*
     * Function that is called once the FPM socket is writeable.
     */
     int zfpm_write_cb (struct thread *thread);

Operation
---------
The `rib_process()` function does the following:

* Selects the best route for a prefix (`rib_dest_t`) and updates the 
  kernel FIB (existing functionality).
* If the best route changed, and the connection to the FPM is up:
    * Enqueues the dest to the FPM `dest_q`.
    * Ensures the `zfpm_write_cb()` function will be called whenever 
      the FPM socket is writable.
 * Calls `rib_gc_dest()` to release the dest if it is no longer required.

When `zfpm_write_cb()` is called it:

* Dequeues the first `rib_dest_t` in FPM `dest_q`.
* Encodes the `rib_dest_t` into a message, and writes the message to 
  the FPM socket out buffer.
* Sets the `RIB_DEST_SENT_TO_FPM` flag if this is a route add, otherwise 
  resets it.
* Calls `rib_gc_dest()` to release the dest if it is no longer required.

The RIB-FPM interface uses replace semantics. That is, if a 'route add'
message for a prefix is followed by another 'route add' message, the
information in the second message is complete by itself, and replaces
the information sent in the first message.

If the connection to the FPM process goes down for some reason (say the
FPM process restarts), then a job is started that walks each prefix in
the RIB and:

* Removes the prefix from the FPM `dest_q`.
* Resets the `RIB_DEST_SENT_TO_FPM` flag.
* Calls `rib_gc_dest()` to release the dest if is no longer required.

When the connection to the FPM comes up, a job is started that walks 
each prefix in the RIB and:

* Enqueues it to the FPM `dest_q`.
* Makes sure that `zfpm_write_cb()` will be called.

User interface
--------------
The following new cli commands were added:

* `debug zebra fpm`
* `show zebra fpm stats`
* `clear zebra fpm stats`

Discussion
----------
### Resynchronization
The RIB-FPM interface defined here is point-to-point and reliable, so we
don't anticipate a scenario where the FPM and the RIB go out of sync.

That said, a crude way to achieve a resync is to flap the RIB-FPM stream
connection. This will cause the RIB to reconnect and send the entire FIB
afresh.

### Message Format Transition
The message format defined by this document depends heavily on netlink.
It may be replaced in the future by something that is
platform-independent and easier to work with.

The switch to any new encoding should be made gracefully. For example,
the RIB-FPM module in zebra could have code for both formats in the
transition period. The format to be used could be selected by a
compile-time or run-time flag.

