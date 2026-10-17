Yes. Your mental model is already close. The missing piece is mainly **how routing works beyond your default gateway**.

The most important correction is:

> A router usually does **not** know the exact IP address of `reactbits.dev`.
> 
> It knows: **“For IP addresses beginning with this prefix, send packets toward that network.”**

The Internet is essentially a hierarchy of routers forwarding packets toward increasingly more appropriate networks.

---

# 1. First: what an ISP network actually looks like

A simplified Internet connection looks like this:

```
Your phone
192.168.1.20
      │
      │ Wi-Fi
      ▼
Your home router
192.168.1.1
WAN: public IP
      │
      │ fiber / cable / DSL / Ethernet
      ▼
ISP access network
      │
      ▼
ISP aggregation router
      │
      ▼
ISP core network
      │
      ▼
ISP edge / border router
      │
      │ BGP
      ▼
Other ISP / transit provider / IXP
      │
      ▼
Destination network / CDN
      │
      ▼
Load balancer / web server
```

Your home router is only the first router.

The Internet contains enormous numbers of routers grouped into networks called **Autonomous Systems**, or ASes.

For example:

```
Your LAN
    │
    ▼
ISP AS
    │
    ▼
Transit AS
    │
    ▼
CDN AS
    │
    ▼
server
```

Each AS has an **Autonomous System Number**, ASN.

Large organizations can run their own AS:

```
Cloudflare
Google
Amazon
Microsoft
ISPs
telecom companies
CDNs
universities
large enterprises
```

---

# 2. Your router only knows where to send the packet next

Imagine DNS tells you:

```
reactbits.dev → 203.0.113.80
```

I'm using a fictional IP here.

Your phone creates an IP packet:

```
Source IP:      192.168.1.20
Destination IP: 203.0.113.80
```

Your phone checks its routing table.

Something roughly like:

```
Destination         Next hop
-----------------------------------
192.168.1.0/24      directly connected
0.0.0.0/0           192.168.1.1
```

`203.0.113.80` doesn't belong to:

```
192.168.1.0/24
```

so:

```
0.0.0.0/0
```

wins.

That's the default route.

So the phone thinks:

```
I don't know where 203.0.113.80 is.

But my router 192.168.1.1 says:
"Give unknown Internet traffic to me."

So I'll give it to the router.
```

The important thing is:

```
Destination IP remains:
203.0.113.80
```

The phone isn't saying:

```
send packet TO my router
```

at the IP layer.

It is saying:

```
send packet THROUGH my router
toward 203.0.113.80
```

This distinction becomes extremely important.

---

# 3. IP destination vs next-hop destination

There are really two destinations involved.

At layer 3:

```
IP destination = 203.0.113.80
```

At layer 2:

```
next-hop MAC = your router's MAC
```

So the packet initially looks approximately like this:

```
Wi-Fi frame
┌─────────────────────────────────────────────┐
│ Destination MAC: HOME_ROUTER_MAC            │
│ Source MAC:      PHONE_MAC                  │
│                                             │
│   IP packet                                 │
│   ┌─────────────────────────────────────┐   │
│   │ Src IP: 192.168.1.20               │   │
│   │ Dst IP: 203.0.113.80               │   │
│   │                                     │   │
│   │ TCP segment ...                     │   │
│   └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

Notice:

```
MAC destination = router

IP destination = website
```

The MAC address changes constantly as the packet crosses routers.

The destination IP normally does not.

---

# 4. Every router removes the old layer-2 frame

Suppose the path is:

```
Phone
  ↓
Router A
  ↓
Router B
  ↓
Router C
  ↓
Server
```

The IP packet might conceptually remain:

```
Src IP = your public IP
Dst IP = server IP
```

But the layer-2 envelope changes at every hop.

First:

```
PHONE_MAC → ROUTER_A_MAC

[ IP packet ]
```

Router A removes that frame.

Then creates another:

```
ROUTER_A_MAC → ROUTER_B_MAC

[ same IP packet ]
```

Router B removes it.

Then:

```
ROUTER_B_MAC → ROUTER_C_MAC

[ same IP packet ]
```

Eventually:

```
LAST_ROUTER_MAC → SERVER/LB_MAC

[ same IP packet ]
```

This is one of the most fundamental ideas in networking:

```
Layer 2:
hop-to-hop

Layer 3:
end-to-end
```

---

# 5. So how does the ISP know where to send `203.0.113.80`?

This is where the Internet routing table comes in.

Routers don't normally store entries like:

```
203.0.113.80 → ReactBits
```

They store prefixes.

For example:

```
203.0.113.0/24 → Router X
```

That means:

```
203.0.113.0
...
203.0.113.255
```

can be reached through Router X.

So `203.0.113.80` matches.

---

# 6. Longest Prefix Match

Imagine a router has:

```
0.0.0.0/0           → Router A
203.0.0.0/8         → Router B
203.0.113.0/24       → Router C
```

Packet destination:

```
203.0.113.80
```

All three technically match.

But routers use:

> **Longest Prefix Match**

The most specific route wins.

So:

```
203.0.113.0/24
```

beats:

```
203.0.0.0/8
```

and:

```
0.0.0.0/0
```

Therefore:

```
send to Router C
```

---

# 7. But where did that `/24` route come from?

This is the really interesting part.

It probably came from **BGP**.

BGP = Border Gateway Protocol.

BGP is essentially how Internet networks tell each other:

> "I know how to reach these IP prefixes."

Imagine the destination network owns:

```
203.0.113.0/24
```

It might announce:

```
AS 65003:

"I can reach 203.0.113.0/24."
```

Its upstream provider learns that.

```
AS65003
   │
   ▼
AS65002
```

AS65002 tells others:

```
203.0.113.0/24
reachable through:

AS65002 → AS65003
```

Then your ISP learns something like:

```
203.0.113.0/24
next hop: ISP-border-router-7
AS path:
65002 65003
```

So your ISP already knows:

```
Traffic toward 203.0.113.x
should leave through this border router.
```

This information exists **before your request arrives**.

Your packet doesn't trigger routers to search the Internet for the destination.

The routing system has already built forwarding tables.

---

# 8. BGP is not usually moving the packet itself

There's another subtle distinction.

BGP may tell your ISP:

```
203.0.113.0/24 should leave our network
through Border Router 7
```

But your packet may currently be entering:

```
Access Router 42
```

So internally the ISP needs to get:

```
Access Router 42
        ↓
Core Router
        ↓
Core Router
        ↓
Border Router 7
```

Inside an ISP, protocols such as:

```
OSPF
IS-IS
```

may handle internal routing.

So conceptually:

```
BGP:
"Which external network should we send this toward?"

IGP:
"How do we reach the correct exit router inside our own network?"
```

Large ISP cores may also use MPLS.

---

# 9. Think of BGP like international highway signs

Imagine driving from Vancouver to a specific house in Tokyo.

At Vancouver, you don't need a sign saying:

```
Takaki's friend's apartment
Building 4
Room 302
```

You need something broader:

```
Japan → this direction
```

Later:

```
Tokyo → this direction
```

Later:

```
Shinjuku → this direction
```

Eventually:

```
this street
```

Finally the local network knows the exact destination.

Internet routing is similar.

Earlier routers know:

```
toward that AS / prefix
```

Later routers become progressively closer to the network that owns the address.

---

# 10. Does one router eventually know the exact IP?

Sort of, but not necessarily as `/32`.

This is the part you were asking about.

You said:

> one of the router has to know the exact IP address of reactbits.dev right?

Not exactly.

Suppose:

```
server = 203.0.113.80
```

The destination network might have:

```
203.0.113.0/24
```

connected to one of its routers.

The router only needs:

```
203.0.113.0/24 is reachable through interface X
```

Then when it needs to send the packet onto the final local network, it may use ARP:

```
Who has 203.0.113.80?
```

A machine or load balancer replies:

```
203.0.113.80 is at MAC AA:BB:CC:DD:EE:FF
```

Then:

```
Router
   │
   │ Ethernet frame
   │ dst MAC = AA:BB:CC:DD:EE:FF
   ▼
203.0.113.80
```

Now we've reached the actual IP endpoint.

In real modern hosting, that endpoint is often not the application server itself.

It might be:

```
CDN edge
reverse proxy
load balancer
DDoS protection system
```

which then forwards to the real application server.

---

# 11. CDNs make this even more interesting

For modern sites, DNS may point you toward a CDN.

Then:

```
reactbits.dev
      │
      ▼
CDN edge IP
      │
      ▼
nearest/best CDN Point of Presence
      │
      ▼
cache / reverse proxy
      │
      ▼
possibly origin server
```

And technologies such as **Anycast** can make the same IP address exist at multiple locations:

```
203.0.113.80
   ├── Vancouver
   ├── Seattle
   ├── Tokyo
   ├── London
   └── Frankfurt
```

Those locations all advertise the same prefix.

BGP decides which advertisement your ISP prefers.

So even:

```
same IP
```

can lead different users to different physical servers.

---

# 12. Now let's trace the entire request

Let's assume:

```
Phone:
192.168.1.20

Home router:
192.168.1.1

Router public IP:
198.51.100.50

reactbits.dev:
203.0.113.80
```

Again, the public addresses are fictional examples.

You type:

```
https://reactbits.dev/
```

---

# Stage A — browser parses the URL

The browser sees:

```
scheme: https
hostname: reactbits.dev
port: 443
path: /
```

Because:

```
https
```

normally means:

```
TCP port 443
```

assuming HTTP/1.1 or HTTP/2.

HTTP/3 is different; we'll come back to that.

---

# Stage B — DNS resolution

The browser needs:

```
reactbits.dev → IP address
```

It cannot establish an IP connection using the domain name alone.

The browser may first check caches:

```
Browser DNS cache
        ↓
OS resolver cache
        ↓
configured DNS resolver
```

Your resolver might be:

```
ISP DNS
Google
Cloudflare
Quad9
a VPN resolver
DoH provider
```

Browsers may also perform DNS-over-HTTPS themselves.

If the recursive DNS resolver doesn't know the answer, conceptually it asks:

```
root DNS
   │
   ▼
.dev nameserver
   │
   ▼
reactbits.dev authoritative nameserver
```

Eventually:

```
reactbits.dev A    → IPv4 address
reactbits.dev AAAA → IPv6 address
```

Suppose:

```
203.0.113.80
```

comes back.

The answer has a TTL:

```
TTL = 300
```

meaning the result may be cached for about five minutes.

---

# Stage C — browser asks the OS to connect

The browser doesn't manually construct Ethernet frames.

It asks the operating system.

Conceptually:

```
socket(AF_INET, SOCK_STREAM, TCP)
```

Then:

```
connect(
    destination = 203.0.113.80,
    port = 443
)
```

The kernel creates TCP state.

It selects an ephemeral source port.

For example:

```
192.168.1.20:52784
        ↓
203.0.113.80:443
```

This four-tuple uniquely identifies this TCP connection:

```
source IP
source port
destination IP
destination port
```

So:

```
192.168.1.20
52784
203.0.113.80
443
```

---

# Stage D — OS routing lookup

Linux/macOS/Android/iOS now asks:

```
Where do I send 203.0.113.80?
```

Your phone's routing table might effectively say:

```
192.168.1.0/24 → Wi-Fi directly
0.0.0.0/0      → 192.168.1.1
```

Therefore:

```
203.0.113.80
    ↓
default gateway
    ↓
192.168.1.1
```

---

# Stage E — ARP

But now your phone has another problem.

It knows:

```
gateway IP:
192.168.1.1
```

But Wi-Fi/Ethernet delivery requires a layer-2 address.

So the phone needs:

```
192.168.1.1 → MAC address
```

With IPv4, ARP handles this.

Conceptually:

```
PHONE:

"Who has 192.168.1.1?"
```

Router responds:

```
192.168.1.1 is at
AA:AA:AA:AA:AA:AA
```

Phone stores that in its neighbor/ARP cache.

Now it can construct a frame.

---

# Stage F — TCP SYN

Before HTTPS can happen, TCP must be established.

The browser's kernel sends:

```
TCP SYN
```

Conceptually:

```
IP
──────────────────────────
source:
192.168.1.20

destination:
203.0.113.80

TCP
──────────────────────────
source port:
52784

destination port:
443

flags:
SYN = 1

sequence:
x
```

TCP also sends useful options in the SYN, commonly including things like:

```
MSS
Window Scale
SACK permitted
timestamps
```

---

# Stage G — MSS and MTU

This ties directly into your MTU question.

Suppose your IP interface MTU is:

```
1500 bytes
```

For IPv4:

```
IP header:
20 bytes

TCP header:
20 bytes
```

So maximum normal TCP payload is approximately:

```
1500 - 20 - 20
= 1460 bytes
```

That's the TCP **MSS**.

So your SYN might advertise:

```
MSS = 1460
```

Meaning:

> Please don't send me TCP segments whose TCP payload is larger than roughly 1460 bytes.

IPv6 has a 40-byte base IP header, so with a 1500 MTU:

```
1500 - 40 - 20
= 1440
```

---

# 13. Why MTU exists

An Ethernet link might support:

```
MTU = 1500
```

But somewhere later you might encounter:

```
PPPoE
MTU = 1492
```

or a tunnel with an even smaller effective MTU.

If your packet is too large, the network needs a mechanism to avoid sending packets that cannot fit.

This leads to:

```
Path MTU Discovery
```

For IPv4, a router may send:

```
ICMP Fragmentation Needed
```

if fragmentation isn't allowed.

For IPv6:

```
ICMPv6 Packet Too Big
```

because IPv6 routers don't fragment packets in transit.

Your OS can then lower its effective packet size.

---

# Stage H — Wi-Fi frame

Your SYN packet is wrapped in a Wi-Fi frame.

Conceptually:

```
802.11 frame

destination:
router/AP MAC

source:
phone MAC

payload:
┌──────────────────────────┐
│ IP                       │
│ Src: 192.168.1.20        │
│ Dst: 203.0.113.80        │
│                          │
│ TCP                      │
│ 52784 → 443              │
│ SYN                      │
└──────────────────────────┘
```

Your AP receives it.

---

# Stage I — NAT on your router

Your phone has:

```
192.168.1.20
```

That's a private IP.

Internet routers won't route that globally.

Your router therefore performs NAT/PAT.

Before:

```
192.168.1.20:52784
        ↓
203.0.113.80:443
```

Router changes it to something like:

```
198.51.100.50:61001
        ↓
203.0.113.80:443
```

It creates a NAT table entry:

```
198.51.100.50:61001
          ↕
192.168.1.20:52784
```

This is incredibly important for the return path.

---

# 14. What if you have two routers?

You specifically described:

```
phone
  ↓
router
  ↓
ISP router
```

If both devices perform NAT, you may actually have **double NAT**.

For example:

```
Phone
192.168.1.20
        │
        ▼ NAT #1
Home router
10.0.0.5
        │
        ▼ NAT #2
ISP router
198.51.100.50
        │
        ▼
Internet
```

There could even be a third NAT layer if your ISP uses CGNAT:

```
phone
 ↓
home NAT
 ↓
ISP-provided router NAT
 ↓
CGNAT
 ↓
Internet
```

But routing works on the same principle.

---

# Stage J — home router performs route lookup

Now your home router asks:

```
destination = 203.0.113.80

where?
```

Its routing table may be simple:

```
LAN routes
WAN route
default via ISP
```

So:

```
default
    ↓
ISP gateway
```

It creates the appropriate layer-2 frame for its WAN interface and sends the packet.

---

# Stage K — ISP receives the packet

The ISP access router sees:

```
Dst:
203.0.113.80
```

It performs a hardware forwarding-table lookup.

Large routers don't normally search giant text routing tables like your Linux machine does.

They have optimized forwarding structures.

Conceptually the routing system has two related ideas:

```
RIB
Routing Information Base

FIB
Forwarding Information Base
```

RIB is roughly:

```
all routing knowledge
```

FIB is:

```
the routes actually used for fast packet forwarding
```

High-end routers may implement prefix lookup in specialized hardware.

---

# Stage L — ISP chooses where to send it

Suppose BGP has taught your ISP:

```
203.0.113.0/24

reachable through
Border Router 7
```

So the ISP moves the packet through its network:

```
Access router
     │
     ▼
Aggregation router
     │
     ▼
Core router
     │
     ▼
Border Router 7
```

Every router performs something conceptually like:

```
destination IP?
203.0.113.80

best matching prefix?
203.0.113.0/24

next hop?
router X

send.
```

Again.

And again.

And again.

---

# 15. TTL changes at every router

Your IPv4 packet includes:

```
TTL
```

Perhaps initially:

```
TTL = 64
```

Router 1:

```
64 → 63
```

Router 2:

```
63 → 62
```

Router 3:

```
62 → 61
```

If TTL reaches:

```
0
```

the router discards the packet and normally returns:

```
ICMP Time Exceeded
```

This mechanism prevents routing loops from circulating forever.

It's also basically the trick used by:

```
traceroute
```

---

# Stage M — ISP leaves its own AS

Eventually your ISP border router might peer with another network.

For example:

```
Your ISP
AS100
   │
   │ peering connection
   ▼
Transit provider
AS200
   │
   ▼
CDN
AS300
```

BGP might have supplied an AS path:

```
AS200 AS300
```

Your ISP chooses this route according to policy.

Not necessarily:

```
physically shortest
```

Not necessarily:

```
fewest routers
```

BGP routing is heavily influenced by business relationships and policy.

---

# 16. Internet Exchange Points

Sometimes two networks connect at an IXP:

```
ISP A ─────┐
           │
ISP B ─────┼── IXP switching fabric
           │
Cloud CDN ─┤
           │
Google ────┘
```

Instead of:

```
ISP
 ↓
expensive transit provider
 ↓
CDN
```

they may directly peer:

```
ISP → IXP → CDN
```

This can greatly reduce latency and cost.

---

# Stage N — destination network receives the packet

Eventually a router in the destination network sees:

```
203.0.113.80
```

Perhaps it has:

```
203.0.113.0/24
directly connected / reachable internally
```

Now the packet travels through the datacenter.

Something like:

```
Internet edge
     │
     ▼
DDoS protection
     │
     ▼
load balancer
     │
     ▼
reverse proxy
     │
     ▼
application server
```

The IP you connected to may belong to the load balancer rather than the application server.

---

# Stage O — SYN reaches TCP on the server

Eventually something owns:

```
203.0.113.80:443
```

Its kernel receives:

```
SYN

Src:
198.51.100.50:61001

Dst:
203.0.113.80:443

SEQ=x
```

It has a listening socket:

```
TCP :443
```

The server responds:

```
SYN + ACK
```

Something like:

```
SEQ=y
ACK=x+1

SYN=1
ACK=1
```

---

# Stage P — response travels back

The response packet:

```
Src:
203.0.113.80:443

Dst:
198.51.100.50:61001
```

travels back across the Internet.

Importantly:

> The return route does not have to be identical to the outgoing route.

You might have:

```
outbound:

A → B → C → D
```

but return:

```
D → X → Y → A
```

IP routing is independently decided in each direction.

---

# Stage Q — NAT reverses the translation

Eventually the SYN-ACK reaches your router:

```
Dst:
198.51.100.50:61001
```

The NAT table says:

```
198.51.100.50:61001
        ↕
192.168.1.20:52784
```

So the router rewrites the packet:

```
destination:
192.168.1.20:52784
```

Then sends it over Wi-Fi to your phone.

---

# Stage R — third TCP packet

Your phone receives:

```
SYN + ACK
```

Then sends:

```
ACK
```

So the famous TCP three-way handshake is:

```
PHONE                              SERVER

 SYN
 SEQ=x
───────────────────────────────────>

                         SYN + ACK
                  SEQ=y, ACK=x+1
<───────────────────────────────────

 ACK
 ACK=y+1
───────────────────────────────────>
```

Now both sides have established:

```
ESTABLISHED
```

They have also synchronized sequence-number spaces.

---

# 17. Why TCP needs sequence numbers

Suppose the server sends:

```
bytes 1-1460
bytes 1461-2920
bytes 2921-4380
```

The packets may arrive:

```
packet 1
packet 3
packet 2
```

TCP can reorder them because it knows sequence numbers.

Or:

```
packet 2 disappears
```

TCP notices the missing byte range and retransmits it.

So TCP provides the browser with the abstraction:

```
reliable ordered byte stream
```

even though IP itself provides only:

```
best-effort packets
```

---

# 18. TCP flow control

Your phone has finite memory.

So TCP advertises a receive window:

```
rwnd
```

Meaning approximately:

> I currently have this much buffer capacity available.

This prevents the server from overwhelming the receiver.

There is also a separate mechanism:

```
congestion control
```

which prevents senders from overwhelming the network.

So sending is constrained by roughly:

```
receiver capacity
AND
network capacity
```

These are different problems.

---

# Stage S — TLS starts

You requested:

```
https://
```

so HTTP isn't sent immediately in plaintext.

First comes TLS.

For modern TLS 1.3, the browser sends something roughly like:

```
ClientHello
```

containing information such as:

```
TLS versions supported
cipher suites
key share
server name
ALPN protocols
```

An important field is:

```
SNI = reactbits.dev
```

Why?

Because one server IP might host:

```
reactbits.dev
example.com
foo.com
bar.com
```

The server needs to know which certificate/site you want.

---

# 19. ALPN

The browser might advertise:

```
h2
http/1.1
```

using ALPN.

Then the server can say:

```
Use HTTP/2.
```

So HTTPS involves several layers:

```
HTTP/2
   ↓
TLS
   ↓
TCP
   ↓
IP
   ↓
Wi-Fi / Ethernet
```

---

# 20. TLS key exchange

Very simplified TLS 1.3:

```
Browser                          Server

ClientHello
key_share
SNI=reactbits.dev
ALPN=h2
────────────────────────────────>

                                ServerHello
                                key_share
<────────────────────────────────

                          Certificate
                          CertificateVerify
                          Finished
<────────────────────────────────

Finished
────────────────────────────────>
```

Both sides derive shared encryption keys.

The server also sends its certificate.

Your browser checks:

```
Is the certificate valid?

Does it cover:
reactbits.dev?

Is it signed through a trusted
certificate authority chain?

Is it expired/revoked/etc.?
```

After that:

```
browser ↔ server
```

can communicate securely.

Routers in the middle generally see:

```
source/destination IP
packet sizes
timing
TCP/UDP information
```

but not the encrypted HTTP contents.

---

# Stage T — actual HTTP request

Only now does your browser send something logically resembling:

```
GET / HTTP/2
Host: reactbits.dev
```

With HTTP/2, the exact wire format is binary rather than that textual representation.

There may also be:

```
cookies
Accept
User-Agent
Accept-Encoding
cache-related headers
etc.
```

TLS encrypts this.

So a router doesn't see:

```
GET /
```

It sees:

```
encrypted TCP payload
```

---

# 21. HTTP data becomes TLS data, then TCP data

Conceptually:

```
HTTP
"GET /"
       │
       ▼
TLS
encrypt()
       │
       ▼
encrypted bytes
       │
       ▼
TCP
split into segments
       │
       ▼
IP packets
       │
       ▼
Wi-Fi/Ethernet frames
```

And on the server:

```
frame
  ↓
IP
  ↓
TCP reassembly
  ↓
TLS decryption
  ↓
HTTP parser
  ↓
web server/application
```

This encapsulation concept is extremely important.

---

# 22. Where MTU becomes relevant again

Suppose the server wants to send:

```
100 KB
```

of HTML.

TCP doesn't send:

```
one 100 KB IP packet
```

Instead, the data stream is divided into segments.

Roughly:

```
1460 bytes
1460 bytes
1460 bytes
1460 bytes
...
```

if MSS is around 1460.

Each becomes something conceptually like:

```
Ethernet:
1500-byte IP packet maximum

IP:
20 bytes

TCP:
20 bytes

payload:
1460 bytes
```

Although options, IPv6, tunnels, offloads, etc. can change the exact numbers.

---

# 23. TCP acknowledgements

Suppose the server sends bytes:

```
1-1460

1461-2920

2921-4380
```

The browser can acknowledge:

```
ACK = 4381
```

meaning:

> I successfully received everything through byte 4380. Send me byte 4381 onward.

ACKs can be cumulative.

If something disappears, TCP can retransmit the missing data.

SACK can provide more specific information about received ranges.

---

# Stage U — server creates the response

The server or CDN might respond:

```
HTTP 200 OK
```

with HTML.

Conceptually:

```
<html>
  ...
  <link href="/style.css">
  <script src="/app.js"></script>
  <img src="/hero.webp">
</html>
```

The browser receives the HTML.

But the website isn't finished yet.

---

# Stage V — browser discovers more resources

The browser parses the HTML and discovers:

```
CSS
JavaScript
images
fonts
API requests
```

Now more requests occur.

Possibly:

```
reactbits.dev
cdn.example.com
fonts.example.com
api.example.com
```

Every new hostname may require DNS.

But connections can often be reused.

For HTTP/2:

```
one TCP connection
        │
        ├── HTML stream
        ├── CSS stream
        ├── JS stream
        └── image stream
```

rather than creating a new TCP connection for every file.

---

# 24. Then the browser renders

Once enough resources arrive:

```
HTML
 ↓
DOM

CSS
 ↓
CSSOM

DOM + CSSOM
 ↓
layout
 ↓
paint
 ↓
compositing
 ↓
pixels on screen
```

JavaScript may modify the DOM, fetch more API data, load chunks, hydrate React components, and so on.

Eventually:

```
reactbits.dev
```

appears on your display.

---

# 25. The full journey in one picture

Here is the entire system:

```
You type:

https://reactbits.dev/
        │
        ▼
Browser parses URL
        │
        ▼
Need IP address
        │
        ▼
DNS
reactbits.dev → IP
        │
        ▼
Browser asks kernel:
connect(IP, 443)
        │
        ▼
Kernel creates TCP socket

src:
192.168.1.20:52784

dst:
203.0.113.80:443
        │
        ▼
Routing table lookup
        │
        ▼
default gateway
192.168.1.1
        │
        ▼
ARP:
192.168.1.1 → router MAC
        │
        ▼
TCP SYN
        │
        ▼
Wi-Fi frame
        │
        ▼
Home router
        │
        ▼
NAT

192.168.1.20:52784
        ↓
198.51.100.50:61001
        │
        ▼
Default route
        │
        ▼
ISP access network
        │
        ▼
ISP aggregation
        │
        ▼
ISP core
        │
        ▼
BGP-selected border router
        │
        ▼
ISP / IXP / transit
        │
        ▼
Destination AS
        │
        ▼
CDN/load balancer
        │
        ▼
TCP endpoint

       SYN
────────────────>

     SYN ACK
<────────────────

       ACK
────────────────>

TCP established
        │
        ▼
TLS ClientHello
        │
        ▼
TLS ServerHello +
certificate +
key exchange
        │
        ▼
encrypted TLS session
        │
        ▼
HTTP request

GET /
Host: reactbits.dev
        │
        ▼
Web server / CDN / app
        │
        ▼
HTTP response
        │
        ▼
TLS encryption
        │
        ▼
TCP segmentation
        │
        ▼
IP packets
        │
        ▼
Internet routers
        │
        ▼
Your ISP
        │
        ▼
Home router
        │
        ▼
NAT reverse mapping
        │
        ▼
Wi-Fi
        │
        ▼
Phone
        │
        ▼
TCP reassembly
        │
        ▼
TLS decrypt
        │
        ▼
HTTP
        │
        ▼
HTML/CSS/JS
        │
        ▼
DOM/layout/paint
        │
        ▼
SCREEN
```

---

# 26. The most important answer to your original confusion

Your intuition was:

> If every router just sends unknown traffic to another default router, eventually someone has to know the exact address.

Close.

But the Internet is more like this:

```
Your phone:
"I don't know 203.0.113.80.
Send to home router."

Home router:
"I don't know that network.
Send to ISP."

ISP access router:
"203.0.113.80 belongs to a prefix
that should exit through Border Router 7."

ISP core:
"Border Router 7 is this direction."

Border router:
"BGP says 203.0.113.0/24
is reachable through AS200."

AS200:
"That prefix is reachable through AS300."

AS300:
"203.0.113.0/24 is inside my network."

Datacenter router:
"203.0.113.0/24 is over there."

Final network:
"203.0.113.80 is at this MAC / load balancer."
```

So nobody needs a universal table saying:

```
every IP address → exact server
```

Instead:

```
prefix → next hop
```

repeated millions of times is enough.

---

# 27. And one final subtlety: HTTP/3 changes part of this

Everything above about TCP is correct for:

```
HTTP/1.1
HTTP/2
```

But modern browsers can also use:

```
HTTP/3
```

HTTP/3 uses:

```
HTTP/3
   ↓
QUIC
   ↓
UDP
   ↓
IP
```

instead of:

```
HTTP/2
   ↓
TLS
   ↓
TCP
   ↓
IP
```

QUIC effectively incorporates functionality traditionally split between TCP and TLS:

```
reliability
streams
congestion control
encryption
handshake
```

So for a real modern website, your browser might initially establish a TCP/TLS connection, learn that HTTP/3 is available, and later use QUIC—or it may already know from cached information and connect using QUIC directly.

But **everything below transport remains conceptually the same**:

```
DNS
routing
IP prefixes
BGP
next hops
NAT
MTU
layer-2 frames
ISP network
```

The most useful next step for mastering this would be to take **one SYN packet from your actual Ubuntu machine** and trace it through `ip route`, ARP/neighbour lookup, NAT, TCP sequence numbers, MTU/MSS, and Wireshark/tcpdump. That would connect `/proc`, Linux networking, routing, sockets, TCP, and the ISP concepts into one concrete packet journey.