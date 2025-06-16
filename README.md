# DATA INTEGRITY IN WLCG TPC

WLCG TPC is a specification created by the WLCG Data Transfers working group
to optimize data movements across network hosts without having to use a middleware
proxy. The specificaton builds on top of WebDAV to provide a mechanism to perform
peer to peer data transfers across the network.

The specification describes some integrity mechanisms and this proposal enhances that
initial work with up to date standards and clear description of the interactions between
hosts and the expectations when performing the data transfer (HTTP request).

The terms data integrity and checksum used interchangeably in this text.


Refer to this [slideck](https://indico.cern.ch/event/1538346/contributions/6474197/attachments/3053366/5397622/DOMA%20BDT%20TPC%20DATA%20INTEGRITY%20(1).pdf
) for a summary (access is protected with CERN login):

Implementors should follow the [RFC9530](https://datatracker.ietf.org/doc/rfc9530/) text for implementing the new headers.
There is one addition to the spec to facilitate the adoption of the standard.
The standard defines the word "adler" to refer to the digest adler32, however, in WLCG, the word "adler32" is widely used already.
Implementors must consider the words "adler" and "adler32" to refer to the same digest.

## NEW DATA INTEGRITY BEHAVIOUR IN TPC
The new behaviour is enabled when sending the HTTP header `Repr-Digest` in the COPY request independently if the TPC is push or pull mode.
```
     COPY /event.root
     Host: activesite.cern.ch
     Source: pasivesite.cern.ch
---> Repr-Digest: adler=:1234:
     ... (elements ommited)
```

### Push mode
The HTTP COPY is sent to the ACTIVE site that will send (PUT) data into the PASIVE site.
The HTTP COPY will containt the following HTTP Headers:
```
     COPY /event.root
     Host: activesite.cern.ch
     Destination: pasivesite.cern.ch
---> Repr-Digest: adler=:1234:
     ... (elements ommited)
```

The ACTIVE site will issue a PUT request with the same header (`Repr-Digest: adler=:1234:`) to the PASIVE site.
The PASIVE site is expeced to verify that the provided checksum matches the one of the new saved file.

```
                                                                                                     
                                                                                                     
                                                                                                     
                        Perf makers     ┌────┐                                                       
                        .....           │ 4  │                                                       
       ┌────────────────────────────────┴────┴───────────────────────┐                               
       │                Ok                                           │                               
       ▼         ┌────┐                                              │                               
┌─────────────┐  │ 1  │  COPY /event.root                            │                               
│             │  └────┘  Host: activesite.cern.ch                    │                               
│   CLIENT    ├──────────Destination: pasivesite.cern.ch             │                               
│             │          Repr-Digest: adler:1234:             ┌────────────┐                         
└─────────────┘                      │                        │            │                         
                                     └───────────────────────▶│   ACTIVE   │◀─────────────────┐      
                                                              │            │                  │      
                                                              └────────────┘                  │      
                                                                     │                        │      
                                                        PUT /event.root                       │      
                                                        Host: pasivesite.cern.ch              │      
                                                        Repr-Digest: adler:1234:         201      
                                                                     └───────────────┐        │      
                                                                             ┌────┐  │        │      
                                                                             │ 2  │  │        │┌────┐
                                                                             └────┘  │        ││ 3  │
                                                                                     │        │└────┘
                                                                                     ▼        │      
                                                                              ┌────────────┐  │      
                                                                              │            │  │      
                                                                              │   PASIVE   │──┘      
                                                                              │            │         
                                                                              └────────────┘         
```


If the checksum is different the HTTP error code 412 (Pre-Condition Failed) MUST be returned to the ACTIVE site.
The ACTIVE site will report the checksum error to the client via the usual performance marker open connection. The error provided to the client
SHOULD be a human readable text explaining that the file coundn't be saved because of client-server checksum mismatch.

```
                                                                                                     
                                                                                                     
                                                                                                     
                  Perf makers                                                                        
                  .....                                                                              
                                                                                                     
                  Error: client/server checksum mismatch                                             
       ┌───────────────────────────────────────────────────────┐                                     
       │                                                       │                                     
       ▼          ┌────┐                                       │┌────┐                               
┌─────────────┐   │ 1  │ COPY /event.root                      ││ 4  │                               
│             │   └────┘ Host: activesite.cern.ch              │└────┘                               
│   CLIENT    ├──────────Destination: pasivesite.cern.ch       │                                     
│             │          Repr-Digest: adler:1234:              │                                     
└─────────────┘                   │                     ┌────────────┐                               
                                  │                     │            │                               
                                  └────────────────────▶│   ACTIVE   │◀───────────────────────┐      
                                                        │            │                        │      
                                                        └────────────┘                        │      
                                                         ┌────┐│                            412      
                                                         │ 2  ││                              │      
                                                         └────┘│                              │      
                                                       PUT /event.root                        │      
                                                       Host: pasivesite.cern.ch──────┐        │      
                                                       Repr-Digest: adler:1234:      │        │      
                                                                                     │        │      
                                                                                     │        │      
                                                                                     ▼        │┌────┐
                                                                              ┌────────────┐  ││ 3  │
                                                                              │            │  │└────┘
                                                                              │   PASIVE   │──┘      
                                                                              │            │         
                                                                              └────────────┘
```

### Pull mode
The HTTP COPY is sent to the ACTIVE site that will get (GET) data from the PASIVE site.
The HTTP COPY will containt the following HTTP Headers:
```
     COPY /event.root
     Host: activesite.cern.ch
     Source: pasivesite.cern.ch
---> Repr-Digest: adler=:1234:
     ... (elements ommited)
```

The ACTIVE site will issue a GET request with the  header `Want-Repr-Digest` (example: `Want-Repr-Digest: adler=9`) to the PASIVE site.
The PASIVE site will return the file contents with its checksum in the header `Repr-Digest` (example: `Repr-Digest: adler=:1234:`).

```
                                                                                                                 
                                                                                                                 
                                                                                                                 
                        Perf makers     ┌────┐                                                                   
                        .....           │ 4  │                                                                   
       ┌────────────────────────────────┴────┴───────────────────────┐                                           
       │                Ok                                           │                                           
       ▼         ┌────┐                                              │                                           
┌─────────────┐  │ 1  │  COPY /event.root                            │                                           
│             │  └────┘  Host: activesite.cern.ch                    │                                           
│   CLIENT    ├──────────Source: pasivesite.cern.ch                  │                                           
│             │          Repr-Digest: adler:1234:             ┌────────────┐                                     
└─────────────┘                      │                        │            │                                     
                                     └───────────────────────▶│   ACTIVE   │◀─────────────────┐                  
                                                              │            │                  │                  
                                                              └────────────┘                  │                  
                                                                     │                        │                  
                                                        GET /event.root               HTTP/1.1 200 OK            
                                                        Host: pasivesite.cern.ch      Repr-Digest: adler=:1234:  
                                                        Want-Repr-Digest: adler=9     ... binary data ...        
                                                                     └───────────────┐        │                  
                                                                             ┌────┐  │        │                  
                                                                             │ 2  │  │        │┌────┐            
                                                                             └────┘  │        ││ 3  │            
                                                                                     │        │└────┘            
                                                                                     ▼        │                  
                                                                              ┌────────────┐  │                  
                                                                              │            │  │                  
                                                                              │   PASIVE   │──┘                  
                                                                              │            │                     
                                                                              └────────────┘                     
```

The ACTIVE site will compute the checksum of the receiving file and compare it to the one obtained from the PASIVE site.
If the checksum is different, the  ACTIVE site will report the checksum error to the client via the usual performance marker open connection.
The error provided to the client SHOULD be a human readable text explaining that the file coundn't be saved because of client-server checksum mismatch.

```
                                                                                                                   
                                                                                                                   
                                                                                                                   
Perf makers                                                                                                        
.....                                                                                                              
                                          ┌────┐                                                                   
Error: client/server checksum mismatch    │ 4  │                                                                   
         ┌────────────────────────────────┴────┴───────────────────────┐                                           
         │                                                             │                                           
         ▼         ┌────┐                                              │                                           
  ┌─────────────┐  │ 1  │  COPY /event.root                            │                                           
  │             │  └────┘  Host: activesite.cern.ch                    │                                           
  │   CLIENT    ├──────────Source: pasivesite.cern.ch                  │                                           
  │             │          Repr-Digest: adler:1234:             ┌────────────┐                                     
  └─────────────┘                      │                        │            │                                     
                                       └───────────────────────▶│   ACTIVE   │◀─────────────────┐                  
                                                                │            │                  │                  
                                                                └────────────┘                  │                  
                                                                       │                        │                  
                                                          GET /event.root               HTTP/1.1 200 OK            
                                                          Host: pasivesite.cern.ch      Repr-Digest: adler=:999:   
                                                          Want-Repr-Digest: adler=9     ... binary data ...        
                                                                       └───────────────┐        │                  
                                                                               ┌────┐  │        │                  
                                                                               │ 2  │  │        │┌────┐            
                                                                               └────┘  │        ││ 3  │            
                                                                                       │        │└────┘            
                                                                                       ▼        │                  
                                                                                ┌────────────┐  │                  
                                                                                │            │  │                  
                                                                                │   PASIVE   │──┘                  
                                                                                │            │                     
                                                                                └────────────┘                     
```

