---
title: "Detecting Outdated Proxy Configuration"
category: std

docname: draft-rosomakho-httpbis-outdated-proxy-config-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: HTTP
keyword:
 - PAC
 - PvD
 - outdated
venue:
  group: HTTP
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "yaroslavros/httpbis-outdated-proxy-config"
  latest: "https://yaroslavros.github.io/httpbis-outdated-proxy-config/draft-rosomakho-httpbis-outdated-proxy-config.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

normative:


informative:
  PAC-FILE:
    author:
      org: Mozilla
    title: "Proxy Auto-Configuration (PAC) file"
    target: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Proxy_servers_and_tunneling/Proxy_Auto-Configuration_PAC_file
    date: May 2025

--- abstract

This document defines a lightweight mechanism that allows explicit HTTP proxies to inform clients when their proxy configuration, such as a Proxy Auto-Configuration (PAC) file or Provisioning Domain (PvD) proxy configuration, is outdated. Clients signal to the proxy the configuration URL and the timestamp of when it was last fetched. In response, the proxy may indicate that a newer version of the configuration is available. This enables clients to refresh their configuration without relying on frequent polling or short expiration intervals. The mechanism is designed to be compatible with existing proxy deployment models and supports both PAC-based and PvD-based configurations.

--- middle

# Introduction

Explicit HTTP proxies are widely deployed in enterprise and managed network environments to enforce routing policies, enable traffic inspection, or implement access control. Clients relying on such proxies typically obtain configuration through a Proxy Auto-Configuration (PAC) file {{PAC-FILE}} or a Provisioning Domain (PvD) {{?PROXY-PVD=I-D.ietf-intarea-proxy-config}}. In many deployments, it is necessary to update these configurations in response to operational changes such as service onboarding, emergency routing adjustments, or policy corrections.

Currently, clients have limited mechanisms to detect whether the proxy configuration they are using is stale. As a result, PAC files are often polled at short intervals, and PvD expiration times are configured with low values to accelerate refreshes. These approaches are inefficient and may introduce unnecessary network traffic or delay the application of important updates.

This document defines a simple mechanism that enables a proxy to inform a client that its current configuration is outdated. The client includes in its request a structured field indicating the URL of the PAC file or PvD and the timestamp of when it last fetched the configuration. If the proxy recognizes the configuration and determines that a newer version is available, it may respond with a boolean indicator prompting the client to refresh the configuration.

This mechanism applies to all forms of explicit proxying over HTTP, including:

- HTTP CONNECT as defined {{Section 9.3.6 of !HTTP=RFC9110}}
- {{?CONNECT-UDP=RFC9298}}
- {{?CONNECT-IP=RFC9484}}
- Templated {{?CONNECT-TCP=I-D.ietf-httpbis-connect-tcp}}

The mechanism is optional, compatible with existing protocols, and requires no persistent state. It allows clients to discover configuration updates proactively while preserving the existing operational model.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Protocol Overview

To enable detection of stale proxy configuration, clients may include a `Proxy-Config` HTTP header field in requests sent to an explicit proxy. This header communicates the URL of the proxy configuration (such as a PAC file or PvD) and the timestamp of when the client last fetched it.

Upon receiving a request containing the `Proxy-Config` header, a proxy that supports this mechanism compares the provided timestamp with the most recent update time of the corresponding configuration. If the proxy determines that the client configuration is outdated, it can signal this condition using the `Proxy-Config-Stale` response header.

This mechanism is optional and advisory. Proxies are not required to track or respond to client configuration metadata, and clients are not required to act immediately upon receiving a staleness indication. The mechanism is designed to supplement, not replace, existing configuration refresh behaviors.

# Proxy-Config Header

The `Proxy-Config` request header field is used by a client to inform an explicit proxy about the proxy configuration it is currently using. The field conveys a dictionary structured field as defined in {{Section 3.2 of !STRUCTURED-FIELD=RFC9651}} with the following keys:

- `url` (optional): A string identifying the URL from which the client fetched the configuration. This may point to a PAC file or a PvD configuration. If omitted, the proxy is expected to infer the default PvD URI based on its own hostname and ".well-known/pvd" path as defined in {{Section 4.1 of ?PVDDATA=RFC8801}}.
- `fetched` (required): A date value indicating when the client last fetched the configuration. The value MUST use the Date format defined in {{Section 3.3.7 of STRUCTURED-FIELD}}.

## Handling Unknown or Sensitive URLs

Clients MUST NOT include URLs that expose local system information (e.g., `file://` URLs). Clients SHOULD limit use of the Proxy-Config header to contexts where it does not introduce privacy or security risks, such as trusted or encrypted proxy connections.

## Examples

A client using a PAC file retrieved from `https://config.example.net/proxy.pac` at 2025-06-01T00:00:00Z MAY include the following header:

~~~~~~~~~~ ascii-art
Proxy-Config: url="https://config.example.net/proxy.pac", fetched=@1748736000
~~~~~~~~~~
{: title="Example of Proxy-Config header with url field"}

If the client is using the default PvD URI associated with the proxy hostname, it may omit the `url` key:

~~~~~~~~~~ ascii-art
Proxy-Config: fetched=@1748736000
~~~~~~~~~~
{: title="Example of Proxy-Config header without url field"}

# `Proxy-Config-Stale` Header

The `Proxy-Config-Stale` response header field is used by a proxy to inform the client whether its current proxy configuration is considered outdated. This allows the client to make informed decisions about whether to refresh the configuration.

The field is a boolean as defined in {{Section 3.3.6 of STRUCTURED-FIELD}}, where:

- `?1` indicates that the configuration used by the client is stale and a newer version is available.
- `?0` indicates that the configuration used by the client is current.

The proxy MUST only include this header if it has received a valid `Proxy-Config` header from the client and is able to recognize and evaluate the configuration URL. If the proxy does not recognize the provided configuration URL, does not track updates for it, or does not support this mechanism, it MUST omit the `Proxy-Config-Stale` header.

The `Proxy-Config-Stale` header MAY appear in both successful and error responses, except for responses related to client authentication (e.g., `407 Proxy Authentication Required`). Including the header in such authentication-related responses could result in unintended exposure of configuration metadata and may interfere with authentication workflows.

The `Proxy-Config-Stale` field is advisory. Its presence does not affect the processing of the current request or response. Clients MAY choose how and when to act upon the information, including deferring configuration refresh until convenient.

# Security Considerations

Clients using the `Proxy-Config` header field must take care to avoid disclosing sensitive information in the URL or metadata they send to the proxy.

## Avoiding Sensitive URLs

Clients MUST NOT include configuration URLs that reveal local system details, such as `file:// URIs` or paths containing user-specific or internal directory structures. To reduce exposure, clients SHOULD only use this mechanism when proxy configuration is relevant to the proxy (e.g., hosted on the proxy or a part of the same enterprise domain).

## Trusted Communication Channels

The `Proxy-Config` header is intended for use over secure and trusted communication channels. Clients SHOULD send this header only when using TLS or when otherwise confident that the request is not observable or modifiable by unauthorized intermediaries.

## Authentication-related Responses

Proxies MUST NOT include the `Proxy-Config-Stale` header in responses related to client authentication (e.g., `407 Proxy Authentication Required`). This avoids potential leakage of client configuration metadata during authentication flows that may be visible to other components or exposed through logging or error handling.

## Denial-of-Service Considerations

A misconfigured or malicious proxy could include `Proxy-Config-Stale: ?1` in every response, causing the client to repeatedly refetch proxy configuration. This behavior can lead to excessive network traffic, CPU usage, or degraded performance on the client, particularly in environments where configuration retrieval is resource-intensive.

To mitigate this risk, clients MUST implement appropriate rate limiting or throttling mechanisms when acting on stale configuration indications. For example, a client may choose to ignore repeated `?1` responses within a minimum refresh interval or apply exponential backoff when encountering multiple stale signals in quick succession.

Clients SHOULD validate the authenticity and integrity of any fetched configuration before applying it, and ensure that configuration refreshes do not interfere with ongoing connection or session state.

# IANA Considerations

This document registers the following HTTP header fields in the "Hypertext Transfer Protocol (HTTP) Field Name Registry":

Proxy-Config

- Field Name: Proxy-Config
- Status: permanent
- Reference: this document

Proxy-Config-Stale

- Field Name: Proxy-Config-Stale
- Status: permanent
- Reference: this document

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Tommy Pauly and Dragana Damjanovic for reviewing the concept and providing initial feedback.
