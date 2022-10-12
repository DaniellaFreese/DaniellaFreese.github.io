---
title: "Kong Mesh - Understanding how the Kong Mesh translates TrafficRoute Policies to Envoy Proxy Configuration"
date: 2022-10-06
draft: false

categories: ["Service Mesh"]
tags: ["service mesh", "Kong Mesh", "Kuma", "Envoy Proxy"]
toc: false
summary: "**Situation:** Learning more about Kong Mesh and Envoy Proxy Part: I wanted to understand specifically how a Traffic Route translates to Envoy Proxy Configuration and which Sidecar Proxies."
author: "djfreese"
---

**Situation:** Learning more about Kong Mesh and Envoy Proxy Part: I wanted to understand specifically how a Traffic Route translates to Envoy Proxy Configuration and which Sidecar Proxies.

In the previous blog I quickly ran through the configuration of a delegated gateway, and services to see all the moving parts of the Envoy Proxy Configuration. In this second blog I want to very specifically notes how the `Traffic Route` is translated to Envoy and passed down.

## Using jid tooling

The proxy configuration is pretty long, so to navigate it I'll be using the `jid` tool [jid - github](https://github.com/simeji/jid#simply-use-jid-command).

`jid` has proved to be super convenient to filter through giant json data.s

## Mesh Services Setup

So I have a Kong Mesh running with three services - 1 delegated Kong Gateway, and 2 standard services.

The `Traffic Route` Policy created on the mesh is below, and it in summary states the following:

* if a request reaches the service `kong` and the intended destination is the `monolith` service, check if there is a url match to `/monolith/resources/card/dispute` if so, direct that request to the `microservice`, all other requests will be directed to the monolith.


```yaml
type: TrafficRoute
mesh: default
name: kong-reroute
sources: 
  - match: 
      kuma.io/service: kong
destinations: 
  - match: 
      kuma.io/service: monolith-service_svc_5000
conf: 
  destination: 
    kuma.io/service: monolith-service_svc_5000
  http: 
    - match: 
        path: 
          prefix: /monolith/resources/card/dispute
      destination: 
        kuma.io/service: microservice_microservice_svc_8080
```

### Grabbing Proxy Configuration

In some cases I am running kong mesh on a host, so to grab the proxy configs I am going to ssh into the box, curl the proxy admin port and keep the output in a file to dynamically filter with the jid tool:

 ```sh
curl -X GET localhost:9901/config_dump > configDump.json

cat configDump.json | jid
```

### Gateway Dynamic Listners

Drilling down into the proxy configuration of the Gateway, it is important to note that will be a listener for each backend service, in other words a dynamic listener was generated for the `monolith` and there is a separate for the `microservice`. In order to understand how the above Traffic Route works, we need to read both listeners and more specificially the filterchain of each one.

#### Monolith Listener

The entire filterchain is posted to avoid any ambiguity. In the filterchain we can see the HTTP Connection Manager Network Filter being used, and with it the `route_config` has the bulk of the information we are looking.

The `route_config`, and filterchains for that matter, are read in order [Envoy Proxy - Route Matching](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/route_matching).

So the literal translation of the route_config is near identical to the Traffic Route policy and states: `analyze the traffic intended for monolith service, if it has the prefix match /monolith/resources/card/dispute direct that request to the microservice, otherwise any request matching to the url / will be sent to the monolith.`

```yaml
[Filter]> .configs[2].dynamic_listeners[1].active_state.listener.filter_chains[0].filters[0]
{
  "name": "envoy.filters.network.http_connection_manager",
  "typed_config": {
    "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
    "common_http_protocol_options": {
      "idle_timeout": "3600s"
    },
    "http_filters": [
      {
        "name": "envoy.filters.http.router",
        "typed_config": {
          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
        }
      }
    ],
    "route_config": {
      "name": "outbound:monolith-service_svc_5000",
      "request_headers_to_add": [
        {
          "header": {
            "key": "x-kuma-tags",
            "value": "\u0026kuma.io/service=kong\u0026\u0026kuma.io/zone=on_prem\u0026"
          }
        }
      ],
      "validate_clusters": false,
      "virtual_hosts": [
        {
          "domains": [
             "*"
          ],
          "name": "monolith-service_svc_5000",
          "retry_policy": {
            "num_retries": 5,
            "per_try_timeout": "16s",
            "retry_back_off": {
              "base_interval": "0.025s",
              "max_interval": "0.250s"
            },
            "retry_on": "gateway-error,connect-failure,refused-stream"
          },
          "routes": [
            {
              "match": {
                "prefix": "/monolith/resources/card/dispute"
              },
              "route": {
                "cluster": "microservice_microservice_svc_8080",
                "timeout": "15s"
              }
            },
            {
              "match": {
                "prefix": "/"
              },
              "route": {
                "cluster": "monolith-service_svc_5000",
                "timeout": "15s"
              }
            }
          ]
        }
      ]
    },
    "stat_prefix": "monolith-service_svc_5000",
    "stream_idle_timeout": "1800s"
  }
```

### Microservice Filterchain

Following up with the listener and filter chain for the microservice. The entire listener is provided below because it was not as dense.

The most valuable takeway from the sample below is the filterchain object is empty. Hence, all the logic for the sample Traffic Route is being driven gateway envoy proxy in the listener assigned to the monolith, which makes sense, because the policy defines:

* **First** the `source` - i.e. the `gateway`

* **Second** the `destination` - i.e. the `listener` defined on the source

```yaml
[Filter]> .configs[2].dynamic_listeners[2]                                                  
{
  "active_state": {                                                
    "last_updated": "2022-10-10T15:30:37.010Z",             
    "listener": {                                                                                                    
      "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",                                                
      "address": {                                                                                                       
        "socket_address": {                                                                                                
          "address": "127.0.0.1",          
          "port_value": 33034      
        }                                   
      },                                      
      "filter_chains": [                                                               
        {}                                                                               
      ],                                                                                   
      "metadata": {                                                                                                          
        "filter_metadata": {                                          
          "io.kuma.tags": {                                                                                                    
            "kuma.io/service": "microservice_microservice_svc_8080"                                                              
          }                                            
        }                                                                                                                          
      },                                            
      "name": "outbound:127.0.0.1:33034",                                              
      "traffic_direction": "OUTBOUND"                                                          
    },                                                                                     
    "version_info": "f9eb4f12-c266-4cda-9932-16a67ecaeb0a"                                           
  },                                                                                               
  "name": "outbound:127.0.0.1:33034"
}
```

## That's all Folks

And that's all folks. I like doing these little snippets just to do some quick concept validation and start the process of demystifying enovy proxy. The key takeway here is a lot of the cool traffic management features of Kong Mesh are present in the filter chains of `source` defined in the `Traffic Route`.
