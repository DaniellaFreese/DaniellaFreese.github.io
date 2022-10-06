---
title: "Kong Mesh - Diving into the Envoy Proxy Configuration"
date: 2022-10-05
draft: false

categories: ["Service Mesh"]
tags: ["service mesh", "Kong Mesh", "Kuma", "Envoy Proxy"]
toc: false
author: "djfreese"
---

Learning more about Kong Mesh and Envoy Proxy, I wanted to be able to understand how Envoy Proxy connects all the networks dots through its configuration file.

So I recently spun Kong Mesh with several dataplanes and I wanted to see if via the xds configuration I could trace it all.

TODO PICTURE OF THE

Basically what I wanted to manually see was all the steps Envoy takes to make the below connection happen:

```
Kong Gateway --> Envoy Proxy of Gateway --> Envoy Proxy of Monolith --> Monolith
```

What I ended up finding was the following:

```
I. Kong Gateway --> Envoy Proxy --> Listener on Port to Outbound Service (Monolith) --> Route Config to Monolith --> Cluster Monolith --> Monolith Endpoint -->

II. Envoy Proxy of Monolith --> Listener in on Port --> Route Config --> Cluster --> Endpoint - localhost 8080 where Monolith`
```

## Installing jid tooling

First I installed the `jid` tool on the host machines I was working on [jid - github](https://github.com/simeji/jid#simply-use-jid-command).

It did prove to be super convenient to filter through the giant json data.

## Envoy Proxy - Gateway

Starting the the Envoy Proxy supporting the Gateway.

**First** - I wanted to get the config dump of the xds configuration. I did that by running the curl command below.

```bash
 curl -X GET localhost:9901/config_dump?include_eds > gateway_configDump.json
```

I had to use the extra query parameter `include_eds` to also retrieve the endpoint information. I didn't know this as first but was directed to [Envoy Proxy API docs](https://www.envoyproxy.io/docs/envoy/latest/operations/admin#get--config_dump?include_eds).

### Gateway - Dynamic Listener

Basically in the output below I could summarize the following reading it top down:

1. proxy is listening on 33033
2. it is using the HTTP Connection Manager in the filter chain to create a route config
3. the route config will foward on any traffic matching to the route `/` (the base url) to the cluster `monolith-service_svc_5000`

```json
[Filter]> .configs[2].dynamic_listeners[1]

{
  "active_state": {
    "last_updated": "2022-10-04T21:24:13.388Z",
    "listener": {
      "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
      "address": {
        "socket_address": {
          "address": "127.0.0.1",
          "port_value": 33033
        }
      },
      "filter_chains": [
        {
          "filters": [
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
                        "value": "&kuma.io/service=kong&&kuma.io/zone=on_prem&"
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
            }
          ]
        }
      ],
      "metadata": {
        "filter_metadata": {
          "io.kuma.tags": {
            "kuma.io/service": "monolith-service_svc_5000"
          }
        }
      },
      "name": "outbound:127.0.0.1:33033",
      "traffic_direction": "OUTBOUND"
    },
    "version_info": "cc2a7552-f767-41b3-9905-5fa7ef71361f"
  },
  "name": "outbound:127.0.0.1:33033"
}
```


The next question I had was, ok this doesn't mention any info about the cluster `monolith-service_svc_5000` so lets go find that.

### Gateway - Dynamic Cluster

Reading the matching dynamic cluster configuration for `monolith-service_svc_5000` shown below I could interpret that basically it's a cluster of type `EDS`.

What the fudge is type EDS? Go back to the [Envoy Docs](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery) I found

**The endpoint discovery service is a xDS management server based on gRPC or REST-JSON API server used by Envoy to fetch cluster members. The cluster members are called “endpoint” in Envoy terminology. For each cluster, Envoy fetch the endpoints from the discovery service.**

```json
[Filter]> .configs[1].dynamic_active_clusters[1]
{    
  "cluster": {                                                     
    "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",   
    "circuit_breakers": {                                               
      "thresholds": [                     
        {                                                              
          "max_connections": 1024,          
          "max_pending_requests": 1024,    
          "max_requests": 1024,    
          "max_retries": 3                   
        }                       
      ]        
    },                         
    "connect_timeout": "5s",  
    "eds_cluster_config": {            
      "eds_config": {                        
        "ads": {},                       
        "resource_api_version": "V3"           
      }                                   
    },                                       
    "name": "monolith-service_svc_5000",
    "outlier_detection": {         
      "enforcing_consecutive_5xx": 0,
      "enforcing_consecutive_gateway_failure": 0,
      "enforcing_consecutive_local_origin_failure": 0,                                                                                                                                                                                                   
      "enforcing_failure_percentage": 0,                                                                                                                                                                                                                 
      "enforcing_success_rate": 0
    },                             
    "type": "EDS",                             
    "typed_extension_protocol_options": {                 
      "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
        "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
        "explicit_http_config": {
          "http2_protocol_options": {}                               
        }                             
      }                                                                
    }                        
  },                                
  "last_updated": "2022-10-04T21:23:37.389Z",
  "version_info": "f4340f9c-59b7-4320-9d87-920109a7f511"
}
```

### Gateway - Dynamic Endpoint Configs

Which Means now I have to go check out the EDS configuration, and which is why the `?include_eds` was appended to the original /config_dump request.

Reading the endpoint_config I could finally see:

1. the actual address of the Envoy Proxy supporting my "Monolith" application. Envoy for the monolith is running on `10.0.0.5` and listening in on port `5000`.

```json
[Filter]> .configs[2].dynamic_endpoint_configs[0]endpoint_config
{
  "endpoint_config": {
    "@type": "type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment",
    "endpoints": [
      {
        "lb_endpoints": [
          {
            "endpoint": {
              "address": {
                "socket_address": {
                  "address": "10.0.0.55",
                  "port_value": 5000
                }
              },
              "health_check_config": {}
            },
            "health_status": "HEALTHY",
            "load_balancing_weight": 1,
            "metadata": {
              "filter_metadata": {
                "envoy.lb": {
                  "kuma.io/protocol": "http",
                  "kuma.io/zone": "on_prem"
                },
                "envoy.transport_socket_match": {
                  "kuma.io/protocol": "http",
                  "kuma.io/zone": "on_prem"
                }
              }
            }
          }
        ],
        "locality": {
          "zone": "on_prem"
        }
      }
    ],
    "policy": {
      "overprovisioning_factor": 140
    }
  }
}
```

Ok so now I know:

`Gateway --> Envoy Proxy --> Listener --> Filter Chain with Route --> to Monolith Cluster --> EDS endpoint pointing to Envoy on Monolith Server`

The next question I had was, how is the Envoy Proxy of the Monolith Configured?

## Monolith - Dynamic Listener

Take the same pattern as I did for the Gateway, I started with the listeners first. This listener has lots of info so we'll break it up.

**First** - I identified the listener, and I can see envoy is listening on `10.0.0.55:5000`:

```json
[Filter]> .configs[2].dynamic_listeners[0].active_state.listener.address
{    
  "socket_address": {                                               
    "address": "10.0.0.55",                                           
    "port_value": 5000                                                  
  }                                                                    
}        
```

**Second** - I looked at the filter chains. There were multiple filters, but I found the HTTP Connection Manager to route traffic into to monolith application. 

```json
[Filter]> .configs[2].dynamic_listeners[0].active_state.listener.filter_chains[0].filters[1]
{    
  "name": "envoy.filters.network.http_connection_manager",          
  "typed_config": {                                                   
    "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
    "common_http_protocol_options": {                                              
      "idle_timeout": "7200s"                                                        
    },                                                                                 
    "forward_client_cert_details": "SANITIZE_SET",                         
    "http_filters": [              
      {                                      
        "name": "envoy.filters.http.router",
        "typed_config": {        
          "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
        }                                      
      }                              
    ],                                                                                   
    "route_config": {                              
      "name": "inbound:monolith-service_svc_5000",                                         
      "request_headers_to_remove": [                                                         
         "x-kuma-tags"                                                                         
      ],                                       
      "validate_clusters": false,                                                                
      "virtual_hosts": [                        
        {                                         
          "domains": [                                                                                                                                                                                                                                   
             "*"                                                                                                                                                                                                                                         
          ],                                                  
          "name": "monolith-service_svc_5000",                                                                         
          "routes": [                                                                                                    
            {                                                                                                              
              "match": {                           
                "prefix": "/"                               
              },                                      
              "route": {                                
                "cluster": "localhost:8080",          
                "timeout": "0s"                                        
              }                                                                          
            }                                                                              
          ]                                                                                                                  
        }                                                             
      ]                                                                                                                        
    },                                                                                                                           
    "set_current_client_cert_details": {                                                                                           
      "uri": true                                       
    },                                                                                                                               
    "stat_prefix": "localhost_8080",                          
    "stream_idle_timeout": "3600s"                              
  }                                                                                            
}      
```

**Third** - and final I located the cluster `localhost:8080` and found that in there is contains the endpoint to reach the actual application it is backing.

```json
[Filter]> .configs[1].dynamic_active_clusters[1]                                            
{    
  "cluster": {                                                        
    "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",   
    "alt_stat_name": "localhost_8080",                                                                               
    "connect_timeout": "10s",                                                      
    "load_assignment": {                                                             
      "cluster_name": "localhost:8080",                                                
      "endpoints": [                                                       
        {                          
          "lb_endpoints": [                  
            {                               
              "endpoint": {      
                "address": {                                                           
                  "socket_address": {          
                    "address": "127.0.0.1",
                    "port_value": 8080                                                   
                  }                                
                }                                                                          
              }                                                                              
            }                                                                                  
          ]                                    
        }                                                                                        
      ]                                         
    },                                            
    "name": "localhost:8080",                                                                                                                                                                                                                            
    "type": "STATIC",                                                                                                                                                                                                                                    
    "typed_extension_protocol_options": {                     
      "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {                                                      
        "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",                           
        "common_http_protocol_options": {                                                                                  
          "idle_timeout": "7200s"                  
        },                                                  
        "explicit_http_config": {                     
          "http_protocol_options": {}                                
        }                                             
      }                                                                
    }                                                                                    
  },                                                                                       
  "last_updated": "2022-10-04T21:24:12.976Z",                                                                                
  "version_info": "27a728ec-3bce-45a0-a4e1-a2d38d3bf4bf"              
}                                                                                                                              
```

## Cool! All done

Just wanted to highlight that it's not mysterious.
