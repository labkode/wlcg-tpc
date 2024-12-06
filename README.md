# Proposal to include client checksum verification in the HTTP TPC WLCG API
specification

# Purpose
WLCG transfers have been primarily dominated by the XRootD protocol[1]. This
protocol provided out the box checksum negotation between client and server and
client checksum verification to ensure maximum safety when writing a file.
With the recent push to move to HTTP protocol as the transfer protocol within
WLCG, this safeguard was lost and left to implementations of the HTTP-TPC API
to be creative and come up with their own solutions (at best); or not come at
all to any solution and opening the door to data corruption.

This proposal adds client-provided checksum based on https://www.rfc-editor.org/rfc/rfc9530.html.

In HTTP TPC Pull mode the following interactions happen.

Rather than proposing next HTTP headers to include checksum verification, this
proposals builts on the https://www.rfc-editor.org/rfc/rfc9530.html that
provides all the necessary semantics to negotiate and transmit full digests
(full file content) and partial digests (byte-range requests).

The [RFC9530] specifics two headers for trasnmiting a digest (a hash of the
file contents).
- Repr-Digest: contains the hash of the full file binary contents, transmitted
in the HTTP body (independenly if only a few bytes are requested, for example,
with Range-Bytes header).
- Content-Digest: contains the hash of the HTTP body, for our purposes, the
hash of the contents transmitted in the HTTP (can be partial or total).


When a client sends a submit requests to FTS, the client can send its
precomputed file checksum alongside other metadata that the transfer tool is
willing to accept. The transmission of the checksum to the trasnfer tool is
transfer tool specific and beyond the specification in this document. However
for illustration purposes, in FTS, the checksum will be submitted in the
payload for submission job:

*Example of FTS3 submission file*
```
      {
        "files": [
          {
            "sources": ["root://source/file"],
            "destinations": ["root://dest/file"],
            ...
            "filesize": 1024,
---->       "checksum": 'adler32:1234',
          }
        ],
        "params": {
         ...
        }
      }
```

The transfer tool will receive the client checksum.
The trasnfer toll will then initiate a 3rd party COPY.
In this example, we use the PULL model.

The HTTP COPY is sent to the ACTIVE site (the one pulling the data from the
source).
The HTTP COPY will containt the following HTTP Headers:

```
     COPY /event.root
     Host: activesite.cern.ch
     Source: pasivesite.cern.ch
---> X-Digest-Behaviour: enum (ABORT, PASS)
---> TransferHeaderRepre-Digest: adler=:1234:
     ... (elements ommited)
```

The ACTIVE site will then issue a GET request to PASIVE site to obtain the file.
The ACTIVE site will write the file and then it must compute the checkusm with
the specific digest algorithm included in the HTTP request. If the ACTIVE site
cannot use the specific digest algorithm, the behaviour is dominated by the
X-Digest-Behaviour, which can contain for now only two case insensitive values: ABORT
or PASS.
If the specified behaviour is PASS, the file is clasified as valid.
This mode is necessary to ensure that checksum
verification can work with sites not supporting yet this specification.
If the value is ABORT, the ACTIVE site should mark the file invalid and
this file should not be visible in the requested location. Implementors may delete the file or
put it ina quarantine area, it's up to them. The constraint here is that the file cannot be visible to clients
until its checksum has been verified.

# Scope

# Objective