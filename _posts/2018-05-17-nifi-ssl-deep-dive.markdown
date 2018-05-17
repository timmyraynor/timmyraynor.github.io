---
layout: post
title:  "NiFi Cluster SSL Deep Dive - How this actually works"
date:   2018-05-16 05:53:12 +0000
categories: NiFi
comments: true
---

# NiFi Cluster SSL
By default NiFi does not require any authentication & authorization, so user could just hit the url and do whatever they like. But when Authentication & Authorization (the *A&A*) are required for your NiFi component, the first thing we usually hit is NiFi SSL and NiFi CA (or self-signed certificates / company CA). Even with NiFi LDAP integration, you have to turn on NiFi SSL to enable NiFi LDAP authentication.

This usually lead to 2 problems in the cluster A&A schema:

- How to auth an entity (e.g. user/node)
- Who can do what  ------ this will be fully discussed in a separate post of the `users.xml / authentications.xml`, here is a light touch of how nodes can work with each other.

## Authentication Schema
NiFi CA provide a self-signed schema to authenticate all NiFi nodes within the cluster. Regardless who's the CA, the entire cluster authentication schema looks like below:
![SSL Auth structure]({{ "/assets/nifi-ca.png" | absolute_url}})
Within the image there are few components:

- NiFi nodes keystore
- NiFi nodes truststore
- CA private key & CA cert

There's already a lot of blogs/threads talking about how to setup NiFi SSL and how SSL/TLS works, here I want to focus on the schema only for bettern understanding, you could find more step-by-step referenes from HCC link below:

- https://community.hortonworks.com/articles/17293/how-to-create-user-generated-keys-for-securing-nif.html

### CA private key & CA cert
If you are doing your own *CA* via self-signed certs, you will see it's quit simple as you generate a private key as the *CA* **private key**, then a **CA cert** with the corresponding **public key** could then be generate and distribute to nodes that we want them to trust the *CA* and certificates signed by the *CA*.

### Keystore: cert about 'Who am I'
NiFi node use the keystore to keep it's private key, and then they could use such private key to generate a CSR(Certificate Signed Request) to ask for CA signature. If the CA signed the request we should be able to get a certificate and use such cert to identify *Who am I*.

With the certificate imported into the same Keystore, NiFi node will then use this cert to identify itself. If this is setup via NiFi CA and you have the *NiFi CA DN Prefix* and *NiFi CA DN Suffix* setup. e.g.

    NiFi CA DN Prefix : CN=
    NiFi CA DN Suffix : ,OU=NiFi

The DN in your certificate will be automatically generate following the pattern of:

    <DN Prefix> + <node FQDN> + <DN Suffix>

Which means the beginning `,` in the Suffix is also important. As each node has their own FQDN (Full Qualified Domain Name), the generated certificate would be different for each node.

### Truststore: cert about 'Who I trust'
When NiFi node trying to communicate with other nodes (within the cluster or outside the cluster, e.g. NiFi site-to-site), the DN of the cert in the node's keystore will be recognized as it's ID. 

But how other nodes trust this is the real ID? That's when the *truststore* comes into play. 

As the RootCA (e.g. NiFi CA) certificate is imported into the *truststore* in the every node within the cluster (if outside the cluster like NiFi site-to-site over SSL, the other NiFis' CA certificate needs to be imported into *truststore* as well). Nodes holding the CA signed certificates will be trusted as well.

From the schema above, each node will know that other nodes with DN names like: `CN=nifiXX.exampledomian.com, OU=NiFi` are certificated by the RootCA which is trusted in the truststore.

### Supported Authentication Schemas
I really cannot remember where I got this reference image but please refer to this image as all the support schemas of authentication:
![SSL Auth Schema]({{ "/assets/authentication-schema.png" | absolute_url}})

As you could see, if we setup the NiFi node to be username / password login, LDAP could be then integrated following such schema. But then what's the ID looks like? 

If you login via LDAP, then no certificate would be used to do the authentication, NiFi and LDAP would communicate their own trust schema. But user will still get a DN name after LDAP login so that that could be used to identify who are you and associate access control policies to this ID by NiFi.

## Authorizations
After NiFi successfully identify each Distinguish Name, you may notice that nodes or users are not that distinguished within NiFi:

- they all have a certificate to identify themselves
- they all represent as an ID (usually the DN from certificates) entity.

So how we enable a cluster to let nodes within the cluster trust each other? 

NiFi resolve this challenges by specifying the 2 initial identities (in *authorizers.xml*)to bootstrap the cluster:

- Initial Admin Identity: which allows a specified ID name as the initial administrator to control and update all followup changes
- Node Identities: which specifies what are the ID names are actually representing nodes within the cluster

After we specified the *Initial Admin Identity* and *Node Identities*, NiFi will then generate the initial *authorizations.xml* and *users.xml* files to **specify these init users** (in *users.xml* file) and **what access do they have** (in *authoriaztions.xml*).

> **NOTE**: Please be aware that the *authorizations.xml* file and *users.xml* files are generated from the *authorizers.xml* file. But NiFi will not regenerate them if these files exist as they will be used to store further polices and users if you add new users and access policies.