# Notes

## Front Door

Log query - Display all Errors

```SQL(Kusto Query Language)
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "FrontdoorAccessLog"
| where isReceivedFromClient_b == true
| where toint(httpStatusCode_s) >= 400
| extend ParsedUrl = parse_url(requestUri_s)
```

[503 responses from Azure Front Door only for HTTPS:](https://docs.microsoft.com/en-us/azure/frontdoor/troubleshoot-issues#503-responses-from-azure-front-door-only-for-https)

[Origins and origin groups in Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/origin?pivots=front-door-classic)

[Backend TLS connection (Azure Front Door to backend)](https://docs.microsoft.com/en-us/azure/frontdoor/end-to-end-tls#backend-tls-connection-azure-front-door-to-backend)

[End-to-end TLS with the v2 SKU](https://docs.microsoft.com/en-us/azure/application-gateway/ssl-overview#end-to-end-tls-with-the-v2-sku)
