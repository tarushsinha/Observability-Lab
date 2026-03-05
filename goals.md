## Goal
The goal of the lab is to understand the full observability pipeline:

Application -> Metrics -> Prometheus -> Grafana -> Alerts

Conceptually:
Application -> OpenTelemetry -> Collector -> Observability Backend

## Real Production Architecture

Applications -> OpenTelemetry SDK -> OpenTelemetry Collector -> { Prometheus, Datadog, Jaeger, ElasticSearch }

Prometheus may still scrape k8s components directly

Typical Prod Architecture:

        K8S Components
            |
            | pull
            V
        Prometheus
            ^
            | scrape
            |
        OTel Collector
            |
            V
        Applications
    
 Prometheus Scrapes:
 - Kubernetes system metrics
 - OTel Collector Metrics
 - Infra exporters

 OTel Collects:
 - Application metrics
 - Traces
 - Logs

## OpenTelemetry

OpenTelemetry typically uses a push model where applications export telemetry to a collector which processes and forwards data to observability backends like Datadog, Prometheus, or Jaeger

Application 
    |
    | - metrics
    | - traces
    | - logs

Exported via SDK - sent to OpenTelemetry Collector

Push is needed for a few scenarios:
1) Short Lived Workloads
 For applications for certain short-lived, ephemeral workloads:
    Examples: kubernetes job, serverless function, batch pipeline
 Some of these workloads run for seconds and a 15s prometheus scraping interval might miss them entirely. In this scenario, push guarentees delivery.

2) Network Boundaries
 For applications that run outside cluster, in different VPCs, behind firewalls, push works across boundaries. Prometheus scraping requires direct network access to the service

3) Traces cannot be scraped
 Traces are event streams, not counters. 
    Examples: HTTP request span, DB query span, Cache miss span
 These must be pushed, Prometheus cannot scrape them

4) OTel Collector exists to Normalize Telemetry
 The OTel Collector acts like a telemetry router. The benefits include: batching, sampling, enrichment, protocol translation
 
 Application
    |
    V
 OTel Collector
    |
    | - Prometheus
    | - DataDog
    | - Jaeger
    | - Kafka

5) Prometheus is never push based
 The OTel Collector doesn't push to Prometheus:
    - The collector exposes /metrics
    - Prometheus then scrapes the collector

## Prometheus

Prometheus uses a pull model where the server scrapes metrics endpoints exposed by applications. This works well in dynamic environments like Kubernetes where service discovery allows Prometheus to find targets automatically (service discovery, controlled scrape rate))

Prometheus -> scrape -> /metrics

Prometheus 
    |
    | - scrape node exporter
    | - scrape kubelet
    | - scrape API Server

If scraping fails, this becomes a signal itself, as push systems cannot easily detect silent failures

## Summary

Prometheus works well for infrastructure monitoring because it can automatically discover and scrape Kubernetes components. However, application telemetry often requires push-based delivery because workloads may be short-lived, cross network boundaries, or produce traces that cannot be scraped. OpenTelemetry provides a standardized instrumentation layer that pushes telemetry to a collector, which can then expose metrics for Prometheus to scrap or forward telemetry to other observability systems. This allows organizations to combine Prometheus's strong infrastructure monitoring capabilities with OpenTelemetry's flexible application instrumentation.