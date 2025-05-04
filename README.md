# PROPOSAL TO ENHANCE DATA INTEGRITY THE HTTP TPC WLCG SPEC

WLCG TPC is a specification created by the WLCG Data Transfers working group
to optimize data movements across network hosts without having to use a middleware
proxy. The specificaton builds on top of WebDAV to provide a mechanism to perform
peer to peer data transfers across the network.

The specification describes some integrity mechanisms and this proposal enhances that
initial work with up to date standards and clear description of the interactions between
hosts and the expectations when performing the data transfer (HTTP request).


Refer to this [slideck](https://indico.cern.ch/event/1538346/contributions/6474197/attachments/3053366/5397622/DOMA%20BDT%20TPC%20DATA%20INTEGRITY%20(1).pdf
) for a summary (access is protected with CERN login):

Implementors should follow the RFC9530 text for implementing the new headers.
There is one addition to the spec to facilitate the adoption of the standard.
The standard defines the word "adler" to refer to the digest adler32, however, in WLCG, the word "adler32" is widely used already.
Implementors must consider the words "adler" and "adler32" to refer to the same digest.

## NEW DATA INTEGRITY BEHAVIOUR IN TPC
The new behaviour is enabled when sending the HTTP header `Content-Digest` in the COPY request independently if the TPC is push or pull mode.
```
     COPY /event.root
     Host: activesite.cern.ch
     Source: pasivesite.cern.ch
---> Content-Digest: adler=:1234:
     ... (elements ommited)
```

### Push mode
The HTTP COPY is sent to the ACTIVE site that will send (PUT) data into the PASIVE site.
The HTTP COPY will containt the following HTTP Headers:
```
     COPY /event.root
     Host: activesite.cern.ch
     Destination: pasivesite.cern.ch
---> Content-Digest: adler=:1234:
     ... (elements ommited)
```

The ACTIVE site will issue a PUT request with the same header (`Content-Digest: adler=:1234:`) to the PASIVE site.
The PASIVE site is expeced to verify that the provided checksum matches the one of the new saved file.
If the checksum is different the HTTP error code 412 (Pre-Condition Failed) MUST be returned to the ACTIVE site.
The ACTIVE site will report the checksum error to the client via the usual performance marker open connection. The error provided to the client
SHOULD be a human readable text explaining that the file coundn't be saved because of client-server checksum mismatch.

### Pull mode
The HTTP COPY is sent to the ACTIVE site that will get (GET) data from the PASIVE site.
The HTTP COPY will containt the following HTTP Headers:
```
     COPY /event.root
     Host: activesite.cern.ch
     Source: pasivesite.cern.ch
---> Content-Digest: adler=:1234:
     ... (elements ommited)
```

The ACTIVE site will issue a GET request with the  header `Want-Content-Digest` (example: `Want-Content-Digest: adler=9`) to the PASIVE site.
The PASIVE site will return the file contents with its checksum in the header `Content-Digest` (example: `Content-Digest: adler=:1234:`).

The PASIVE site is expeced to verify that the provided checksum matches the one of the new saved file.
If the checksum is different the HTTP error code 412 (Pre-Condition Failed) MUST be returned to the ACTIVE site.
The ACTIVE site will report the checksum error to the client via the usual performance marker open connection. The error provided to the client
SHOULD be a human readable text explaining that the file coundn't be saved because of client-server checksum mismatch.


---


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
