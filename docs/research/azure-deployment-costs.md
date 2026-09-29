# Affordable persistent Azure deployment: research findings

Research date: 2026-09-29. Decision ticket: [Find the cheapest viable persistent Azure deployment](https://github.com/EduardoRaygoza/TinyURL/issues/4). This is evidence for later decisions, not an approved architecture or budget.

## Findings

A low-volume managed deployment is a credible inexpensive baseline: Static Web Apps Free, Container Apps Consumption for the ASP.NET Core API and event-driven worker, Azure SQL Basic, Managed Redis B0, and Service Bus Basic. Its modeled fixed services start around **USD 16.58/month with one Redis billing unit**, before variable consumption and optional networking. A two-unit Redis allowance raises that subtotal to **USD 28.26**. Validate the Redis HA configuration and billed quantity in the calculator before treating either as a quote. This investigation does not claim a globally cheapest configuration across every Azure region or subscription offer.

A consolidated Linux VM can also run the API, worker, PostgreSQL, and Redis, but its compute, disk, and public IPv4 subtotal is about USD 21.16 for B1ms or USD 36.42 for B2s. Backups and operations are additional. It therefore does not automatically beat the managed candidate, and minimum viable memory and sustained CPU remain unmeasured.

SQL's recurring free offer can reduce the bill further for intermittent use. Exhausting its allowance can pause the database until next month; enabling paid overage avoids that particular availability limit but creates variable spend. “Continuously deployed” and “always warm” are different requirements. Neither topology eliminates maintenance or platform outages.

## Cost assumptions and reproducibility

Region: **West US 2**, a comparison region only. Currency: **USD**, public Consumption prices; no negotiated discounts, reservations, Spot, tax, currency conversion, student credits, or initial trial credits. Use 730 hours and 730/24 days per modeled month. Verify regional capacity and subscription quotas at provisioning time.

Illustrative small demo: 10,000 redirects/month, fewer than 1,000 management requests, less than 1 GB database contents, 1 GB log ingestion, and less than 5 GB internet egress. These are calculation inputs, not accepted workload targets. Model combined API/worker active allocation at 100 hours/month of 0.25 vCPU and 0.5 GiB (90,000 vCPU-seconds; 180,000 GiB-seconds), assuming all other time scales to zero and no other workloads consume the subscription grants. Request count alone cannot establish compute cost.

Prices below were retrieved from Microsoft's [Retail Prices API](https://learn.microsoft.com/en-us/rest/api/cost-management/retail-prices/azure-retail-prices), filtering `armRegionName eq 'westus2'`, retaining `type == 'Consumption'`, and checking product, meter, and unit. Query [the public endpoint](https://prices.azure.com/api/retail/prices) with `$filter`, follow `NextPageLink`, and distinguish Linux from Windows and Consumption from reservations. Redis's API service name is `Redis Cache`, even for Managed Redis. The retail meter is evidence of price, not proof of deployability or final billed quantity.

| Component / meter | Retrieved retail rate | Modeled recurring contribution |
|---|---:|---:|
| Container Apps Standard vCPU Active Usage | $0.000034/vCPU-second | $0 in illustrative grant-covered usage |
| Container Apps Standard Memory Active/Idle Usage | $0.000004/GiB-second | $0 in illustrative grant-covered usage |
| Container Apps Standard vCPU Idle Usage | $0.000004/vCPU-second | Only applicable qualifying idle replicas; not free warm hosting |
| Container Apps Standard Requests | $0.40/million | $0 below shared grant |
| SQL Database Single Basic, B DTU | $0.161/day | $4.90 |
| Managed Redis Balanced B0 Cache Instance | $0.016/hour per metered unit | $11.68 per 730-hour unit; HA quantity must be confirmed |
| Service Bus Basic Messaging Operations | $0.05/million operations | $0.0015 for illustrative 30,000 operations; polling/retries add operations |
| PostgreSQL Flexible Server Burstable B1MS | $0.017/hour | $12.41 compute |
| PostgreSQL Flexible Server Storage | $0.115/GB-month | $3.68 for 32 GB, before extra IOPS/backup requirements |
| Linux VM B1ms / B2s | $0.0207 / $0.0416 per hour | $15.11 / $30.37 |
| Standard SSD Managed Disk E4 LRS | $2.40/month | $2.40 plus disk transactions |
| Standard IPv4 Static Public IP | $0.005/hour | $3.65 |

Useful verification identifiers: Redis meter `5314b811-d820-55e1-8371-458d4d8d1aaa` (effective 2026-01-01); Container Apps active CPU `2711ef08-ef48-5416-9ef2-f1110cd09c79`; PostgreSQL B1MS `d546f150-a21d-5df8-8ead-459d2f2b0c00`.

The [Managed Redis pricing page](https://azure.microsoft.com/en-us/pricing/details/managed-redis/) distinguishes one-node non-HA and two-node HA prices; its default-location figures differ from the region-filtered API. Do not silently mix those totals. Non-HA is documented for development/test, without the availability SLA; enabling HA is not reversibly switchable off in place. [Architecture](https://learn.microsoft.com/en-us/azure/redis/architecture).

## Candidate comparison

### Managed services, paid small database

Static Web Apps Free serves Angular; browser requests go directly to the API with explicit CORS and authentication, without assuming paid linked-backend features. [Free hosting](https://azure.microsoft.com/en-us/pricing/details/app-service/static/).

Container Apps Consumption can use zero minimum replicas; a queue-triggered worker must have an event scaling rule so work can restart it. Combined monthly grants are 180,000 vCPU-seconds, 360,000 GiB-seconds, and two million requests per subscription. A continuously active 0.25 vCPU/0.5 GiB allocation for 730 hours would cost approximately **$20.57 after those grants**, before requests and other services. Thus a Redis sidecar or permanently polling worker can change the economics materially. [Billing](https://learn.microsoft.com/en-us/azure/container-apps/billing).

SQL Basic provides 5 DTUs and at most 2 GB. That small storage limit makes raw analytics retention and idempotency-record cleanup material design concerns. PostgreSQL Flexible Server B1MS plus 32 GB storage is approximately $16.09 before extras, compared with SQL Basic's $4.90; this is a cost tradeoff, not a database selection. [SQL limits](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-dtu), [PostgreSQL compute/storage](https://learn.microsoft.com/en-us/azure/postgresql/compute-storage/concepts-compute).

Service Bus Basic is sufficient to investigate a queue-based worker. It lacks broker duplicate detection; application idempotency is still required even when upgrading to a tier offering duplicate detection. Send, receive, completion, retries, and polling must be modeled, rather than equating one click with one operation. [Service Bus pricing](https://azure.microsoft.com/en-us/pricing/details/service-bus/), [duplicate detection](https://learn.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection), [delivery guarantees](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-message-loss-and-duplicates).

Use Managed Redis for new planning: legacy Azure Cache for Redis creation is blocked for new customers from April 1, 2026, and Basic/Standard/Premium retirement is September 30, 2028. [Current retirement timeline](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-whats-new).

### Managed services, SQL recurring free allowance

The SQL offer renews monthly with 100,000 vCore-seconds and 32 GB each of data and backup storage. It is not merely an introductory credit. Allowance exhaustion can stop service for the remainder of the month; choosing charged continuation cannot be reverted to the monthly-stop option. Auto-resume adds another cold-start layer; open connections, health checks, or database polling can undermine the intended savings. Compare measured active usage with the allowance before recommending this variant. [SQL free offer](https://learn.microsoft.com/en-us/azure/azure-sql/database/free-offer?view=azuresql).

### Consolidated or hybrid hosting

A VM with PostgreSQL, Redis, API, and worker offers a simple fixed compute bill, but shares CPU, memory, disk, and failure domain. Service Bus Basic can remain managed. Database data belongs on persistent disk with off-machine backups and a tested restore procedure; a disposable cache needs no persistence. B1ms is an unvalidated lower-bound candidate, not a promise that the full stack fits or sustains load. B2s provides more headroom at a higher subtotal. OS updates, certificate renewal, database patching, backup storage, restore time, and burst-credit exhaustion become owner responsibilities.

An API-local Redis container with managed SQL is another cost experiment, but replica-local caches are not a shared cache when the API scales out. Container Apps ephemeral files disappear on shutdown and are unsuitable as the only durable database or analytics queue. A shared self-hosted Redis service adds its own running allocation and availability work. [Storage lifetimes](https://learn.microsoft.com/en-us/azure/container-apps/storage-mounts).

## Complete-bill checklist

The candidate subtotals are not all-inclusive spending ceilings. Confirm these before deployment:

- Application Insights/Log Analytics: illustrative 1 GB fits the documented 5 GB monthly billing-account allowance if unused elsewhere; sampling, retention, alerts, tests, and additional tables can incur charges. [FAQ](https://learn.microsoft.com/en-us/azure/azure-monitor/app/application-insights-faq).
- Network: colocate services; quote internet egress, cross-region traffic, private endpoints, private DNS, NAT, and any load balancer separately. The managed subtotal assumes authenticated TLS public service endpoints and no paid private networking; security acceptability is a later decision. Managed Redis supports public networking and Private Link. [Networking](https://learn.microsoft.com/en-us/azure/redis/private-link).
- Durable storage: SQL Basic includes its limited database capacity; validate backup retention and extra backup billing. VM totals omit off-host backups, snapshots, and disk transactions. Redis must be reconstructible from authoritative records.
- Images, secrets, domain: choose and price registry storage/egress, secret operations, optional domain/DNS, and CI usage. A public image distribution option may avoid an Azure Container Registry fixed bill; no paid registry is assumed in the subtotal.
- Temporary experiments: isolate load generation, larger tiers, extra replicas, verbose telemetry, and Front Door into a time-bounded estimate. Reverting compute does not necessarily shrink database storage or undo HA settings.
- Budgets alert; do not assume they hard-cap costs. Pair alerts with explicit replica ceilings, retention controls, abuse limits, and an agreed emergency response. [Budget automation](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/manage-automation).

## Capacity path and Front Door research inputs

Start by measuring cold and warm latency independently, cache hit rate, database CPU/IO/connections, queue age, worker throughput, and cost per test. Raising replica counts can multiply database connection pools. Increase concurrency only alongside those measurements. Moving from one API replica to several requires shared state for caching and rate-limiting guarantees; VM-to-managed database migration is a data migration, not merely an instance-size change. API resources, replicas, database tiers, cache size, and worker concurrency are distinct capacity controls.

Front Door Standard has a published **$35/month base**, Premium **$330/month**, plus usage charges. This alone can exceed the small baseline, so deferral is financially meaningful. It provides global ingress, TLS, dynamic acceleration, routing, and edge caching; Premium adds capabilities including managed WAF protection and Private Link. Use Standard/Premium planning, not retiring Classic. [Pricing](https://azure.microsoft.com/en-us/pricing/details/frontdoor/), [best practices](https://learn.microsoft.com/en-us/azure/frontdoor/best-practices).

A concrete later integration design should specify custom DNS and certificate validation, HTTPS origin hostname, separate static/API/redirect routes, a cheap health endpoint, and origin access restrictions. Front Door backend network restrictions plus validation of the specific Front Door instance identity prevent bypass more effectively than checking a spoofable header alone. Test TLS, probes, forwarded headers, authentication, direct-origin denial, and rollback before DNS cutover. [Origin security](https://learn.microsoft.com/en-us/azure/frontdoor/origin-security).

Research recommendation for that decision: initially cache only versioned static assets and bypass management and redirect caching. Edge-served redirects would bypass application event capture, and stale cached results could conflict with disabling/expiration. If redirect caching is later proposed, explicitly redesign analytics coverage, TTLs, and purge guarantees. Front Door does not require caching to provide routing/acceleration benefits. [Caching behavior](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching).

## Remaining human decisions and verification

Choose database preference, tolerated cold-start delay, availability expectations for a portfolio demo, willingness to operate a VM or non-HA cache, network controls, and monthly spending ceiling. These belong to the subsequent human-led tickets. No Azure resources were provisioned and no performance or availability claims were measured. Before deployment, save a calculator estimate for the chosen region and configuration, confirm Redis HA quantity, quotas and all ancillary services, then validate actual daily costs and revise the estimate. The research question is answered with comparable candidates and explicit uncertainties; architecture remains open.
