---
docname: draft-ietf-wish-whep-00
title: WebRTC-HTTP Egress Protocol (WHEP)
abbrev: whip
category: std
ipr: trust200902
area: ART
workgroup: wish

keyword: WebRTC

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:

 -
    ins: S. Murillo
    name: Sergio Garcia Murillo
    organization: Millicast
    email: sergio.garcia.murillo@cosmosoftware.io
    
 -
    ins: C. Chen
    name: Cheng Chen
    organization: ByteDance
    email: webrtc@bytedance.com
    
normative:
  FETCH:
    author:
      org: WHATWG
    title: Fetch - Living Standard
    target: https://fetch.spec.whatwg.org
      
--- abstract

This document describes a simple HTTP-based protocol that will allow WebRTC-based viewers to watch content from streaming services and/or Content Delivery Networks (CDNs) or WebRTC Transmission Network (WTNs).

--- middle

# Introduction

The IETF RTCWEB working group standardized JSEP ({{!RFC8829}}), a mechanism used to control the setup, management, and teardown of a multimedia session. It also describes how to negotiate media flows using the Offer/Answer Model with the Session Description Protocol (SDP) {{!RFC3264}} as well as the formats for data sent over the wire (e.g., media types, codec parameters, and encryption). WebRTC intentionally does not specify a signaling transport protocol at application level. This flexibility has allowed the implementation of a wide range of services. However, those services are typically standalone silos which don't require interoperability with other services or leverage the existence of tools that can communicate with them.

While WebRTC can be integrated with standard signaling protocols like SIP {{?RFC3261}} or XMPP {{?RFC6120}}, they are not designed to be used in broadcasting/streaming services, and there also is no sign of adoption in that industry. RTSP {{?RFC7826}}, which is based on RTP, is not compatible with the SDP offer/answer model {{!RFC3264}}.

There are many situations in which the lack of a standard protocol for consuming media from streaming service using WebRTC has become a problem:
  
- Interoperability between WebRTC services and products.
- Reusing player software which can be integrated easily.
- Integration with Dynamic Adaptive Streaming over HTTP (DASH) for offering live streams via WebRTC while offering a time-shifted version via DASH.
- Playing WebRTC streams on devices that don't support custom javascript to be run (like TVs).

This document mimics what has been done in the WebRTC HTTP Ingest Protocol (WHIP) {{?I-D.draft-ietf-wish-whip}} for ingestion and specifies a simple HTTP-based protocol that can be used for consuming media from a streaming service using WebRTC.

# Terminology

{::boilerplate bcp14-tagged}

# Overview

The WebRTC-HTTP Ingest Protocol (WHIP) is designed to facilitate a one-time exchange of Session Description Protocol (SDP) offers and answers using HTTP POST requests. This exchange is a fundamental step in establishing an Interactive Connectivity Establishment (ICE) and Datagram Transport Layer Security (DTLS) session between WHEP player and the streaming service endpoint (Media Server).

Upon successful establishment of the ICE/DTLS session, unidirectional media data transmission commences from the media server to the WHEP player. It is important to note that SDP renegotiations are not supported in WHEP, meaning that no modifications to the "m=" sections can be made after the initial SDP offer/answer exchange via HTTP POST is completed and only ICE related information can be updated via HTTP PATCH requests as defined in {{ice-support}}.

The following diagram illustrates the core operation of the WHEP protocol for initiating and terminating a viewing session:

~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP player |    | WHEP endpoint | | Media Server | | WHEP session |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |       
    |                         |              |                  |       
    |HTTP POST (SDP Offer)    |              |                  |       
    +------------------------>+              |                  |       
    |201 Created (SDP answer) |              |                  |       
    +<------------------------+              |                  |       
    |          ICE REQUEST                   |                  |       
    +--------------------------------------->+                  |       
    |          ICE RESPONSE                  |                  |       
    |<---------------------------------------+                  |       
    |          DTLS SETUP                    |                  |       
    |<======================================>|                  |       
    |          RTP/RTCP FLOW                 |                  |       
    +<-------------------------------------->+                  |       
    | HTTP DELETE                                               |       
    +---------------------------------------------------------->+       
    | 200 OK                                                    |       
    <-----------------------------------------------------------x       
                                                                               
~~~~~
{: title="WHEP session setup and teardown" #whep-protocol-operation}

The elements in {{whep-protocol-operation}} are described as follows:

- WHEP player: This represents the WebRTC media player, which functions as a client of the WHEP protocol by receiving and decoding the media from a remote media server.
- WHEP endpoint: This denotes the egress server that receives the initial WHEP request.
- WHEP endpoint URL: Refers to the URL of the WHEP endpoint responsible for creating the WHEP session.
- Media server: This is the WebRTC Media Server that establishes the media session with the WHEP player and delivers the media to it.
- WHEP sesion: Indicates the allocated HTTP resource by the WHEP endpoint for an ongoing egress session.
- WHEP session URL: Refers to the URL of the WHEP resource allocated by the WHEP endpoint for a specific media session. The WHEP player can send requests to the WHEP session using this URL to modify the session, such as ICE operations or termination.

# Protocol Operation

In order to set up a streaming session, the WHEP player MUST generate an SDP offer according to the JSEP rules for an initial offer as in {{Section 5.2.1 of !RFC8829}} and perform an HTTP POST request as per {{Section 9.3.3 of !RFC9110}} to the configured WHEP endpoint URL.

The HTTP POST request MUST have a content type of "application/sdp" and contain the SDP offer as the body. The WHEP endpoint MUST generate an SDP answer according to the JSEP rules for an initial answer as in {{Section 5.3.1 of !RFC8829}} and return a "201 Created" response with a content type of "application/sdp", the SDP answer as the body, and a Location header field pointing to the newly created WHEP session.

As the WHEP protocol only supports the playback use case with unidirectional media, the WHEP player SHOULD use "recvonly" attribute in the SDP offer but MAY use the "sendrecv" attribute instead, "inactive" and "recvonly" attributes MUST NOT be used. The WHEP endpoint MUST use "sendonly" attribute in the SDP answer. 

If the HTTP POST to the WHEP endpoint has a content type different than "application/sdp", the WHEP endpoint MUST reject the HTTP POST request with a "415 Unsupported Media Type" error response. If the SDP body is malformed, the WHIP session MUST reject the HTTP POST with a "400 Bad Request" error response. 

Following is an example of an HTTP POST sent from a WHEP player to a WHEP endpoint and the "201 Created" response from the WHIP endpoint containing the Location header pointing to the newly created WHEP session:

~~~~~
POST /whep/endpoint HTTP/1.1
Host: whep.example.com
Content-Type: application/sdp
Content-Length: 1326

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-options:trickle ice2
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtcp-mux-only
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 0 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 201 Created
ETag: "xyzzy"
Content-Type: application/sdp
Content-Length: 1400
Location: https://whep.example.org/sessions/id

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=ice-options:trickle ice2
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
m=video 0 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
~~~~~
{: title="Example of SDP offer/answer exchange done via an HTTP POST"}

The WHEP endpoint COULD require a live publishing to be happening in order to allow a WHEP players to start viewing a stream.
In that case, the WHEP endpoint SHALL return a "409 Conflict" response to the POST request issued by the WHEP player with a "Retry-After" header indicating the number of seconds before sending a new request.
WHEP players MAY periodically try to connect to the WHEP session with exponential backoff period with an initial value of the "Retry-After" header value in the "409 Conflict" response.

Once a session is setup, consent freshness as per {{!RFC7675}} SHALL be used to detect non-graceful disconnection by full ICE implementations and DTLS teardown for session termination by either side.

To explicitly terminate a WHIP session, the WHEP player MUST perform an HTTP DELETE request to the WHEP session URL returned in the Location header field of the initial HTTP POST. Upon receiving the HTTP DELETE request, the WHIP session will be removed and the resources freed on the media server, terminating the ICE and DTLS sessions.

A media server terminating a session MUST follow the procedures in {{Section 5.2 of !RFC7675}}  for immediate revocation of consent.

The WHEP endpoints MUST return an "405 Method Not Allowed" response for any HTTP request method different than OPTIONS and POST on the endpoint URL in order to reserve their usage for future versions of this protocol specification.

The WHEP endpoints MUST support OPTIONS requests for Cross-Origin Resource Sharing (CORS) as defined in {{FETCH}}. The "200 OK" response to any OPTIONS request SHOULD include an "Accept-Post" header with a media type value of "application/sdp" as per {{!W3C.REC-ldp-20150226}}.

The WHEP sessions MUST return an "405 Method Not Allowed" response for any HTTP request method different than PATCH and DELETE on the session URLs in order to reserve their usage for future versions of this protocol specification.

## ICE support {#ice-support}

ICE {{!RFC8845}} is a protocol addressing the complexities of Network Address Translation (NAT) traversal, commonly encountered in Internet communication. NATs hinder direct communication between devices on different local networks, posing challenges for real-time applications. ICE facilitates seamless connectivity by employing techniques to discover and negotiate efficient communication paths. 

Trickle ICE {{!RFC8838}} optimizes the connectivity process by incrementally sharing potential communication paths, reducing latency, and facilitating quicker establishment. 

ICE Restarts are crucial for maintaining connectivity in dynamic network conditions or disruptions, allowing devices to re-establish communication paths without complete renegotiation. This ensures minimal latency and reliable real-time communication.

Trickle ICE and ICE restart support are RECOMMENDED for both WHEP sessions and clients.

### HTTP PATCH request usage {#http-patch-usage}

The WHEP player MAY perform trickle ICE or ICE restarts by sending an HTTP PATCH request as per {{!RFC5789}} to the WHEP session URL, with a body containing a SDP fragment with media type "application/trickle-ice-sdpfrag" as specified in {{!RFC8840}} carrying the relevant ICE information.

If the HTTP PATCH to the WHEP session has a content type different than "application/trickle-ice-sdpfrag", the WHEP session MUST reject the HTTP PATCH request with a "415 Unsupported Media Type" error response. If the SDP fragment is malformed, the WHEP session MUST reject the HTTP PATCH with a "400 Bad Request" error response.

If the WHEP session supports either Trickle ICE or ICE restarts, but not both, it MUST return a "422 Unprocessable Content" response for the HTTP PATCH requests that are not supported as per {{Section 15.5.21 of !RFC9110}}. 

The WHEP player MAY send overlapping HTTP PATCH requests to one WHEP session. Consequently, as those HTTP PATCH requests may be received out-of-order by the WHEP session, if WHEP session supports ICE restarts, it MUST generate a unique strong entity-tag identifying the ICE session as per {{Section 8.8.3 of !RFC9110}}, being OPTIONAL otherwise. The initial value of the entity-tag identifying the initial ICE session MUST be returned in an ETag header field in the "201 Created" response to the initial POST request to the WHEP endpoint.

WHEP players SHOULD NOT use entity-tag validation when matching a specific ICE session is not required, such as for example when initiating a DELETE request to terminate a session. WHEP sessions MUST ignore any entity-tag value sent by the WHEP player when ICE session matching is not required, as in the HTTP DELETE request.

### Trickle ICE {#trickle-ice}

Depending on the Trickle ICE support on the WHEP player, the initial offer by the WHEP player MAY be sent after the full ICE gathering is complete with the full list of ICE candidates, or it MAY only contain local candidates (or even an empty list of candidates) as per {{!RFC8845}}. For the purpose of reducing setup times, when using Trickle ICE the WHEP player SHOULD send the SDP offer as soon as possible, containing either locally gathered ICE candidates or an empty list of candidates.

In order to simplify the protocol, the WHEP session cannot signal additional ICE candidates to the WHEP player after the SDP answer has been sent. The WHEP endpoint SHALL gather all the ICE candidates for the media server before responding to the client request and the SDP answer SHALL contain the full list of ICE candidates of the media server.

As the WHEP player needs to know the WHEP session URL associated with the ICE session in order to send a PATCH request containing new ICE candidates, it MUST wait and buffer any gathered candidates until the "201 Created" HTTP response to the initial POST request is received.
In order to lower the HTTP traffic and processing time required the WHEP player SHOULD send a single aggregated HTTP PATCH request with all the buffered ICE candidates once the response is received.
Additionally, if ICE restarts are supported by the WHEP session, the WHEP player needs to know the entity-tag associated with the ICE session in order to send a PATCH request containing new ICE candidates, so it MUST also wait and buffer any gathered candidates until it receives the HTTP response with the new entity-tag value to the last PATCH request performing an ICE restart.

WHEP players generating the HTTP PATCH body with the SDP fragment and its subsequent processing by WHEP sessions MUST follow to the guidelines defined in {{Section 4.4 of !RFC8840}} with the following considerations:

 - As per {{!RFC8829}}, only m-sections not marked as bundle-only can gather ICE candidates, so given that the "max-bundle" policy is being used, the SDP fragment will contain only the offerer-tagged m-line of the bundle group.
 - The WHEP player MAY exclude ICE candidates from the HTTP PATCH body if they have already been confirmed by the WHEP session with a successful HTTP response to a previous HTTP PATCH request.

If the WHEP session is using entity-tags for identifying the ICE sessions in explained in {{http-patch-usage}}, a WHEP player sending a PATCH request for performing trickle ICE MUST include an "If-Match" header field with the latest known entity-tag as per {{Section 13.1.1 of !RFC9110}}.
When the PATCH request is received by the WHEP session, it MUST compare the indicated entity-tag value with the current entity-tag of the resource as per {{Section 13.1.1 of !RFC9110}} and return a "412 Precondition Failed" response if they do not match. If the HTTP PATCH request does not contain an "If-Match" header the WHEP session MUST return an "428 Precondition Required" response as per {{Section 3 of !RFC6585}}.

When a WHEP session receives a PATCH request that adds new ICE candidates without performing an ICE restart, it MUST return a "204 No Content" response without a body and MUST NOT include an ETag header in the response. If the WHEP session does not support a candidate transport or is not able to resolve the connection address, it MUST silently discard the candidate and continue processing the rest of the request normally.

~~~~~
PATCH /session/id HTTP/1.1
Host: whep.example.com
If-Match: "xyzzy"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 576

a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:EsAw
a=ice-pwd:P2uYro0UCOQ4zxjKXaWCBui1
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.2 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2
a=end-of-candidates

HTTP/1.1 204 No Content
~~~~~
{: title="Example of a Trickle ICE request and response"}

### ICE Restarts {#ice-restarts}

As defined in {{!RFC8839}}, when an ICE restart occurs, a new SDP offer/answer exchange is triggered. However, as WHEP does not support renegotiation of non-ICE related SDP information, a WHEP player will not send a new offer when an ICE restart occurs. Instead, the WHEP player and WHEP session will only exchange the relevant ICE information via an HTTP PATCH request as defined in {{http-patch-usage}} and MUST assume that the previously negotiated non-ICE related SDP information still apply after the ICE restart.

When performing an ICE restart, the WHEP player MUST include the updated "ice-pwd" and "ice-ufrag" in the SDP fragment of the HTTP PATCH request body as well as the new set of gathered ICE candidates as defined in {{!RFC8840}}.
Similar what is defined in {{trickle-ice}}, as per {{!RFC8829}} only m-sections not marked as bundle-only can gather ICE candidates, so given that the "max-bundle" policy is being used, the SDP fragment will contain only the offerer-tagged m-line of the bundle group.
A WHEP player sending a PATCH request for performing ICE restart MUST contain an "If-Match" header field with a field-value "*" as per {{Section 13.1.1 of !RFC9110}}. 

{{!RFC8840}} states that an agent MUST discard any received requests containing "ice-pwd" and "ice-ufrag" attributes that do not match those of the current ICE Negotiation Session, however, any WHEP session receiving an updated "ice-pwd" and "ice-ufrag" attributes MUST consider the request as performing an ICE restart instead and, if supported, SHALL return a "200 OK" with an "application/trickle-ice-sdpfrag" body containing the new ICE username fragment and password and a new set of ICE candidates for the WHEP session. Also, the "200 OK" response for a successful ICE restart MUST contain the new entity-tag corresponding to the new ICE session in an ETag response header field and MAY contain a new set of ICE candidates for the media server. 

As defined in {{Section 4.4.1.1.1 of !RFC8839}} the set of candidates after an ICE restart may include some, none, or all of the previous candidates for that data stream and may include a totally new set of candidates. So after performing a successful ICE restart, both the WHEP player and the WHEP session MUST replace the previous set of remote candidates with the new set exchanged in the HTTP PATCH request and response, discarding any remote ICE candidate not present on the new set. Both the WHEP player and the WHEP session MUST ensure that the HTTP PATCH requests and response bodies include the same 'ice-options,' 'ice-pacing,' and 'ice-lite' attributes as those used in the SDP offer or answer.

If the ICE restart request cannot be satisfied by the WHEP session, the resource MUST return an appropriate HTTP error code and MUST NOT terminate the session immediately and keep the existing ICE session. The WHEP player MAY retry performing a new ICE restart or terminate the session by issuing an HTTP DELETE request instead. In any case, the session MUST be terminated if the ICE consent expires as a consequence of the failed ICE restart as per {{Section 5.1 of !RFC7675}}.

In case of unstable network conditions, the ICE restart HTTP PATCH requests and responses might be received out of order. In order to mitigate this scenario, when the client performs an ICE restart, it MUST discard any previous ICE username and passwords fragments and ignore any further HTTP PATCH response received from a pending HTTP PATCH request. WHEP players MUST apply only the ICE information received in the response to the last sent request. If there is a mismatch between the ICE information at the WHEP player and at the WHEP session (because of an out-of-order request), the STUN requests will contain invalid ICE information and will be dropped by the receiving side. If this situation is detected by the WHEP player, it MUST send a new ICE restart request to the server.

~~~~~
PATCH /session/id HTTP/1.1
Host: whep.example.com
If-Match: "*"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 82

a=ice-options:trickle ice2
a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:ysXw
a=ice-pwd:vw5LmwG4y/e6dPP/zAP9Gp5k
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.2 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2

HTTP/1.1 200 OK
ETag: "abccd"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 252

a=ice-lite
a=ice-options:trickle ice2
a=group:BUNDLE 0 1
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=mid:0
a=ice-ufrag:289b31b754eaa438
a=ice-pwd:0b66f472495ef0ccac7bda653ab6be49ea13114472a5d10a
a=candidate:1 1 udp 2130706431 198.51.100.1 39132 typ host
a=end-of-candidates
~~~~~
{: title="Example of an ICE restart request and response"}

## WebRTC constraints

In order to reduce the complexity of implementing WHEP in both clients and media servers, WHEP imposes the following restrictions regarding WebRTC usage:

### SDP Bundle

Both the WHEP player and the WHEP endpoint SHALL support {{!RFC9143}} and use "max-bundle" policy as defined in {{!RFC8829}}. The WHEP player and the media server MUST support multiplexed media associated with the BUNDLE group as per {{Section 9 of !RFC9143}}. In addition, per {{!RFC9143}} the WHEP player and media server SHALL use RTP/RTCP multiplexing for all bundled media. In order to reduce the network resources required at the media server, both The WHEP player and WHEP endpoints MUST include the "rtcp-mux-only" attribute in each bundled "m=" sections as per {{Section 3 of !RFC8858}}.

### Single MediaStream

WHEP only supports a single MediaStream as defined in {{!RFC8830}} and therefore all "m=" sections MUST contain an "msid" attribute with the same value. The MediaStream MUST contain at least one MediaStreamTrack of any media kind and it MUST NOT have two or more than MediaStreamTracks for the same media (audio or video). However, it would be possible for future revisions of this spec to allow more than a single MediaStream or MediaStreamTrack of each media kind, so in order to ensure forward compatibility, if the number of audio and or video MediaStreamTracks or number of MediaStreams are not supported by the WHEP endpoint, it MUST reject the HTTP POST request with a "406 Not Acceptable" error response.

### Trickle ICE and ICE restarts

The media server SHOULD support full ICE, unless it is connected to the Internet with an IP address that is accessible by each WHEP player that is authorized to use it, in which case it MAY support only ICE lite. The WHEP player MUST implement and use full ICE.

Trickle ICE and ICE restarts support is OPTIONAL for both the WHEP players and media servers as explained in {{ice-support}}.

## Load balancing and redirections

WHEP endpoints and media servers might not be colocated on the same server, so it is possible to load balance incoming requests to different media servers. 

WHEP players SHALL support HTTP redirections as per {{Section 15.4 of !RFC9110}}. In order to avoid POST requests to be redirected as GET requests, status codes 301 and 302 MUST NOT be used and the preferred method for performing load balancing is via the "307 Temporary Redirect" response status code as described in {{Section 15.4.8 of !RFC9110}}. Redirections are not required to be supported for the PATCH and DELETE requests.

In case of high load, the WHEP endpoints MAY return a "503 Service Unavailable" response indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance as described in {{Section 15.6.4 of !RFC9110}}, which will likely be alleviated after some delay. The WHEP endpoint might send a Retry-After header field indicating the minimum time that the user agent ought to wait before making a follow-up request as described in {{Section 10.2.3 of !RFC9110}}.

## STUN/TURN server configuration

The WHEP Endpoint MAY return STUN/TURN server configuration URLs and credentials usable by the client in the "201 Created" response to the HTTP POST request to the WHEP Endpoint URL.

Each STUN/TURN server will be returned using the "Link" header field {{!RFC8288}} with a "rel" attribute value of "ice-server" as specified in {{I-D.draft-ietf-wish-whip}}

It might be also possible to configure the STUN/TURN server URLs with long-term credentials provided by either the broadcasting service or an external TURN provider on the WHEP player, overriding the values provided by the WHEP Endpoint.

## Authentication and authorization

All WHEP endpoints, sessions and clients MUST support HTTP Authentication as per {{Section 11 of !RFC9110}} and in order to ensure interoperability, bearer token authentication as defined in the next section MUST be supported by all WHEP entities. However this does not preclude the support of additional HTTP authentication schemes as defined in {{Section 11.6 of !RFC9110}}.

### Bearer token authentication

WHEP endpoints and sessions MAY require the HTTP request to be authenticated using an HTTP Authorization header field with a Bearer token as specified in {{Section 2.1 of !RFC6750}}. WHEP players MUST implement this authentication and authorization mechanism and send the HTTP Authorization header field in all HTTP requests sent to either the WHEP endpoint or session except the preflight OPTIONS requests for CORS.

The nature, syntax, and semantics of the bearer token, as well as how to distribute it to the client, is outside the scope of this document. Some examples of the kind of tokens that could be used are, but are not limited to, JWT tokens as per {{!RFC6750}} and {{!RFC8725}} or a shared secret stored on a database. The tokens are typically made available to the end user alongside the WHEP endpoint URL and configured on the WHEP players (similar to the way RTMP URLs and Stream Keys are distributed).

WHEP endpoints and sessions could perform the authentication and authorization by encoding an authentication token within the URLs for the WHEP endpoints or sessions instead. In case the WHEP player is not configured to use a bearer token, the HTTP Authorization header field must not be sent in any request.

## Protocol extensions

In order to support future extensions to be defined for the WHEP protocol, a common procedure for registering and announcing the new extensions is defined.

Protocol extensions supported by the WHEP server MUST be advertised to the WHEP player in the "201 Created" response to the initial HTTP POST request sent to the WHEP Endpoint.
The WHEP Endpoint MUST return one "Link" header field for each extension that it supports, with the extension "rel" attribute value containing the extension URN and the URL for the HTTP resource that will be available for receiving requests related to that extension.

Protocol extensions are optional for both WHEP players and WHEP Endpoints and sessions. WHEP players MUST ignore any Link attribute with an unknown "rel" attribute value and WHEP Endpoints and sessions MUST NOT require the usage of any of the extensions.

Each protocol extension MUST register a unique "rel" attribute value at IANA starting with the prefix: "urn:ietf:params:whep:ext" as specified in {{urn-whep-subspace}}.

In the first version of the WHEP specification, two optional extensions are defined: the Server Sent Events and the Video Layer Selection.

### Server Sent Events extension

This optional extesion provides support for server-to-client communication using WHATWG server sent events protocol as specified in https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events. When supported by the WHEP resource, a "Link" header field with a "rel" attribute of "urn:ietf:params:whep:ext:core:server-sent-events" MUST be returned in the initial HTTP "201 Created" response, with the Url of the Server Sent Events REST API entrypoint. The "Link" header field MAY also contain an "events" attribute with a coma separated list of supported event types. 

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whep.example.org/resource/213786HF
Link: <https://whep.ietf.org/resource/213786HF/sse>;
      rel="urn:ietf:params:whep:ext:core:server-sent-events"
      events="active,inactive,layers,viewercount"
~~~~~
{: title="HTTP 201 response example containing the Server Sent Events extension"}

If the extension is also supported by the WHEP player, it MAY send a POST request to the Server Sent Events REST API entrypoint to create a server-to-client event stream using WHATWG server sent events protocol. The POST request MAY contain an "application/json" body with an JSON array indicating the subset of the event list announced by the WHEP Resource on the "events" atribute which COULD be sent by the server using the server-to-client communication channel. The WHEP Endpoint will return a "201 Created" response with a Location header field pointing to the newly created server-to-client event stream.

~~~~~
POST /resource/213786HF/sse HTTP/1.1
Host: whep.example.com
Content-Type: application/sjon

["active","inactive","layers","viewercount"]

HTTP/1.1 201 Created
Location: https://whep.example.org/resource/213786HF/sse/event-stream
~~~~~
{: title="HTTP POST request to create a server-to-client event stream"}

Once the server-to-client communication channel has been created the WHEP player can perform a long pull using the Url returned on the location header as expecified in the WHATWG server sent events protocol.

When an event is generated, the WHEP Resource MUST check for each event stream if the type is on the list provided by the WHEP player when the event stream was created, and if so enque it for delivering when an active long pull request is available.

The events types supported by this specification are the following:

- active: indicating that there is an active publication ongoing for this resource.
- inactive: indicating that there is no active publication ongoing for this resource.
- layers: provides information about the video layers being published for this resource.
- viewercount: provides the number of viewers currently connected to this resource.

The WHEP resource must indicate the event type in the "event" field and a JSON serialized string in the "data" field of the WHATWG server sent events message. In order to make the processing simpler on the WHEP player, the WHEP resource MUST encode the event data in a single "data" line.

~~~~~
event: viewercount
data: {"viewercount":3}
~~~~~
{: title="Example event"}

The WHEP player MAY destroy the event stream at anytime by sending a HTTP DELETE request to the Url returned on the location header on the created request. The WHEP Resource MUST drop any pending queued event and return a "404 Not found" if any further long pull request is received for the event stream.

All the event streams associated with a WHEP Resource MUST be destroyed when the WHEP Resource is terminated.

#### active event
The event is sent by the WHEP Resource when an active publication for the WHEP resource, either at the begining of the playback when the resource is created or later during the playback session.

- event name: "active"
- event data: JSON object (TBD)

#### inactive event
The event is sent by the WHEP Resource when an active publication is no longer available. The WHEP Resource MUST not send an initial "inactive" event if there is no active publication when the resource is created.

- event name: "active"
- event data: JSON object (TBD)

#### layers event
The event is sent by the WHEP Resource to provide information to the WHEP player about the avialable video layers or renditions to be used in conjuction with the Layer Selection extension defined in Chapter {TBD}.

- event name: "layers"
- event data: JSON object

The WHEP Resource MAY send the event periodically or just when the layer information has changed.

The event data JSON object contains the video layer information available for each "m-line" indexed by the "m-line" order in the SDP.

Each value of the JSON object entries will be a JSON object with the following attributes

- active: (Array&lt;Object&gt;) Containing the information of the active simulcast layers.
- inactive: (Array&lt;Object&gt;) Containing the information of the inactive simulcast layers.
- layers: (Array&lt;Object&gt;) Containing the information of the active simulcast, spatials or temporal layers available for layer selection.

Each "active" JSON objet contains the following information:

- id: (String) rid value of the simulcast encoding of the layer
- simulcastIdx: (Number) the simulcast order of the encoding layer.
- bitrate: (Number) the spatial layer id of the encoding layer.
- width: (Number) the current video with of the encoding layer.
- heigth: (Number) the current video height of the encoding layer.
 
Each "inactive" JSON contains the following information:

- id: (String) rid value of the simulcast encoding of the layer.
- simulcastIdx: (Number) the simulcast order of the encoding layer.
- width: (Number) the current video with of the encoding layer
- heigth: (Number) the current video height of the encoding layer.
 
Each "layer" JSON contains the following information:

- encodingId: (String) rid value of the simulcast encoding of the layer
- simulcastIdx: (Number) the simulcast order of the encoding layer.
- spatialLayerId: (Number) the spatial layer id of the encoding layer.
- temporalLayerId: (Number) the temporal layer id of the encoding layer.
- bitrate: (Number) the spatial layer id of the encoding layer.
- width: (Number) the current video with of the encoding layer.
- heigth: (Number) the current video height of the encoding layer.

The "layer" object MUST containt at least one of the encodingId, spatialLayerId or temporalLayerId attributes, the other attributes are OPTIONAL.

~~~~~
{
  "0": {
    "active": [
      {
        "id": "1", "simulcastIdx": 1, "bitrate": 538288, width: 640, height: 360
      },
      {
        "id": "0", "simulcastIdx": 0, "bitrate": 111600, width: 320, height: 180
      }
    ],
    "inactive": [
      {
        "id": "2", "simulcastIdx": 2
      },
    ],
    "layers": [
      { "encodingId": "1", "simulcastIdx": 1, "spatialLayerId": 0, "temporalLayerId": 1, "bitrate": 557112, width: 640, height: 360 },
      { "encodingId": "1", "simulcastIdx": 1, "spatialLayerId": 0, "temporalLayerId": 0, "bitrate": 343592, width: 640, height: 360 },
      { "encodingId": "0", "simulcastIdx": 0, "spatialLayerId": 0, "temporalLayerId": 1, "bitrate": 116352, width: 320, height: 180 },
      { "encodingId": "0", "simulcastIdx": 0, "spatialLayerId": 0, "temporalLayerId": 0, "bitrate": 67464 , width: 320, height: 180 }
    ]
  }
}
~~~~~
{: title="Example event"}

#### viewercount event
The event is sent by the WHEP Resource to provide the WHIP Player the information of number of viewers currently connected to this resource.

- event name: "viewercount"
- event data: JSON object containing a "viewercount" attribute with a Number value indicating the number of viewers currently watching the WHIP resource.

The viewer count provided by the WHEP Resource MAY be approximate and not updated in real time but periodically to avoid  overloading both the event stream and the Media Server.

### Video Layer Selection extension

The Layer Selection extensions allows the WHEP player to control which video layer or rendition is being delivered through the negotiated video MediaStreamTrack. When supported by the WHEP resource, a "Link" header field with a "rel" attribute of "urn:ietf:params:whep:ext:core:layer" MUST be returned in the initial HTTP "201 Created" response, with the Url of the Video Layer Selection REST API entrypoint. If this extension is supported by the WHEP Resource, the Server Sent Events extension MUST be supported as well and the "layers" event MUST be advertised as well.

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whep.example.org/resource/213786HF
Link: <https://whep.ietf.org/resource/213786HF/layer>;
      rel="urn:ietf:params:whep:ext:core:layer"
Link: <https://whep.ietf.org/resource/213786HF/layer>;
      rel="urn:ietf:params:whep:ext:core:server-sent-events"
      events="layers"
~~~~~
{: title="HTTP 201 response example containing the Video Layer Selection extension"}

In case that Simulcast or Scalable Video Codecs are supported by the Media Server and used in the active publication to the WHEP Resource, by default, the Media Server will choose one of the available video layers to be sent to the WHEP player (based on bandwidth estimation or any other business logic). However, the WHEP player (or the person watching the stream) may decide that it whishes to receive a different one (to preserve bandwidth or to best fit in the UI). In this case the WHEP player MAY send a HTTP POST request to theVideo Layer Selection  API entrypoint containing an "application/json" body with an JSON object indicating the information of the video layer that wishes to be received. The WHEP Endpoint will return a "200 OK" if the switch to the new video layer can be performed or an appropiate HTTP error response if not.

The information that can sent on the JSON object in the POST request for doing layer selection is as follows:

- mediaId: (String) m-line index to apply the layer selection(default: first video m-line)
- encodingId: (String)  rid value of the simulcast encoding of the track (default: automatic selection)
- spatialLayerId: (Number) The spatial layer id to send to the outgoing stream (default: max layer available)
- temporalLayerId: (Number) The temporaral layer id to send to the outgoing stream (default: max layer available)
- maxSpatialLayerId: (Number) Max spatial layer id (default: unlimited)
- maxTemporalLayerId: (Number) Max temporal layer id (default: unlimited)

The information about the avialable encodings, spatial or temporal layers should be retrieverd from a "layers" event sent by the WHEP Resource using the Server Sent Events extension:

~~~~~
POST /resource/213786HF/layer HTTP/1.1
Host: whep.example.com
Content-Type: application/sjon

{mediaId:"0", "encodingId": "hd"}

HTTP/1.1 200 OK
~~~~~

If the WHEP player wishes to return to the default selection performed by the Media Server, it just need to send an empty JSON Object instead:

~~~~~
POST /resource/213786HF/layer HTTP/1.1
Host: whep.example.com
Content-Type: application/sjon

{}

HTTP/1.1 200 OK
~~~~~

# Security Considerations

This document specifies a new protocol on top of HTTP and WebRTC, thus, security protocols and considerations from related specifications apply to the WHEP specification. These include:****

- WebRTC security considerations: {{!RFC8826}}. HTTPS SHALL be used in order to preserve the WebRTC security model.
- Transport Layer Security (TLS): {{!RFC8446}} and {{!RFC9147}}.
- HTTP security: {{Section 11 of !RFC9112}} and {{Section 17 of !RFC9110}}.
- URI security: {{Section 7 of !RFC3986}}.

On top of that, the WHEP protocol exposes a thin new attack surface specific of the REST API methods used within it:

- HTTP POST flooding and resource exhaustion:
  It would be possible for an attacker in possession of authentication credentials valid for watching a WHEP stream to make multiple HTTP POST to the WHIP endpoint.
  This will force the WHEP endpoint to process the incoming SDP and allocate resources for being able to setup the DTLS/ICE connection.
  While the malicious client does not need to initiate the DTLS/ICE connection at all, the WHIP session will have to wait for the DTLS/ICE connection timeout in order to release the associated resources.
  If the connection rate is high enough, this could lead to resource exhaustion on the servers handling the requests and it will not be able to process legitimate incoming ingests.
  In order to prevent this scenario, WHEP endpoints SHOULD implement a rate limit and avalanche control mechanism for incoming initial HTTP POST requests.

- Insecure direct object references (IDOR) on the WHEP session locations:
  If the URLs returned by the WHIP endpoint for the WHEP sessions location are easy to guess, it would be possible for an attacker to send multiple HTTP DELETE requests and terminate all the WHEP sessions currently running.
  In order to prevent this scenario, WHEP endpoints SHOULD generate URLs with enough randomness, using a cryptographically secure pseudorandom number generator following the best practices in Randomness Requirements for Security {{!RFC4086}}, and implement a rate limit and avalanche control mechanism for HTTP DELETE requests.
  The security considerations for Universally Unique IDentifier (UUID) {{!RFC4122, Section 6}} are applicable for generating the WHEP sessions location URL.

- HTTP PATCH flooding: 
Similar to the HTTP POST flooding, a malicious client could also create a resource exhaustion by sending multiple HTTP PATCH request to the WHEP session, although the WHEP sessions can limit the impact by not allocating new ICE candidates and reusing the existing ICE candidates when doing ICE restarts.
In order to prevent this scenario, WHEP endpoints SHOULD implement a rate limit and avalanche control mechanism for incoming HTTP PATCH requests.

# IANA Considerations

This specification adds a registry for URN sub-namespaces for WHEP protocol extensions.

## Registration of WHEP URN Sub-namespace and WHEP Registry

IANA is asked to add an entry to the "IETF URN Sub-namespace for Registered Protocol Parameter Identifiers" registry and create a sub-namespace for the Registered Parameter Identifier as per {{!RFC3553}}: "urn:ietf:params:whep".

To manage this sub-namespace, IANA is asked to create the "WebRTC-HTTP ingestion protocol (WHEP) URNs" registry, which is used to manage entries within the "urn:ietf:params:whep" namespace. The registry description is as follows:

   - Registry name: WebRTC-HTTP ingestion protocol (WHEP) URNs

   - Specification: this document (RFC TBD)

   - Registration policy: Specification Required

   - Repository: See Section {{urn-whep-subspace}}

   - Index value: See Section {{urn-whep-subspace}}
     

## URN Sub-namespace for WHEP {#urn-whep-subspace}

WHEP endpoint utilizes URNs to identify the supported WHEP protocol extensions on the "rel" attribute of the Link header as defined in {{protocol-extensions}}.

This section creates and registers an IETF URN Sub-namespace for use in the WHEP specifications and future extensions.

### Specification Template

Namespace ID:

- The Namespace ID "whep" has been assigned.

Registration Information:

- Version: 1

- Date: TBD

Declared registrant of the namespace:

- Registering organization: The Internet Engineering Task Force.

- Designated contact: A designated expert will monitor the WHEP public mailing list, "wish@ietf.org".

Declaration of Syntactic Structure:

- The Namespace Specific String (NSS) of all URNs that use the "whep" Namespace ID shall have the following structure: urn:ietf:params:whep:{type}:{name}:{other}.

 - The keywords have the following meaning:

     - type: The entity type. This specification only defines the "ext" type.

     - name: A required US-ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines a major namespace of a WHEP protocol extension. The value MAY also be an industry name or organization name.

     - other: Any US-ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines the sub-namespace (which MAY be further broken down in namespaces delimited by colons) as needed to uniquely identify an WHEP protocol extension.

Relevant Ancillary Documentation:

 - None

Identifier Uniqueness Considerations:

- The designated contact shall be responsible for reviewing and enforcing uniqueness.

Identifier Persistence Considerations:

 - Once a name has been allocated, it MUST NOT be reallocated for a different purpose.
 - The rules provided for assignments of values within a sub-namespace MUST be constructed so that the meanings of values cannot change.
 - This registration mechanism is not appropriate for naming values whose meanings may change over time.

Process of Identifier Assignment:

- Namespace with type "ext" (e.g., "urn:ietf:params:whep:ext") is reserved for IETF-approved WHEP specifications.

Process of Identifier Resolution:

 - None specified.

Rules for Lexical Equivalence:

 - No special considerations; the rules for lexical equivalence specified in {{?RFC8141}} apply.

Conformance with URN Syntax:

 - No special considerations.

Validation Mechanism:

 - None specified.

Scope:

 - Global.

## Registering WHEP Protocol Extensions URNs

This section defines the process for registering new WHEP protocol extensions URNs with IANA in the "WebRTC-HTTP ingestion protocol (WHEP) URNs" registry (see {{urn-whep-subspace}}). 
   
A WHEP Protocol Extension URNs is used as a value in the "rel" attribute of the Link header as defined in {{protocol-extensions}} for the purpose of signaling the WHEP protocol extensions supported by the WHEP endpoints.
   
WHEP Protocol Extensions URNs have a "ext" type as defined in {{urn-whep-subspace}}.

###  Registration Procedure

   The IETF has created a mailing list, "wish@ietf.org", which can be used
   for public discussion of WHEP protocol extensions proposals prior to registration.
   Use of the mailing list is strongly encouraged. The IESG has
   appointed a designated expert {{RFC8126}} who will monitor the
   wish@ietf.org mailing list and review registrations.

   Registration of new "ext" type URNs (in the namespace "urn:ietf:params:whep:ext") belonging to a WHEP Protocol Extension MUST be documented in a permanent and readily available public specification, in sufficient detail so that interoperability between independent implementations is possible and reviewed by the designated expert as per Section 4.6 of {{!BCP26}} .
   An RFC is REQUIRED for the registration of new value data types that modify existing properties.
   An RFC is also REQUIRED for registration of WHEP Protocol Extensions URNs that modify WHEP Protocol Extensions previously documented in an existing RFC.

   The registration procedure begins when a completed registration template, defined in the sections below, is sent to iana@iana.org.
   Decisions made by the designated expert can be appealed to an Applications and Real Time (ART) Area Director, then to the IESG.
   The normal appeals procedure described in {{!BCP9}} is to be followed. 
   
   Once the registration procedure concludes successfully, IANA creates
   or modifies the corresponding record in the WHEP Protocol Extension registry.

   An RFC specifying one or more new WHEP Protocol Extension URNs MUST include the
   completed registration templates, which MAY be expanded with
   additional information. These completed templates are intended to go
   in the body of the document, not in the IANA Considerations section.
   The RFC MUST include the syntax and semantics of any extension-specific attributes that may be provided in a Link header
   field advertising the extension.

### Guidance for Designated Experts

The Designated Expert (DE) is expected to ascertain the existence of suitable documentation (a specification) as described in {{RFC8126}} and to verify that the document is permanently and publicly available. 

The DE is also expected to check the clarity of purpose and use of the requested registration.

Additionally, the DE must verify that any request for one of these registrations has been made available for review and comment within the IETF: the DE will post the request to the WebRTC Ingest Signaling over HTTPS (wish) Working Group mailing list (or a successor mailing list designated by the IESG). 

If the request comes from within the IETF, it should be documented in an Internet-Draft. Lastly, the DE must ensure that any other request for a code point does not conflict with work that is active or already published within the IETF.

###  WHEP Protocol Extension Registration Template

A WHEP Protocol Extension URNs is defined by completing the following template:

 -   URN: A unique URN for the WHEP Protocol Extension.
 -   Reference: A formal reference to the publicly available specification
 -   Name: A descriptive name of the WHEP Protocol Extension extension.
 -   Description: A brief description of the function of the extension, in a short paragraph or two.
 -   Contact information: Contact information for the organization or person making the registration.


# Acknowledgements

--- back


