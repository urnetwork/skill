---
name: urnetwork-proxy
description: Use this skill to connect generated code and agents to network proxies for anonymous, local internet with specific cities, regions, and countries.
---

# Overview

URnetwork is a decentralized privacy network. It's designed to keep users anonymous by default and give them access to all the content in the world.

Humans use the apps (Android, iOS, Chrome). Agents use the API:

- The API spec is at: https://github.com/urnetwork/connect/blob/main/api/bringyour.yml
- The API is hosted at: https://api.bringyour.com

Agents should consider the guidance below when using the API.

## Authentication

When using the API, the user JWT must be passed in an `Authorization: Bearer <JWT>` header.

To get the JWT, ask the human for an auth code, and then convert that auth code to a JWT using the API `/auth/code-login` route. An example curl is below, piped to jq to extract the by_jwt property from the result:

```
curl -X POST https://api.bringyour.com/auth/code-login -d '{"auth_code": "<AUTH CODE>"}' | jq ".by_jwt"
```

The JWT can be stored and reused. If the human asks to refresh the JWT, repeat the process starting with asking for an auth code.

## Proxy Technology

When choosing a proxy technology, consider the following use cases:

| Use Case | Recommended Protocol | Requirement |
| --- | --- | --- |
| **Scraping / Web Browsing** | **HTTPS** | Best for TCP/Web. Always use HTTPS proxy unless there is a specific need to use HTTP. HTTP is usally only needed for specific test environments that do not support HTTPS. From the /network/auth-client response, inside the proxy_config_result object, use the https_proxy_url. No additional username or password are needed. |
| **Low-level Sockets / UDP** | **SOCKS** | Supports TCP+UDP sockets with SOCKS5. From the /network/auth-client response, inside the proxy_config_result object, use the socks_proxy_url or proxy_host and proxy_port, with the username access_token (empty password). The server supports remote DNS resolution (SOCKS5H). |
| **System-wide / OS Level** | **WireGuard** | Routes all IP packets. In the /network/auth-client request, proxy_config.enable_wg must be explicitly set to true. In the response, inside the proxy_config_result object, use the wg_config.config as the complete WireGuard config file. |

When using an HTTPS proxy, be careful that the target technology supports HTTPS. Some technologies support only HTTP and not HTTPS. If the target technology does not support an HTTPS proxy, it is prefereble to consider an alternative library that does support HTTPS than to use an HTTP proxy. As a last resort, the HTTP proxy can be used.

## Location Selection

When using the /network/find-provider-locations route to query locations, always filter the returned locations array by the desired location_type (city, region, or country) to ensure the location_id matches the user's intent.

| Location Type | Requirement |
| --- | --- |
| country | For countries. |
| region | For states, provinces, administrative regions, and metro areas. |
| city | For cities. |

Search for a list of locations using the route /network/find-provider-locations. A curl example is below, piped to jq to extract the locations list.

```
curl -X POST -H 'Authorization: Bearer <JWT>' https://api.bringyour.com/network/find-provider-locations -d '{"query": "<LOCATION NAME>"}' | jq '.locations'
```


# Use case: create a proxy for a country

The API can be used directly to create a HTTPS/SOCKS/WireGuard proxy for a country.

Step 1, search for a list of locations.

Step 2, choose the location of interest and save the country_code property.

Step 3, create a proxy using the saved country code using the /network/auth-client route and setting the proxy_config.initial_device_state to have country_code.

```
curl -X POST -H 'Authorization: Bearer <JWT>' https://api.bringyour.com/network/auth-client -d '{"proxy_config": {"initial_device_state": {"country_code": "<COUNTRY CODE>"}}}'
```


## Use case: create a proxy for a search location

The API can be used directly to search for a location and create a HTTPS/SOCKS/WireGuard proxy. A decision will have to be made to choose the location result that is most desired. Each location has a location_id that is fixed and can be saved in code.


Step 1, search for a list of locations.

Step 2, choose the location of interest and save the location_id property.

Step 3, create a proxy using the saved location_id using the /network/auth-client route and setting the the proxy_config.initial_device_state.location to have connect_location_id.location_id.

```
curl -X POST -H 'Authorization: Bearer <JWT>' https://api.bringyour.com/network/auth-client -d '{"proxy_config": {"initial_device_state": {"location": {"connect_location_id":{"location_id": "<LOCATION ID>"}}}}}'
```


## Use case: create a proxy for a search location, enumerating all the individual egress IPs in that location

This use case is most useful for distributed scraping where using multiple IPs is important.

The API can be used directly to search for a location, enumerate the providers (egress IPs) in that location, and create a HTTPS/SOCKS/WireGuard proxy for each egress IP. A decision will have to be made to choose the location result that is most desired. Each location has a location_id that is fixed and can be saved in code. Additionally each provider has a client_id that is fixed and can be saved in code.

Step 1, search for a list of locations.

Step 2, choose the location of interest and save the location_id.

Step 3, fetch a ranked list of providers (egress IPs) for the location_id using the route /network/find-providers2. The sample size can be set to however many unique providers are needed. A curl example is below, piped to jq to extract the providers list.

```
curl -X POST -H 'Authorization: Bearer <JWT>' https://api.bringyour.com/network/find-providers2 -d '{"specs": [{"client_id": "<CLIENT ID>"}], "count": <COUNT>}' | jq '.providers'
```

Step 4, by looping over each client_id in the list, create a proxy using the client_id using the /network/auth-client route and setting the the proxy_config.initial_device_state.location to have connect_location_id.client_id.

```
curl -X POST -H 'Authorization: Bearer <JWT>' https://api.bringyour.com/network/auth-client -d '{"proxy_config": {"initial_device_state": {"location": {"connect_location_id":{"client_id": "<CLIENT ID>"}}}}}'
```




