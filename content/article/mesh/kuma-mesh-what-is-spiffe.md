---
title: "What Is Spiffe and How is it used in Kuma Mesh"
date: 2022-10-14
draft: false

categories: ["Service Mesh"]
tags: ["service mesh", "Kong Mesh", "Kuma", "Envoy Proxy", "spiffe"]
toc: false
summary: "**Situation:** One of the first features everyone wants from a Service Mesh is to take advantage of the `Zero Trust / mTLS` that comes right out of the box. When you start to wondering into how this is all feasible in the world of workloads dynamically being created, destroyed, autoscaling the question becomes how does it all work?"
author: "djfreese"
---

**Situation:** One of the first features everyone wants from a Service Mesh is to take advantage of the `Zero Trust / mTLS` that comes right out of the box. When you start to wondering into how this is all feasible in the world of workloads dynamically being created, destroyed, autoscaling the question becomes how does it all work?

In the [Mutual TLS Doc of Kuma Mesh](https://kuma.io/docs/1.8.x/policies/mutual-tls/) it states the following:

`The certificates that Kuma generates have a SAN set to spiffe://<mesh name>/<service name>. When Kuma enforces policies that require an identity like TrafficPermission it will extract the SAN from the client certificate and use it to match the service identity.`

So! What is a `spiffe` and how is Kuma Mesh leveraging it?

## Secure Production Identity Framework for Everyone - Spiffe

Spiffe stands for `Secure Production Identity Framework for Everyone`, and is a part of the CNCF.

A great resource to watch to get the fundamentals is the youtube video linked on the [spiffe.io homepage](https://spiffe.io/).

`Spiffe` is basically a `specification` solving for security of cloud native workloads via `workload identity`. In translation:

In the world of dynamically shifting workloads that may be moving nodes, autoscaling, changing platforms, we need need to be able to secure communication between `Services` (i.e. `workloads`). In other words spiffe solves for the problem of `workload identity`.

### Spiffe Specification contains three main concepts:

**First** - `spiffe id`: in the form of `spiffe://trust domain/workload identifier` which defines workload identity.

**Second** - `SVID`: is used to prove the authenticatity of the spiffe id because the SVID is signed by a CA and can be in the format of an x509 certificate.

**Third** - `Workload API` : to provision the svid and authority.

So how do these concepts apply to Kuma Mesh ?

## How does this concept map to Kuma Mesh?

Well the spiffe.io has a great summary of that:

`Kuma automatically generates SPIFFE-compatible certificates that identify all the services and workloads running in the service mesh, and encrypts all the traffic generated between them`.

To validate this, we can enable mTLS on a a Kuma Mesh, grab the xDS configuration and trace it down. And that is what we will step through below.

### Enable mTLS on the Mesh

mTLS needs to be enabled on the mesh before starting. This can be done by taking advantage of the builtin-CA and just enabling it on the the default mesh:

```yaml
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: builtin
      dpCert:
        rotation:
          expiration: 1d
      conf:
        caCert:
          RSAbits: 2048
          expiration: 10y
```

With mTLS enabled we can start to read the xDS configuration of any dataplanes we have deployed in our mesh.

## Navigating the xDS of a Dataplane

**So What did I find?** Within the dynamic listener associated to the DP of a service, there is an additional TLS filter `DownstreamTlsContext` used by Envoy to filter any connections by valiating the SAN of the certification presented.

**1** -  Line 9 - defines what matching SAN it will accept, and in this case in line 11 we can see it's configured to expecte a `spiffe id`.

**2** - Line 15 - defines the secret where the trusted CA secret is held.

**3** - Line 23 - defines the secret where the `svid` can be found.

**4** - Line 33 - we can also see from the configuration that a client cert is `required`.

```yaml
[Filter]> .configs[2].dynamicListeners[0].activeState.listener.filterChains[0].transportSocket
{                                                                                                                                
  "name": "envoy.transport_sockets.tls",                                                                                         
  "typedConfig": {                                                                                                               
    "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext",                               
    "commonTlsContext": {                                                                                                        
      "combinedValidationContext": {                                                                                             
        "defaultValidationContext": {                                                                                            
          "matchSubjectAltNames": [                                                                                              
            {                                                                                                                    
              "prefix": "spiffe://default/"                                                                                      
            }                                                                                                                    
          ]                                                                                                                      
        },                                                                                                                       
        "validationContextSdsSecretConfig": {                                                                                    
          "name": "mesh_ca:secret:default",                                                                                      
          "sdsConfig": {                                                                                                         
            "ads": {},                                                                                                           
            "resourceApiVersion": "V3"                                                                                           
          }                                                                                                                      
        }                                                                                                                        
      },                                                                                                                         
      "tlsCertificateSdsSecretConfigs": [                                                                                        
        {                                                                                                                        
          "name": "identity_cert:secret:default",                                                                                
          "sdsConfig": {                                                                                                         
            "ads": {},                                                                                                           
            "resourceApiVersion": "V3"                                                                                           
          }                                                                                                                      
        }                                                                                                                        
      ]                                                                                                                          
    },                                                                                                                           
    "requireClientCertificate": true                                                                                             
  }                                                                                                                              
}
```


**So the `svid` and `trusted CA` are held in secrets. We can locate those as well.**

Under the dynamicActiveSecrets were the `identity_cert:secret:default` holding the tls cert and the `mesh_ca:secret:default` holding the trusted CA.

Locating the `identity_cert:secret:default` I used the jid filter:

```bash
.configs[5].dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes
```

The secret is base64 encoded so base64 decoded it:

`echo LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0... | base64 -d > identity_cert.pem`

and then using openssl to read the pem file: `openssl x509 -in identity_cert.pem -text -noout`

```bash
...
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                bf:93:6b:e1:fb:03:81:c8:3d:32:95:8a:ea:61:58:79
        Signature Algorithm: sha256WithRSAEncryption
            Issuer: O=Kuma, OU=Mesh, CN=default
            Validity
                Not Before: Oct 18 12:48:15 2022 GMT
                Not After : Oct 19 12:48:25 2022 GMT
        ...
            X509v3 extensions:
            ...
                X509v3 Subject Alternative Name: critical
                    URI:spiffe://default/monolith-service_svc_5000, URI:kuma://kuma.io/protocol/http, URI:kuma://kuma.io/service/monolith-service_svc_5000, URI:kuma://kuma.io/zone/on_prem
            ...
```

And we can see the spiffe id in the the Subject Alternative Name (SAN).

Then the second secret is the trusted CA, which is self explanatory.

```yaml
[Filter]> .configs[5].dynamicActiveSecrets[1]    
{                                         
  "lastUpdated": "2022-10-18T12:48:25.899Z",                      
  "name": "mesh_ca:secret:default",                                   
  "secret": {                                                           
    "@type": "type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret",
    "name": "mesh_ca:secret:default",                                                 
    "validationContext": {                   
      "trustedCa": {                                                                    
        "inlineBytes": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURPakNDQWlLZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREF3TVEwd0N3WURWUVFLRXdSTGRXMWgKTVEwd0N3WURWUVFMRXdSTlpYTm9NUkF3RGdZRFZRUURFd2RrWldaaGRXeDBNQjRYRFRJeU1UQ
      }                                                                                                                                                                                                                      
    }                          
  },                                                                                                                                                                                                                         
  "versionInfo": "0dc8f126-827c-449e-8af2-f36c8b704d24"                                                                                                                                                                      
}
```

## Cool! All done

That was a fun trace of what all is going on!

## References

Just a list of all the references mentioned during the post.

* [Mutual TLS Doc of Kuma Mesh](https://kuma.io/docs/1.8.x/policies/mutual-tls/)

* [spiffe.io homepage](https://spiffe.io/)

* [Tutorial - Using Envoy with x509-SVIDs](https://spiffe.io/docs/latest/microservices/envoy-x509/readme/)