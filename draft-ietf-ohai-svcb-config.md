---
title: "Discovery of Oblivious Services via Service Binding Records"
abbrev: "Oblivious Services in SVCB"
category: std
stream: IETF

docname: draft-ietf-ohai-svcb-config-latest
area: "Security"
workgroup: "Oblivious HTTP Application Intermediation"
keyword: Internet-Draft
venue:
  group: "Oblivious HTTP Application Intermediation"
  type: "Working Group"
  mail: "ohai@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ohai/"
  github: ietf-wg-ohai/draft-ohai-svcb-config

v: 3

author:
 -
    name: Tommy Pauly
    organization: Apple Inc.
    email: tpauly@apple.com
 -
    name: Tirumaleswar Reddy
    organization: Nokia
    email: kondtir@gmail.com

normative:

informative:


--- abstract

This document defines a parameter that can be included in SVCB and HTTPS
DNS resource records to denote that a service is accessible using Oblivious
HTTP, by offering an Oblivious Gateway Resource through which to access the
target. This document also defines a mechanism to learn the key configuration
of the discovered Oblivious Gateway Resource.

--- middle

# Introduction

Oblivious HTTP {{!OHTTP=I-D.draft-ietf-ohai-ohttp}} allows clients to encrypt
messages exchanged with an Oblivious Target Resource (target). The messages
are encapsulated in encrypted messages to an Oblivious Gateway Resource
(gateway), which gates access to the target. The gateway is access via an
Oblivious Relay Resource (relay), which proxies the encapsulated messages
to hide the identity of the client. Overall, this architecture is designed
in such a way that the relay cannot inspect the contents of messages, and
the gateway and target cannot discover the client's identity.

Since Oblivious HTTP deployments will often involve very specific coordination
between clients, relays, and gateways, the key configuration can often be
shared in a bespoke fashion. However, some deployments involve clients
discovering oblivious targets and their associated gateways more dynamically.
For example, a network might operate a DNS resolver that provides more optimized
or more relevant DNS answers and is accessible using Oblivious HTTP, and might
want to advertise support for Oblivious HTTP via mechanisms like Discovery of
Designated Resolvers ({{!DDR=I-D.draft-ietf-add-ddr}}). Clients can work with trusted
relays to access these gateways.

This document defines a way to use DNS records to advertise that an HTTP service
supports Oblivious HTTP. This indication is a parameter that can be included in SVCB
and HTTPS DNS resource records {{!SVCB=I-D.draft-ietf-dnsop-svcb-https}} ({{svc-param}}).
The presence of this parameter indicates that a service can act as an oblivious
target and has an oblivious gateway that can provide access to the target.

The client learns the URI to use for the oblivious gateway using a well-known
URI {{!WELLKNOWN=RFC8615}}, "ohttp-gateway", which is accessed on the
oblivious target ({{gateway-location}}). This means that for deployments that
support this kind of discovery, the gateway and target resources need to
be located on the same host.

This document also defines a way to fetch an oblivious gateway's key
configuration from the oblivious gateway ({{config-fetch}}).

This mechanism does not aid in the discovery of oblivious relays;
relay configuration is out of scope for this document. Models in which
this discovery mechanism is applicable are described in {{applicability}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Applicability {#applicability}

There are multiple models in which the discovery mechanism defined
in this document can be used.

- Upgrading non-oblivious HTTP to oblivious HTTP. In this model, the
client intends to communicate with a specific target service, and
prefers to use oblivious HTTP if it is available. The target service
has an oblivious gateway that it offers to allow access using oblivious
HTTP. Once the client learns about the oblivious gateway, it "upgrades"
to using oblivious HTTP to access the target service.

- Discovering alternative oblivious HTTP services. In this model,
the client has a default oblivious target service that it uses. For
example, this may be a public DNS resolver that is accessible over
oblivious HTTP. The client is willing to use alternative oblivious
target services if they are discovered, which may provide more
optimized or more relevant responses.

In both of these deployment models, the client is assumed to already
know of an oblivious relay that it trusts and works with. This oblivious
relay either needs to provide generic access to oblivious gateways, or
provide a service to clients to allow them to check which gateways
are accessible.

# The ohttp SvcParamKey {#svc-param}

The "ohttp" SvcParamKey ({{iana}}) is used to indicate that a
service described in an SVCB record can be accessed as an oblivious target
using an associated gateway. The service that is queried by the client hosts
one or more target resources.

In order to access the service's target resources obliviously, the client
needs to send encapsulated messages to the gateway resource and the gateway's
key configuration (both of which can be retrieved using the method described in
{{config-fetch}}).

Both the presentation and wire format values for the "ohttp" parameter
MUST be empty.

The "ohttp" parameter can be included in the mandatory parameter
list to ensure that clients that do not support oblivious access
do not try to use the service. Services that mark the "ohttp"
parameter as mandatory can, therefore, indicate that the service might
not be accessible in a non-oblivious fashion. Services that are
intended to be accessed either obliviously or directly
SHOULD NOT mark the "ohttp" parameter as mandatory. Note that since
multiple SVCB responses can be provided for a single query, the oblivious
and non-oblivious versions of a single service can have different SVCB
records to support different names or properties.

The media type to use for encapsulated requests made to a target service
depends on the scheme of the SVCB record. This document defines the
interpretation for the "https" {{SVCB}} and "dns" {{!DNS-SVCB=I-D.draft-ietf-add-svcb-dns}}
schemes. Other schemes that want to use this parameter MUST define the
interpretation and meaning of the configuration.

## Use in HTTPS service records

For the "https" scheme, which uses the HTTPS RR type instead of SVCB,
the presence of the "ohttp" parameter means that the target
being described is an Oblivious HTTP service that is accessible using
the default "message/bhttp" media type {{OHTTP}}
{{!BINARY-HTTP=RFC9292}}.

For example, an HTTPS service record for svc.example.com that supports
an oblivious gateway could look like this:

~~~
svc.example.com. 7200  IN HTTPS 1 . ( alpn=h2 ohttp )
~~~

A similar record for a service that only supports oblivious connectivity
could look like this:

~~~
svc.example.com. 7200  IN HTTPS 1 . ( mandatory=ohttp ohttp )
~~~

## Use in DNS server SVCB records

For the "dns" scheme, as defined in {{DNS-SVCB}}, the presence of
the "ohttp" parameter means that the DNS server being
described has a DNS over HTTP (DoH) {{!DOH=RFC8484}} service that can
be accessed using Oblivious HTTP. Requests to the resolver are sent to
the oblivious gateway using binary HTTP with the default "message/bhttp"
media type {{BINARY-HTTP}}, containing inner requests that use the
"application/dns-message" media type {{DOH}}.

If the "ohttp" parameter is included in an DNS server SVCB record,
the "alpn" MUST include at least one HTTP value (such as "h2" or
"h3").

In order for DoH servers to function as oblivious targets, their
associated gateways need to be accessible via an oblivious relay.
DoH servers used with the discovery mechanisms described
in this section can either be publicly accessible, or specific to a
network. In general, only publicly accessible DoH servers will work
as oblivious targets, unless there is a coordinated deployment
with an oblivious relay that is also hosted within a network.

### Use with DDR {#ddr}

Clients can discover that a DoH server support Oblivious HTTP using
DDR, either by querying _dns.resolver.arpa to a locally configured
resolver or by querying using the name of a resolver {{DDR}}.

For example, a DoH service advertised over DDR can be annotated
as supporting oblivious resolution using the following record:

~~~
_dns.resolver.arpa  7200  IN SVCB 1 doh.example.net (
     alpn=h2 dohpath=/dns-query{?dns} ohttp )
~~~

Clients still need to perform some verification of oblivious DoH servers,
such as the TLS certificate check described in {{DDR}}. This certificate
check can be done when looking up the configuration on the gateway
as described in {{config-fetch}}, which can either be done directly,
or via the relay or another proxy to avoid exposing client IP addresses.
Since the oblivious gateway that is discovered dynamically uses a well-known
URI on the same host as the target, as described in {{config-fetch}}, the
certificate evaluation for the connection to well-known gateway URI also
covers the name of the target DoH server.

Opportunistic discovery {{DDR}}, where only the IP address is validated,
SHOULD NOT be used in general with oblivious HTTP, since this mode
primarily exists to support resolvers that use private or local IP
addresses, which will usually not be accessible when using an oblivious
relay. If a configuration occurs where the resolver is accessible, but
cannot use certificate-based validation, the client needs to ensure
that the oblivious relay only accesses the gateway and target using
the unencrypted resolver's original IP address.

For the case of DoH servers, clients also need to ensure that they are not
being targeted with unique DoH paths that would reveal their identity. See
{{security}} for more discussion.

### Use with DNR {#dnr}

The SvcParamKeys defined in this document also can be used with Discovery
of Network-designated Resolvers (DNR) {{!DNR=I-D.draft-ietf-add-dnr}}. In this
case, the oblivious configuration and path parameters can be included
in DHCP and Router Advertisement messages.

While DNR does not require the same kind of verification as DDR, clients
that learn about DoH servers still need to ensure that they are not being
targeted with unique DoH paths that would reveal their identity. See {{security}}
for more discussion.

# Gateway Location {#gateway-location}

Clients that know a service is available as an oblivious target
via discovery through the "ohttp" parameter in a SVCB or HTTPS
record need to know the location of the associated oblivious gateway
before sending oblivious requests.

By default, the oblivious gateway for an oblivious target is defined
as a well-known resource ({{WELLKNOWN}}) on the target,
"/.well-known/ohttp-gateway".

Commonly, servers will not want to actually operate the oblivious gateway
on a well-known URI. In such cases, servers can use 3xx redirection responses
({{Section 15.4 of !HTTP=RFC9110}}) to direct clients and relays to the correct
location of the oblivious gateway. Such redirects would apply both to requests
made to fetch key configurations (as defined in {{config-fetch}}) and to
oblivious requests made via an oblivious relay.

If a client receives a redirect when fetching the key configuration from the
well-known gateway resource, it MUST NOT communicate the redirected
gateway URI to the oblivious relay as the location of the gateway to use.
Doing so would allow the oblivious gateway to target clients by encoding
unique or client-identifying values in the redirected URI. Instead,
relays being used with dynamically discovered gateways MUST use the
well-known gateway resource and follow any redirects independently of
redirects that clients received. The relay can remember such redirects
across oblivious requests for all clients in order to avoid added latency.

# Key Configuration Fetching {#config-fetch}

Clients also need to know the key configuration of an oblivious gateway before
sending oblivious requests.

In order to fetch the key configuration of an oblivious gateway discovered
in the manner described in {{gateway-location}}, the client issues a GET request
to the URI of the gateway specifying the "application/ohttp-keys" ({{OHTTP}})
media type in the Accept header.

For example, if the client knows an oblivious gateway URI,
"https://svc.example.com/.well-known/ohttp-gateway", it could fetch the
key configuration with the following request:

~~~
GET /.well-known/ohttp-gateway HTTP/1.1
Host: svc.example.com
Accept: application/ohttp-keys
~~~

Oblivious gateways that coordinate with targets that advertise oblivious
support SHOULD support GET requests for their key configuration in this
manner, unless there is another out-of-band configuration model that is
usable by clients. Gateways respond with their key configuration in the
response body, with a content type of "application/ohttp-keys".

Clients can either fetch this key configuration directly, or do so via
a proxy in order to avoid the server discovering information about the
client's identity. See {{security}} for more discussion of avoiding key
targeting attacks.

# Security and Privacy Considerations {#security}

Attackers on a network can remove SVCB information from cleartext DNS
answers that are not protected by DNSSEC {{?DNSSEC=RFC4033}}. This
can effectively downgrade clients. However, since SVCB indications
for oblivious support are just hints, a client can mitigate this by
always checking for oblivious gateway configuration {{config-fetch}}
on the well-known gateway location {{gateway-location}}.
Use of encrypted DNS along with DNSSEC can also be used as a mitigation.

When clients fetch a gateway's configuration ({{config-fetch}}),
they can expose their identity in the form of an IP address if they do not
connect via a proxy or some other IP-hiding mechanism. In some circumstances,
this might not be a privacy concern, since revealing that a particular
client IP address is preparing to use an Oblivious HTTP service can be
expected. However, if a client is otherwise trying to hide its IP
address or location (and not merely decouple its specific requests from its
IP address), or if revealing its IP address facilitates key targeting attacks
(if a gateway service uses IP addresses to associate specific configurations
with specific clients), a proxy or similar mechanism can be used to fetch
the gateway's configuration.

When discovering designated oblivious DoH servers using this mechanism,
clients need to ensure that the designation is trusted in lieu of
being able to directly check the contents of the gateway server's TLS
certificate. See {{ddr}} for more discussion, as well as the Security
Considerations of {{DNS-SVCB}}.

## Key Targeting Attacks

As discussed in {{OHTTP}}, client requests using Oblivious HTTP
can only be linked by recognizing the key configuration. In order to
prevent unwanted linkability and tracking, clients using any key
configuration discovery mechanism need to be concerned with attacks
that target a specific user or population with a unique key configuration.

There are several approaches clients can use to mitigate key targeting
attacks. {{?CONSISTENCY=I-D.ietf-privacypass-key-consistency}} provides an analysis
of the options for ensuring the key configurations are consistent between
different clients. Clients SHOULD employ some technique to mitigate key
targeting attacks, such as the option of confirming the key with a shared
proxy as described in {{CONSISTENCY}}. Oblivious gateways that are detected
to use targeted key configurations per-client MUST NOT be used.

## dohpath Targeting Attacks

For oblivious DoH servers, an attacker could use unique `dohpath` values
to target or identify specific clients. This attack is very similar to
the generic OHTTP key targeting attack described above.

Clients SHOULD mitigate such attacks. This can be done with a
check for consistency, such as using a mechanism described in {{CONSISTENCY}}
to validate the `dohpath` value with another source. It can also be
done by limiting the the allowable values of `dohpath` to a single
value, such as the commonly used "/dns-query{?dns}".

# IANA Considerations {#iana}

## SVCB Service Parameter

IANA is requested to add the following entry to the SVCB Service Parameters
registry ({{SVCB}}).

| Number  | Name           | Meaning                            | Reference       |
| ------- | -------------- | ---------------------------------- | --------------- |
| TBD     | ohttp          | Denotes that a service operates an Oblivious HTTP target  | (This document) |

## Well-Known URI

IANA is requested to add one new entry in the "Well-Known URIs" registry {{WELLKNOWN}}.

URI suffix: ohttp-gateway

Change controller: IETF

Specification document: This document

Status: permanent

Related information: N/A

--- back
