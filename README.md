# secrets-manager

![Javascript](https://github.com/Keeper-Security/secrets-manager/actions/workflows/test.js.yml/badge.svg)

Monorepo for secrets manager sdk's and tools



For structure, using guidance from https://medium.com/interstellar/monorepo-e1847cbd6800

To test ci locally, install https://github.com/nektos/act and run ./test_ci.sh

### Overview

Secrets Manager is an API hosted by Keeper, and a set of SDK's that consume the API.
It is designed to allow sharing the secrets (passwords, tokens, keys, etc.) with various devices, 
physical and virtual, in a zero-knowledge way. Secrets are either a single Keeper records, or a Keeper Shared Folders
containing a set of records. Only typed records can be shared via Secrets Manager, and only typed records within
shared folders are returned to the client.

Secrets Manager uses password-less authentication, relying on digital signature to authenticate clients.
The client's private key is generated by the SDK, and must be stored securely by the client. 

### API

All API calls take a key value storage implementation as a parameter.
The key value storage is used to persist the data generated by the SDK (keys, cached data, etc.)
The key value storage implementation needs to be secure - keys compromise will allow an attacker to 
access the secret shared with the device.

#### getSecrets
Takes a key value storage implementation and returns Keeper Secret as JSON object. 
It is a client responsibility to parse the JSON and use the content as intended. 

### Authentication protocol

Client is given a one time "Secret Key" which is the encryption key for the Application Master Key.

The Secret Key must be securely passed from the secret owner to the client in some way that does not involve Keeper
to preserve zero knowledge.

Client uses the SHA256 hash of the Secret Key to obtain the permanent Client ID.

On the first call to the server, client is authenticated by using the Client ID only.

On the subsequent calls, client is using the digital signature to authenticate.

Client sends public key until receives encrypted master key from server, then it stops sending the public key.

Server keeps sending encrypted master key until receives request from client with no public key.

#### Detailed steps:  

1. Secret owner creates an "Application" by generating an "Application Master Key" and storing it within Keeper Typed Record.  
1. Secret owner encrypts the secret key (shared folder or record key) with the master key and attaches the share to the application.
1. Secret owner generates a random 256-bit **Secret Key** (SK) and calculates an SHA-256 hash of that key (**Client ID** or CID).
1. Secret owner encrypts the master key with SK and attaches the "client record" that contains encrypted master key and CID to the application. 
1. Secret owner sends the SK to the client (device or service that needs the secret)
1. Upon reception of SK, the client calls initializeStorage function, which calculates CID by hashing the SK 
   and initializes the key value storage with SK and CID.
1. Client calls getSecrets function of the SDK.
1. SDK generates private/public key pair and sends CID and the public key to Keeper.
1. Keeper verifies that the CID does not have an associated public key yet and responds with an error if 
   the CID has the associated public key, but the request is not signed.
1. Keeper associates the CID with the received public key, and responds with the encrypted master key and attached secrets.
1. The SDK uses the SK to decrypt the master key and stores it in the key value storage.
1. The SDK decrypts the received secrets using the master key and returns the decrypted data to the client.
1. On the next call, the SDK uses the private key to sign the request.
1. When Keeper receives a signed request it checks if the signature matches the associated public key and responds 
    with an error if the signature does not match.
1. When Keeper receives the request that does not contain a public key, it erases the encrypted master key from 
    the client record to prevent the SK from ever be reused.
    At this point, the client is "bound" to the server. The client is in control of the keys and can request the secret 
    again at any time if the sharing policy permits it.
    
 





