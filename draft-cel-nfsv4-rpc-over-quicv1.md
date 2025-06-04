---

title: Remote Procedure Call over QUIC Version 1
abbrev: RPC over QUIC
docname: draft-cel-nfsv4-rpc-over-quicv1-latest
category: std
ipr: trust200902
area: Transport
workgroup: Network File System Version 4
obsoletes:
updates:
stand_alone: yes
submissiontype: IETF
pi: [toc, sortrefs, symrefs, docmapping]

author:

-
  name: Benjamin Coddington
  org: Red Hat
  country: United States of America
  email: bcodding@redhat.com
-
  name: Scott Mayhew
  org: Red Hat
  country: United States of America
  email: smayhew@redhat.com
-
  name: Charles Lever
  role: editor
  org: Oracle Corporation
  abbrev: Oracle
  country: United States of America
  email: chuck.lever@oracle.com

normative:
  RFC1833:
  RFC5056:
  RFC5531:
  RFC5665:
  RFC9000:
  RFC9001:
  RFC9266:
  RFC9289:
  IANA:
    target: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt
    title: IANA Service Name and Transport Protocol Port Number registry

informative:
  RFC0768:
  RFC0793:
  RFC7942:
  RFC8166:
  RFC8881:

--- abstract

This document specifies a protocol for conveying Remote Procedure
(RPC) messages via QUIC version 1 connections. It requires no revision
to application RPC protocols or the RPC protocol itself.

--- note_Note

This note is to be removed before publishing as an RFC.

Discussion of this draft occurs on the [NFSv4 working group mailing
list](nfsv4@ietf.org), archived at
[](https://mailarchive.ietf.org/arch/browse/nfsv4/). Working Group
information is available at
[](https://datatracker.ietf.org/wg/nfsv4/about/).

Submit suggestions and changes as pull requests at
[](https://github.com/chucklever/i-d-rpc-over-quicv1). Instructions
are on that page.

--- middle

# Introduction

The QUIC version 1 protocol is a secure, reliable connection-oriented
network transport described in {{RFC9000}}. Its features include
integrated transport layer security, multiple independent streams over
each connection, fast reconnecting, and advanced packet loss recovery
and congestion avoidance mechanisms.

Open Network Computing Remote Procedure Call (often shortened to "RPC")
is a Remote Procedure Call protocol that runs over a variety of network
transports {{RFC5531}}. RPC implementations so far use UDP {{RFC0768}},
TCP {{RFC0793}}, or RDMA {{RFC8166}}. This document specifies how to
transport RPC messages over QUIC version 1.

## Motivation For a New RPC Transport

Viewed at a moderate distance, RPC over QUIC provides a similar
feature set as RPC over TCP with TLS (as described in {{RFC9289}}).
However, a closer look reveals some essential benefits of using
QUIC transports:

* Even though the QUIC protocol utilizes the same set of encryption
  algorithms as TLSv1.3, the QUIC record protocol encrypts nearly
  the entire transport layer header and authenticates each IP packet.
  Advanced traffic analysis which was possible with TLS on TCP is no
  longer possible. QUIC protects against transport packet spoofing
  and downgrade attacks better than TLS on TCP.

* Because many real IP networks are oversubscribed, packet loss
  due to momentary link or switch saturation continues to be likely
  even on well-maintained data center-quality network fabrics.

  The QUIC protocol utilizes packet loss recovery and congestion
  avoidance features that are lacking in TCP. Because TCP protocol
  design has ossified, it is unlikely to gain these improvements.
  QUIC is more extensible than TCP, meaning future improvements in
  this area can be designed and deployed without application
  disruption.

* Further, because QUIC handles packet loss on a per-stream rather
  than a per-connection basis, spreading RPC traffic across multiple
  streams enables workloads to continue largely unperturbed while
  packet recovery proceeds.

* The QUIC protocol is designed to facilitate secure and automatic
  transit of firewalls. Firewall transparency is a foundational
  feature of NFSv4 (which is built on RPC).

# Requirements Language

{::boilerplate bcp14-tagged}

# RPC-over-QUIC Framework

RPC is first and foremost a message-passing protocol. This section
covers the implementaion details of exchanging RPC messages over
QUIC. Readers should already be familiar with the fundamentals
of ONC RPC {{RFC5531}}.

RPC-over-QUIC relies on QUIC version 1 as the underlying transport
{{RFC9000}}. The use of other QUIC transport versions with RPC MAY
be defined by future specifications.

## Establishing a Connection

When a network host wishes to send RPC requests to a remote
service via QUICv1, it must first find an established QUICv1
connection, or establish a new one.

For the purpose of explanation, this document refers to the
peer that initiates QUICv1 connection establishment as an
"RPC client" peer. This document refers to the peer that
passively accepts an incoming connection request as an
"RPC server" peer.

QUICv1 connections are not completely defined by the classic
5-tuple (IP proto, source address, source port, destination
address, and destination port). Each connection is also defined
by its QUIC connection ID. For instance, if the IP address of
either peer should change, or a NAT/PAT binding and the source
UDP port changes, the receiver can still recognize an ingress
QUICv1 packet as belonging to an established connection.

As a result, due to network conditions or administrative actions,
an RPC-over-QUIC connection can be replaced (a reconnect event)
or migrated (a failover event) without interrupting the operation
of an upper layer protocol such as RPC-over-QUIC. A more complete
discussion can be found in {{Section 9 of RFC9000}}.

### Connection Transport Parameters

When establishing a connection, peers exchange transport
parameters, as described in {{Section 7.4 of RFC9000}}.

#### Initial Flow Control Limits

These limits control the amount of data that each peer may send
on a newly-created stream. The limits are used for flow control
and cap the amount of memory needed by both peers to keep data
flowing on the connection. The value of these limits are typically
based on the bandwidth-delay of the physical link between the
peers, and are not exposed to RPC applications.

#### Number of Streams Per Connection

Each QUICv1 peer may limit the number of streams per
connection; see {{Section 4.6 of RFC9000}}.

Given the definition of RPC message framing in {{sec-framing}},
it is possible for an RPC client to create a stream, send one
RPC Call, receive one RPC Reply, then destroy the stream. That
usage might be common with simple RPC-based protocols like
rpcbind.

For protocols that carry a more intensive workload, this style of
stream allocation generates needless overhead. Moreover, stream
identifiers cannot be re-used on a single QUICv1 connection,
so eventually a QUICv1 connection can no longer create a new
stream for each RPC XID. Finally, a connection peer may
advertise a max_streams value that is significantly lower than
2 ^ 60.

Instead, RPC clients may create enough streams to maximize
opportunities for workload parallism, and should avoid sending
only a few RPCs on each stream.

For example, an RPC client could allocate a handful of streams
per CPU core to reduce contention for the streams and their
associated data structures. Or, an RPC client could create a
set of streams whose count is the same as the number of slots
in an NFSv4.1 session.

Servers that implement RPC-over-QUIC must be mindful that each
additional stream amounts to incremental overhead. RPC servers
MAY deny the creation of new streams if an RPC client already
has many active streams. RPC clients need to be prepared for
that behavior.

#### Maximum Frame Size

This size is the largest QUIC frame that can appear in any
stream on this connection. The QUIC framing protocol is not
visible to the RPC application. The RPC client and server
can therefore negotiate a frame size that enables efficient
transit of RPC traffic with minimal internal memory
fragmentation.

## RPC Service Discovery

For RPC, the destination port is special. RPC services may use
a standardized destination port that is bound to an RPC program
number. Such ports are assigned in the IANA Service Name and
Transport Protocol Port Number registry {{IANA}}.

For example, the rpcbind program, which is RPC program 100000,
listens on port 111. This is done so that RPC clients can always
contact the rpcbind service and discover the other RPC services
that are operating on that network peer.

In other cases, an RPC service might use any available port. The
RPC server registers its port number with the local rpcbind service
so that RPC clients can contact that service.

This mechanism is no different for RPC-over-QUIC than it is for
RPC on other network transports. rpcbind clients specify an RPC
program number and either the "quic" or "quic6" netid when
requesting information about a QUIC-based RPC service. More detail
is available in {{sec-netids}}.

### Transport Layer Security

During connection establishment, the client peer indicates
RPC-over-QUIC support by presenting the ALPN token "sunrpc" in
the TLS handshake. Support for other application-layer protocols
MAY be offered in the same handshake.

As part of establishing a QUICv1 connection, the two connecting
peers authenticate to each other and choose encryption parameters
to establish a confidential channel of communication. All traffic
on one QUICv1 connection is thus bound to the authenticated
identities that were presented during connection establishment.
These peer identities apply to the streams and RPC messages
carried by that connection.

RPC-over-QUIC provides peer authentication and encryption services
using a framework based on Transport Layer Security (TLS). Ergo,
RPC-over-QUIC inherently fulfills many of the requirements of
{{RFC9289}}. The details of QUIC's use of TLS are specified in
{{RFC9001}}. In particular:

- With QUICv1, security at the transport layer is always enabled.
  Therefore, the discussion in {{RFC9289}} about the opportunistic
  use of TLS does not apply to RPC-over-QUIC, and the STARTTLS
  mechanism described in {{Section 4 of RFC9289}} MUST NOT be used
  on RPC-over-QUIC connections.

- The peer authentication requirements in {{Section 5.2 of
  RFC9289}} apply to RPC-over-QUIC.

- The PKIX Extended Key Usage values defined in {{RFC9289}} are
  valid for use with RPC-over-QUIC.

- The ALPN defined in {{Section 8.2 of RFC9289}} is also used
  for RPC-over-QUIC.

## QUIC Streams

RPC-over-QUIC connections are mediated entirely by each peer's
RPC layer and, aside from authentication and connection transport
parameters, are not otherwise visible to RPC applications. An RPC
client establishes an RPC-over-QUIC connection whenever there are
application RPC transactions to be executed.

QUICv1 provides a "stream" abstraction, described in {{Section 2 of
RFC9000}}. A QUICv1 connection carries one or more streams. Once a
QUICv1 connection has been established, either connection peer may
create a stream. Typically, the RPC client peer creates the first
stream on a connection.

Unless explicitly specified, when RPC upper layer protocol
specifications refer to a "connection", for RPC-over-QUIC, this
is a QUIC stream. As an example, an NFSv4.1 BIND_CONN_TO_SESSION
operation {{RFC8881}} binds to a QUICv1 stream. As another example,
to signify the loss of an RPC request, an NFS server closes the
QUICv1 stream that received that request, but it does close not the
encompassing QUICv1 connection.

In terms of TI-RPC semantic labels, a QUICv1 stream behaves as a
"tpi_cots_ord" transport: connection-oriented and in order.

{:aside}
> cel: There is an opportunity here to add a stream that acts
  as a control plane.

## RPC Message Framing {#sec-framing}

RPC-over-QUIC uses only bidirectional streams.

When a connection peer creates a QUICv1 stream, that peer's stream
endpoint is referred to as a "Requester", and MUST emit only RPC
Call messages on that stream.
The other endpoint is referred to as a "Responder", and MUST emit
only RPC Reply messages on that stream.
Receivers MUST silently discard RPC messages whose direction field
does not match its Requester/Responder role.

Requesters and Responders match RPC Calls to RPC Replies using
the XID carried in each RPC message. Responders MUST send RPC
Replies on the same stream on which they received the matching
RPC Call.

Each QUICv1 stream provides reliable in-order delivery of bytes.
However, each stream makes no guarantees about delivery order with
regard to bytes on other streams on the same connection.

The stream data containing RPC records is carried by QUIC STREAM
frames, but this framing is invisible to the RPC layer. The
transport layer buffers and orders received stream data, exposing
only a reliable byte stream to the RPC layer. Although QUIC permits
out-of-order delivery within a stream, RPC-over-QUIC does not make
use of this feature.

Because each QUICv1 stream is an ordered-byte stream, an
RPC-with-QUIC stream carries only a sequence of complete RPC
messages. Although data from multiple streams can be interleaved
on a single QUICv1 connection, RPC messages MUST NOT be interleaved
on one stream.

Just as with RPC on a TCP socket, each RPC message is an ordered
sequence of one or more records on a single stream. Such RPC records
bear no relationship to QUIC stream frames; in fact, stream frames
as defined in {{RFC9000}} are not visible to RPC endpoints.

Each RPC record begins with a four-octet record marker. A record
marker contains the count of octets in the record in its lower 31
bits, and a flag that indicates whether the record is the last
record in the RPC message in the highest order bit. See
{{Section 11 of RFC5531}} for a comparison with TCP record markers.

{:aside}
>bfields: Open question whether we should do something more
 complicated that adds RDMA-like features or at least provides some
 minimal help with data alignment.  Possibilities might include a
 single additional integer giving the offset of a payload, serving
 only as a hint; or reference additional streams in the same
 connection for payloads; or even looking into full RDMA--long-term
 there may be interest in supporting RDMA over QUIC, and we may be
 able to piggyback on that effort.
>
>cel: Direct data placement over TCP can already be accomplished
 today using MPA/DDP protocols (formerly known as iWARP). Using a
 software iWARP implementation means no special hardware is
 necessary. Likewise, if MPA/DDP can be made to support QUIC, much of
 the need for a separate RPC-over-QUIC is moot. In addition, it would
 automatically bring transport layer security to other RDMA-enabled
 protocols (such as RPC-over-RDMA).
>
> cel: I will say however that it should be possible to create a
  "chunk" abstraction (similar to RPC-over-RDMA) where the
  RPC-over-QUIC protocol header identifies octet ranges of an RPC
  message that are to be sent via other QUIC streams. This would
  assume that it is valid for some streams on a QUIC connection to
  carry traffic that is not in the form of an RPC message sequence.
>
> lars: If changes to the RPC-over-QUIC binding might be desired in
  the future, how would they be negotiated/expressed? Should a
  versioned ALPN be used instead of the one from
  {{RFC9289}}?

{:aside}
>NFS requirement on resends: QUIC allows reconnecting using the same
 connection ID, so isn't breaking/reconnection somewhat ambiguous?
 When can a server drop or a client resend? Any advice needed for
 server-side DRC implementations?
>
> lars: I'm not sure I understand what is meant by "reconnecting"
  above. Is this referring to connection migration? Or a 0-RTT
  repeated connection instance? Something else?
>
> lars: Also, I'm not sure if the use of streams is fully specified by
  the above. Is the intent here to leave it to callers to decide if
  they want to use a fresh stream for each RPC, or reuse an existing
  stream for a series of RPCs?
>
> cel: We need to define a server backpressure mechanism akin to the
  TCP window.
>
> cel: Still not clear how RPC program/version discovery will work
  in a world with no endpoint port numbers.
>
> cel: Should we limit each stream to carry only on RPC program and
  version combination? Doing so would delegate demultiplexing of
  ingress RPC traffic to QUIC -- eg, NFSACL and NFS would be required
  to flow over separate streams.

# RPC Authentication Flavors

Streams in a QUIC connection may use different RPC authentication
flavors. One stream might use RPC_AUTH_UNIX, while at the same time,
another might use RPCSEC_GSS.

GSS mutual (peer) authentication occurs only after a QUIC connection
has been established. It is a separate process, and is unchanged by
the use of QUIC. Additionally, authentication of RPCSEC_GSS users is
unchanged by the use of QUIC.

RPCSEC_GSS can optionally perform per-RPC integrity or confidentiality
protection. When operating within a QUIC connection, these GSS services
become largely redundant.  An RPC implementation capable of concurrently
using QUIC and RPCSEC_GSS MUST use Generic Security Service Application
Program Interface (GSS-API) channel binding, as defined in {{RFC5056}},
to determine when an underlying transport already provides a sufficient
degree of confidentiality.

RPC-over-QUIC implementations MUST provide the "tls-exporter" channel
binding type, as defined in {{RFC9266}}.

# Implementation Status

This section is to be removed before publishing as an RFC.

This section records the status of known implementations of the
protocol defined by this specification at the time of posting of this
Internet-Draft, and is based on a proposal described in
{{RFC7942}}. The description of implementations in this section is
intended to assist the IETF in its decision processes in progressing
drafts to RFCs.

Please note that the listing of any individual implementation here
does not imply endorsement by the IETF. Furthermore, no effort has
been spent to verify the information presented here that was supplied
by IETF contributors. This is not intended as, and must not be
construed to be, a catalog of available implementations or their
features. Readers are advised to note that other implementations may
exist.

There are no known implementations of RPC-over-QUIC as
described in this document.

# Security Considerations

Readers should refer to the discussion of QUIC's transport layer
security in {{Section 21 of RFC9000}}.

# IANA Considerations

RFC Editor: In the following subsections, please replace RFC-TBD with
the RFC number assigned to this document. Furthermore, please remove
this Editor's Note before this document is published.

## Netids for RPC-over-QUIC {#sec-netids}

Each new RPC transport is assigned one or more RPC "netid" strings.
These strings are an rpcbind {{RFC1833}} string naming the underlying
transport protocol, appropriate message framing, and the format of
service addresses and ports, among other things.

This document requests that IANA allocate
the following "Netid" registry strings in the "ONC RPC Netid"
registry, as defined in {{RFC5665}}:

~~~
      NC_QUIC    "quic"
      NC_QUIC6   "quic6"
~~~

These netids MUST be used for any transport satisfying the
requirements described in this document. The "quic" netid is
to be used when IPv4 addressing is employed by the underlying
transport, and "quic6" for IPv6 addressing. IANA should use this
document (RFC-TBD) as the reference for the new entries.

{:aside}
> lars: Why one per IP address family? This seems common practice with
  netids, but also seems to be a layering violation?
>
> cel: That question might be out of scope for this document.
  netids very nearly amount to technical debt at this point.

## ALPN Identifier for SunRPC on QUIC

RPC-over-QUIC utilizes the same ALPN string as RPC-with-TLS
does, as defined in {{Section 7.2 of RFC9289}}:

~~~
   Identification Sequence:  0x73 0x75 0x6e 0x72 0x70 0x63 ("sunrpc")
~~~

This document requests that a reference to (RFC-TBD) be added to
the SunRPC protocol entry in the "TLS Application-Layer Protocol
Negotiation (ALPN) Protocol IDs" registry.

--- back

# Acknowledgments
{: numbered="no"}

The authors express their deepest appreciation for the
contributions of J. Bruce Fields who was an original author
of this document.
In addition, we are indebted to Lars Eggert and the QUIC
working group for the creation of the QUIC transport protocol.

The editor is grateful to
Bill Baker,
Greg Marsden,
Richard Scheffenegger,
Martin Thomson,
and
Long Xin
for their input and support.


Special thanks to
Area Director
Gorry Fairhurst,
NFSV4 Working Group Chairs
Brian Pawlowski
and
Christopher Inacio,
and
NFSV4 Working Group Secretary
Thomas Haynes
for their guidance and oversight.
