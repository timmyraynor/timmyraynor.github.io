---
layout: single
title:  "Kerberos All-In-One Image"
date:   2018-09-27 17:23:12 +0000
categories: 
  - Kerberos
  - Security
tags:
  - Kerberos
  - Security
  - KDC
comments: true
---
## 1 Key Exchange in Kerberos
Kerberos, in my understanding, is to establish a trust connection between an authenticated user and a existing service in the realm:

1. Are you a trusted user?
2. Is the service a trusted service?
3. How to build a session based connection channel between the user and the service?

## 2 Entire Flow
![Automation Stack]({{ "/assets/kerberos-flow.png" | absolute_url }})

* Note: the colour of the key is matching with the lock on the message, meaning who ever owns the right colour key could unpack the *locked up* message with the same colour.

As you could see from the above flow, the pattern is quite simple.

1. Intermediate services (Authenitcation Service / Ticketing Service) is holding the trust credentials of both parties requested for this communication.
2. Key exchange starts with:
    - One side initiated the request
    - The intermediate service would generate the session key, and then package up & send messages that encrypted for both parties.
    - Each party would have their key to unpack the package that holds the right session key.
3. Both parties using the new session key to communicate.

### 2.1 User to TGS Server VIA Authentication Server
As we summarized from above, you could find the redline drawed top down will include the separation mentioned in *key exchange solution*:

![Automation Stack]({{ "/assets/kerberos-pattern1.png" | absolute_url }})

- *One side initiated the request*: User initiate the request and ask for authentication, server authenticated user with the password hash encrypted message (noted: no password is transferred during this scope, password becomes the seed of a hash to encrypt the message, and only user with the same password could then decrypted the message. This however could also fit in the pattern of public and private key pairs). 
- *Intermediate service generate the session key for communication*: The generated session key will be packaged up and put into 2 packages, and both user and TGS server could each decrypt one of them.
- *Each party wourld have their key to unpack*: the flow would look like below
    - One package for user to decrypted and then get the session key
    - One package for the TGS to decrypted and then get the session key, this package will travel through the user but user can't decrypted it, user will send the same package without touching it to the TGS server + its own authenticator
    - TGS get the request and decrypt the package with its key, get the session key and decrypt the user's authenticator message

### 2.2 User to Service VIA TGS Server
As we summarized from above, you could find the redline drawed top down will include the separation mentioned in *key exchange solution*:

![Automation Stack]({{ "/assets/kerberos-pattern2.png" | absolute_url }})

- *One side initiated the request*: User already trusted by Authentication Server, TGS server will get the requested Service Name
- *Intermediate service (TGS) generate the session key for communication*: The generated session key will be packaged up and put into 2 packages, and both user and Http Service could each decrypt one of them.
- *Each party wourld have their key to unpack*: the flow would look like below
    - One package for **User** to decrypted and then get the session key
    - One package for the **Service** to decrypted and then get the session key, this package will travel through the user but user can't decrypted it, user will send the same package without touching it to the Service + its own authenticator
    - **Service** get the request and decrypt the package with its key, get the session key and decrypt the user's authenticator message, session key now has distributed into both side, they could then start communication with the session key