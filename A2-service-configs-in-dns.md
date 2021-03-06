Service Config via DNS
----------------------
* Author(s): Mark D. Roth (roth@google.com)
* Approver: a11r
* Status: Implemented
* Implemented in: C-core (Go and Java in progress)
* Last updated: 2017-09-13
* Discussion at: https://groups.google.com/d/topic/grpc-io/DkweyrWEXxU/discussion

## Abstract

This document proposes a mechanism for encoding gRPC service config data
in DNS for use in the open-source world.

## Background

The [service
config](https://github.com/grpc/grpc/blob/master/doc/service_config.md)
mechanism was originally designed for use inside of Google.  However, all
but one part of the original design works fine in the open-source world.
That one part is the specification of how the service config data will
be encoded in DNS.  This proposal fills in this missing piece.

### Related Proposals: 

N/A

## Proposal

There are two parts to this proposal.  The first part is to add some
JSON wrapping for controlling how service config changes are canary
tested.  The second part describes how the service config is encoded in
DNS.

### Canarying Changes

When deploying a change to a service config, it is useful to be able to
canary test changes to avoid wide-spread breakage by slowly increasing the
number of clients that see the new version.  To that end, multiple
service configs choices can be listed, in order, along with criteria that
determine which choice will be selected by a given client:

```
// A list of one or more service config choices.
// The first matching entry wins.
[
  {
    // Criteria used to select this choice.
    // If a field is absent or empty, it matches all clients.
    // All fields must match a client for this choice to be selected.
    //
    // Client language(s): a list of strings (e.g., 'c++', 'java', 'go',
    // 'python', etc).
    'clientLanguage': [string],
    // Percentage: integer from 0 to 100 indicating the percentage of
    // clients that should use this choice.
    'percentage': number,
    // Client hostname(s): a list of strings.
    'clientHostname': [string],

    // The service config data object for clients that match the above
    // criteria.  (The format for this object is defined in
    // https://github.com/grpc/grpc/blob/master/doc/service_config.md.)
    'serviceConfig': object
  }
]
```

### Encoding in DNS TXT Records

In DNS, the service config data (in the form documented in the previous
section) will be encoded in a TXT record via the mechanism described in
[RFC-1464](https://tools.ietf.org/html/rfc1464) using the attribute name
`grpc_config`.  The attribute value will be a JSON list containing service
config choices.  For example, here is an example TXT record:

```
myserver  3600  TXT  "grpc_config=[{'serviceConfig':{'loadBalancingPolicy':'round_robin','methodConfig':[{'name':[{'service':'MyService','method':'Foo'}],'waitForReady':true}]}}]"
```

Note that TXT records are limited to 255 bytes per string, as per
[RFC-1035 section 3.3](https://tools.ietf.org/html/rfc1035#section-3.3).
However, there can be multiple strings, which will be
concatenated together, as described in [RFC-4408 section
3.1.3](https://tools.ietf.org/html/rfc4408#section-3.1.3).  The total
DNS response cannot exceed 65535 bytes.  (See the "Open Issues"
section below for more discussion.)

Note that because TXT records must be ASCII, this also imposes the
restruction that the contents of the service config are also ASCII
(e.g., service and method names, load balancing policy names, etc).

## Rationale

The service config is designed to be returned as part of name
resolution, so encoding it in DNS makes the most sense.  Sites that use
a naming system other than DNS can, of course, implement their own
resolvers with their own mechanism for encoding service config data.

When encoding the service config in DNS, TXT records are the "obvious"
choice, since the service config is effectively additional metadata
associated with the DNS name.

## Implementation

The implementation will be done in C-core first.  Once the new c-ares
DNS resolver code (https://github.com/grpc/grpc/pull/7771) has been
merged, we will extend it to query for the TXT records and return the
resulting service config JSON data.

Note that, due to platform support issues, we will initially *not*
support the c-ares resolver under Windows or for Node.  Alternatives
will need to be found for these environments.

## Open issues (if applicable)

DNS TXT records do have some limitations that need to be taken into
account here.  In particular:

- If a DNS response exceeds 512 bytes, it will fall back from UDP to
  TCP, which adds overhead.
- The total DNS response cannot exceed 65535 bytes.
- It is not clear whether individual DNS implementations will allow
  anywhere close to 65535 bytes, even though the spec says that they
  should.

Feedback is requested on whether these considerations will be a
significant drawback for this design (in which case the design will
probably have to be changed).
