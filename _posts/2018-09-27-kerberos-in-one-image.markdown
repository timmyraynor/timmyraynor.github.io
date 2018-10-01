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
## Key Exchange in Kerberos
Kerberos, in my understanding, is to establish a trust connection between an authenticated user and a existing service in the realm:

1. Are you a trusted user?
2. Is the service a trusted service?
3. How to build a session based connection channel between the user and the service?

## Entire Flow
![Automation Stack]({{ "/assets/kerberos-flow.png" | absolute_url }})

As you could see from the above flow, the pattern is quite simple.

- Intermediate services (Authenitcation Service / Ticketing Service) is holding the trust credentials of both parties who are requesting for communication.
- When one side initiated the request, the intermediate service would generate the session key, and then package up & send messages that encrypted for both parties, each party would have their key to unpack the package that holds the right session key.
- Both parties using the new session key to communicate.