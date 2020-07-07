# otel-k8s
Quick start manifests for demonstrating OpenTelemetry on Kubernetes

## Kubernetes Deployment of OpenTelemetry Stack
This repo contains kubernetes manifests necessary to deploy the OpenTelemetry stack.
Mostly inspired from https://github.com/open-telemetry/opentelemetry-collector/blob/master/examples/demo/

- Load Generator
  - deployment
    - OpenCensus instrumented go application
      - source: https://github.com/open-telemetry/opentelemetry-collector/blob/master/examples/main.go
    - [omnition/synthetic-load-generator](https://github.com/Omnition/synthetic-load-generator)
      - jaeger-emitter
      - zipkin-emitter
    - [freshtracks.io/avalanche](https://github.com/open-fresh/avalanche)
      - prometheus metrics generator
  - service
    - `:9001/metrics` endpoint exposed through a service
- opentelemetry-agent
  - daemonset
    - receivers
      - opencensus
      - jaeger
      - zipkin
      - prometheus
    - processors
      - batch
      - queued-retry
    - exporters
      - opencensus
  - headless service
    - `55678` for opencensus protocol
    - `14268` for jaeger thrift api
    - `9411` for zipkin proto
- opentelemetry-collector
  - deployment
    - receivers
      - opencensus
    - processors
      - batch
      - queued-retry
    - exporters
      - jaeger
      - zipkin
      - prometheus
- data sinks
  - jaeger-all-in-one, exposed at `:14250`
  - zipkin-all-in-one exposed at `:9411`
  - prometheus
    - scrapes opentelemetry service `:8889` for load-generator metrics
    - scrapes opentelemetry service `:8888` for collector metrics

## Architecture Diagram

          +-------------------------------------------------------------------------------------------------------------+                                                                      
          | load-generator deployment                                                                                   |                                                                      
          |+------------------+  +--------------------------+  +--------------------------+ +--------------------------+|                                                                      
          || OpenCensus       |  | omnition                 |  |omnition                  | | freshtracks.io           ||                                                                      
          || trace + metrics  |  | synthetic-load-generator |  |synthetic-load-generator  | | avalanche                ||                                                                      
          || generator        |  | jaeger-emtter            |  |zipkin-emitter            | | prom metrics generator   ||                                                                      
          |+----------\-------+  +-------------|------------+  +-----------/--------------+ +-------------|------------+|                                                                      
          +---------------\--------------------|-----------------------/----------------------------------|-------------+                                                                      
                           --\                 |                   /---                     +-------------|------------+                                                                       
                              ---\             |               /---                         |    prometheus-avalanche  |                                                                       
                          :55678  --\          |:14268     /---  :9411                      |    service               |                                                                       
                                     ---\      |       /---                                 +-------------|------------+                                                                       
                                    +----------|----------+                                               |                                                                                    
                                    | opentelemetry-agent |                                               |                                                                                    
                                    | headless service    |                                               |                                                                                    
                                    +---/------|----------+                                               |                                                                                    
                                  /-----       |       \---                                               |                                                                                    
          +-----------------/------------------|-----------\----------------------------------------------|--------------+                                                                     
          |+----------/------------------------|---------------\------------------------------------------|-------------+|                                                                     
          || +-----------+                +----|---+            +--\-----+     +-------------+  +---------|---------+   ||                                                                     
          || |opencensus |                | jaeger |            |zipkin  |     | prometheus  ---- scrape-config     |   ||                                                                     
          || +-----------+                +--------+            +--------+     +-------------+  +-------------------+   ||                                                                     
          || Receivers                                                                                                  ||                                                                     
          |+------------------------------------------------------------------------------------------------------------+|                                                                     
          |+------------------------------------------------------------------------------------------------------------+|                                                                     
          ||                              +-------+                            +--------------+                         ||                                                                     
          ||                              | batch |                            | queued-retry |                         ||                                                                     
          || Processors                   +-------+                            +--------------+                         ||                                                                     
          |+------------------------------------------------------------------------------------------------------------+|                                                                     
          |+------------------------------------------------------------------------------------------------------------+|                                                                     
          ||                                             +------------------+                                           ||                                                                     
          ||                                             |    opencensus    |                                           ||                                                                     
          || Exporters                                   +---------|--------+                                           ||                                                                     
          |+-------------------------------------------------------|----------------------------------------------------+|                                                                     
          |  open-telemetry agent daemonset                        |                                                     |                                                                     
          +--------------------------------------------------------|-----------------------------------------------------+                                                                     
                                                                   | :55678                                                                                                                    
                                                      +------------|------------+                                                                                                              
                                                      |opentelemetry-collector  |                                                                                                              
                                                      |headless service         |                                                                                                              
                                                      +------------|------------+                                                                                                              
                                                                   |                                                                                                                           
          +--------------------------------------------------------|-----------------------------------------------------+                                                                     
          |+-------------------------------------------------------|---------------------------------------------------+ |                                                                     
          ||                                                +------|-----+                                             | |                                                                     
          ||                                                | opencensus |                                             | |                                                                     
          ||  Receivers                                     +------------+                                             | |                                                                     
          |+-----------------------------------------------------------------------------------------------------------+ |                                                                     
          |+-----------------------------------------------------------------------------------------------------------+ |                                                                     
          ||                              +-------+                             +--------------+                       | |                                                                     
          ||                              | batch |                             | queued-retry |                       | |                                                                     
          ||  Processors                  +-------+                             +--------------+                       | |                                                                     
          |+-----------------------------------------------------------------------------------------------------------+ |                                                                     
          |+-----------------------------------------------------------------------------------------------------------+ |                                                                     
          ||                  +------------+            +------------+            +--------------------+               | |                                                                     
          ||                  |  jaeger    |            |  zipkin    |            | prometheus         |               | |                                                                     
          ||  Exporters       +------|-----+            +------|-----+            +----------|---------+               | |                                                                     
          |+-------------------------|-------------------------|-----------------------------|-------------------------+ |                                                                     
          +--------------------------|-------------------------|-----------------------------|---------------------------+                                                                     
                                     |                         |                +------------|-----------+                                                                                     
                                     |                         |                |opentelemetry-collector |                                                                                     
                                     |                         |                |headless service        |                                                                                     
                                     |                         |                +------------|-----------+                                                                                     
                                     |:14250                   |:9411                        | :8889                                                                                           
                         +-----------|----------+  +-----------|-----------+    +------------|-----------+                                                                                     
                         |   jaeger-all-in-one  |  |  zipkin-all-in-one    |    |     prometheus         |                                                                                     
                         +----------------------+  +-----------------------+    +------------------------+                                                                                                                                 

## Deployment
```
kubectl apply -f load-generator/*
kubectl apply -f otel-agent/*
kubectl apply -f otel-collector/*
kubectl apply -f jaeger-all-in-one/*
kubectl apply -f zipkin-all-in-one/*
kubectl apply -f prometheus/*
```
