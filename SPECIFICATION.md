# CoRPC

- **This is a draft. It will change in the future, as new features might be added, or already existing features might be modified.**

- CoRPC uses Protocol Buffers for message body encoding. 

- TLS is mandatory. 

When a client connects, it enters [**Authentication**](#authentication-state) state. 
After successful authentication/registration, client will enter [**Command**](#command-state) state.

## Authentication state

Authentication state is designed to be used synchronously. Authentication and registration packets have same packet types,
so it is strongly recommended to always wait for the result.

### Table of contents

1. Packets
    1. [Service description packet](#service-description-packet)
    2. [Authentication packet](#authentication-packet)
    3. [Registration confirmation packets](#registration-confirmation-packets)
    4. [Authentication confirmation packets](#authentication-confirmation-packets)
2. Authentication methods
    1. [Anonymous](#anonymous)
    2. [Credentials](#credentials)
    3. [Token](#token)
    4. [Ed25519 key](#ed25519)

### Service description packet

Server sends this packet right after the client finishes TLS handshake. 
This method describes service, that the server is running.

| Name                          | Type      | Description                                                                        |
|-------------------------------|-----------|------------------------------------------------------------------------------------|
| Magic                         | byte\[5\] | Contains constant string `CoRPC` (`43 6F 52 50 43`)                                |
| Service name length           | uvarint   | Contains size of service name string                                               |
| Service name                  | string    | Service name is a unique string that identifies this service                       |
| Major                         | uint16    | Major version of the service                                                       |
| Minor                         | uint16    | Minor version of the service                                                       |
| Patch                         | uint16    | Patch version of the service                                                       |
| Flags                         | uint16    | Field that contains bit-flags for service, see [Flags](#service-description-flags) |
| Authentication methods length | byte      | Contains the amount of supported authentication methods                            |
| Authentication methods        | byte[]    | Lists all available authentication methods, see [Methods](#authentication-methods) |

#### Service description flags

Bits are counted from least significant to most significant

| Bit   | Name         | Description                                    |
|-------|--------------|------------------------------------------------|
| 0     | Registration | Shows whether registration is supported or not |
| 1..15 | Reserved     | These bits are reserved for future use         |

#### Authentication methods

Custom methods must be added starting from `0xFF` and counting down, 
as to not cause conflicts with methods, that may be introduced in the future.

| Method ID | Name        | Authentication procedure        |
|-----------|-------------|---------------------------------|
| `0x00`    | Anonymous   | See [Anonymous](#anonymous)     |
| `0x01`    | Credentials | See [Credentials](#credentials) |
| `0x02`    | Token       | See [Token](#token)             |
| `0x03`    | Ed25519 key | See [Ed25519](#ed25519)         |

### Authentication packet

| Name                  | Type   | Description                                                   |
|-----------------------|--------|---------------------------------------------------------------|
| Packet type           | byte   | Contains a constant value that describes this packet - `0x00` |
| Authentication method | byte   | Contains selected authentication method                       |
| Flags                 | uint16 | See [Flags](#authentication-flags)                            |
| Method specific data  | byte[] | Data, that depends on the selected authentication method      |

#### Authentication flags

| Bit   | Name         | Description                                    |
|-------|--------------|------------------------------------------------|
| 0     | Registration | Shows whether client registers or logs in      |
| 1..15 | Reserved     | These bits are reserved for future use         |

##### Anonymous

Method specific data is empty

##### Credentials

Method specific data is as follows:

| Name        | Type                       | Description                    |
|-------------|----------------------------|--------------------------------|
| Data length | uint16                     | Contains length of the data    |
| Data        | Protobuf `CredentialsAuth` | Contains protobuf-encoded data |

```protobuf
syntax = "proto3";

message CredentialsAuth {
    string username = 1;
    string password = 2;
    // May be extended with custom fields 
}
```

##### Token

Method specific data depends on whether the client registers or logs in.

###### Registration

Registration does not automatically authenticate user.

Request (method specific data):

| Name        | Type                       | Description                      |
|-------------|----------------------------|----------------------------------|
| Data length | uint16                     | Contains length of the data      |
| Data        | Custom protobuf message    | Contains protobuf-encoded data   |

Response (method specific data):

| Name         | Type   | Description                       |
|--------------|--------|-----------------------------------|
| Token length | uint16 | Contains length of the token      |
| Token        | byte[] | Contains the authentication token |

###### Authentication

Method specific data is as follows:

| Name         | Type   | Description                       |
|--------------|--------|-----------------------------------|
| Token length | uint16 | Contains length of the token      |
| Token        | byte[] | Contains the authentication token |

##### Ed25519

###### Registration

Method specific data is as follows:

| Name         | Type                    | Description                                 |
|--------------|-------------------------|---------------------------------------------|
| Public key   | byte\[32\]              | Ed25519 public key                          |
| Data length  | uint16                  | Length of custom data                       |
| Data         | Custom protobuf message | Additional application-specific information |

###### Authentication

Authentication request (method specific data):

| Name                   | Type       | Description            |
|------------------------|------------|------------------------|
| Public key fingerprint | byte\[32\] | SHA256 hash of the key |

Challenge bytes (raw data):

| Name         | Type   | Description                     |
|--------------|--------|---------------------------------|
| Data length  | uint16 | Size of the challenge           |
| Data         | byte[] | Bytes that the client must sign |

Finalization (raw data):

| Name         | Type       | Description                       |
|--------------|------------|-----------------------------------|
| Signature    | byte\[64\] | Signature for challenge bytes     |

### Registration confirmation packets

#### Successful registration

| Name         | Type   | Description                                      |
|--------------|--------|--------------------------------------------------|
| Packet type  | byte   | Byte that identifies the packet, constant `0x01` |
| Data         | byte[] | Method specific data                             |

#### Failed registration

| Name         | Type   | Description                                          |
|--------------|--------|------------------------------------------------------|
| Packet type  | byte   | Byte that identifies the packet, constant `0x02`     |
| Error code   | uint64 | Local error code (as defined in service description) |

### Authentication confirmation packets

#### Successful authentication

| Name         | Type   | Description                                      |
|--------------|--------|--------------------------------------------------|
| Packet type  | byte   | Byte that identifies the packet, constant `0x01` |
| Data         | byte[] | Application specific data                        |

After successful authentication, connection will switch to [**Command**](#command-state) state.

#### Failed authentication

| Name         | Type   | Description                                          |
|--------------|--------|------------------------------------------------------|
| Packet type  | byte   | Byte that identifies the packet, constant `0x02`     |
| Error code   | uint64 | Local error code (as defined in service description) |



## Command state

### Table of contents:

1. Client to Server methods
    1. [Method call](#method-call-packet-client---server)
    2. [Stream close request](#stream-close-request-client---server)
    3. [Stream data packet](#stream-data-packet-client---server)
2. Server to Client methods
    1. [Method call result](#method-call-result-server---client)
    2. [Method call error](#method-call-error-server---client)
    3. [Event notification](#event-notification-server---client)
    4. [Stream bus setup](#stream-bus-setup-server---client)
    5. [Stream close notification](#stream-close-notification-server---client)
    6. [Stream data packet](#stream-data-packet-server---client)

### Method call packet (Client -> Server)

| Name         | Type     | Description                                                                     |
|--------------|----------|---------------------------------------------------------------------------------|
| Packet type  | byte     | Byte that identifies the packet, constant `0x01`                                |
| Call ID      | uint64   | Identifier of the current call, must be unique for all running calls            |
| Method group | uint16   | Group of the called method                                                      |
| Method ID    | uint16   | ID of the called method inside the group                                        |
| Body length  | uvarint  | Size of the body                                                                |
| Body         | Protobuf | Body of the request, protobuf message type is predefined in service description |

Methods can be called asynchronously. Call ID will be used to link specific call with the result.

### Method call result (Server -> Client)

| Name         | Type     | Description                                                                             |
|--------------|----------|-----------------------------------------------------------------------------------------|
| Packet type  | byte     | Byte that identifies the packet, constant `0x01`                                        |
| Call ID      | uint64   | Identifier of the completed call                                                        |
| Body length  | uvarint  | Size of the body                                                                        |
| Body         | Protobuf | Result of the specific call, protobuf message type is predefined in service description |

### Method call error (Server -> Client)

| Name         | Type     | Description                                                                        |
|--------------|----------|------------------------------------------------------------------------------------|
| Packet type  | byte     | Byte that identifies the packet, constant `0x02`                                   |
| Error code   | uint64   | Local error code (as defined in service description)                               |
| Body length  | uvarint  | Size of the error body                                                             |
| Body         | Protobuf | Error specific details, protobuf message type is predefined in service description |

### Event notification (Server -> Client)

| Name         | Type     | Description                                                                   |
|--------------|----------|-------------------------------------------------------------------------------|
| Packet type  | byte     | Byte that identifies the packet, constant `0x03`                              |
| Event bus ID | uint64   | Unique identifier of the event bus                                            |
| Event ID     | uint64   | Unique identifier of the event inside the bus                                 |
| Body length  | uvarint  | Length of the event body                                                      |
| Body         | Protobuf | Body of the event, protobuf message type is predefined in service description |

### Stream bus setup (Server -> Client)

| Name         | Type    | Description                                                        |
|--------------|---------|--------------------------------------------------------------------|
| Packet type  | byte    | Byte that identifies the packet, constant `0x04`                   |
| Call ID      | uint64  | ID of the associated call                                          |
| Stream ID    | UUID    | UUID of the stream in raw format                                   |
| Stream type  | byte    | Byte that describes stream type, see [Stream types](#stream-types) |
| Stream size  | uvarint | Size of the stream (0 if size is undefined)                        |

- Sized streams are closed implicitly when all the bytes are sent. Unsized streams are closed explicitly. 

- Streams must be initialized before the associated call is finished, but can outlive associated calls.

#### Stream types

| Type ID | Name     | Description                  |
|---------|----------|------------------------------|
| `0x00`  | Incoming | Data is sent to the server   |
| `0x01`  | Outgoing | Data is sent from the server |

### Stream close notification (Server -> Client)

| Name         | Type    | Description                                      |
|--------------|---------|--------------------------------------------------|
| Packet type  | byte    | Byte that identifies the packet, constant `0x05` |
| Stream ID    | UUID    | UUID of the closed stream                        |

### Stream close request (Client -> Server)

| Name         | Type    | Description                                      |
|--------------|---------|--------------------------------------------------|
| Packet type  | byte    | Byte that identifies the packet, constant `0x02` |
| Stream ID    | UUID    | UUID of the closed stream                        |

After this method, server must reply with a stream close notification

### Stream data packet (Server -> Client)

| Name         | Type    | Description                                      |
|--------------|---------|--------------------------------------------------|
| Packet type  | byte    | Byte that identifies the packet, constant `0x06` |
| Stream ID    | UUID    | UUID of associated stream                        |
| Sequence ID  | uint64  | Number of the packet                             |
| Data size    | uvarint | Size of the data                                 |
| Data         | byte[]  | Streamed data                                    |

### Stream data packet (Client -> Server)

| Name         | Type    | Description                                      |
|--------------|---------|--------------------------------------------------|
| Packet type  | byte    | Byte that identifies the packet, constant `0x03` |
| Stream ID    | UUID    | UUID of associated stream                        |
| Sequence ID  | uint64  | Number of the packet                             |
| Data size    | uvarint | Size of the data                                 |
| Data         | byte[]  | Streamed data                                    |