# Project Bifrost: Multi-Platform Data Integration

**Status:** Live in production since April 2026
**Role:** Sole engineer
**Employer:** Triquetra Health (health/wellness e-commerce)

> This is a case study describing proprietary work. Production code is not published. For code demonstrating the underlying patterns, see the [shopify-oauth-token-caching](../shopify-oauth-token-caching) sample repository.

## The Problem

The operations team was spending 7 to 10 hours per week manually pulling sales and inventory data from three different SaaS platforms into Google Sheets to produce a unified weekly performance report for leadership. Each platform had its own login, its own UI export flow, and its own edge cases:

- **Shopify Plus** (GraphQL Admin API): the primary sales channel
- **Amazon Seller Central** (SP-API): FBA and FBM sales, plus separate AWD and FBA inventory endpoints
- **ShipHero** (GraphQL): warehouse-level inventory and fulfillment status
- **TikTok Shop** (API): additional sales channel

The report was often late, often wrong, and always painful. Errors compounded because reconciliation was manual and no one person could hold the full data model in their head.

## The Approach

I built a Google Apps Script-based integration that runs on hourly and daily schedules, pulling data from all four platforms into a unified spreadsheet with automated pivot tables and KPI dashboards.

Design decisions worth naming:

- **Google Apps Script over a cloud function.** Apps Script sits inside the operations team's Google Workspace and their existing Sheets. No new infrastructure to maintain. No additional cost. Trigger management is native.
- **Separate ingestion functions per platform.** Each platform got its own module with its own auth flow, its own pagination handling, and its own rate-limit strategy. Failure in one platform does not break the others.
- **Client credentials grant for Shopify** (rather than legacy custom app tokens). Legacy custom apps were deprecated as of January 2026. The new pattern requires `POST /admin/oauth/access_token` with client_id and client_secret, returning a `shpat_` token valid for 24 hours. Token is cached in Apps Script CacheService for 23 hours to avoid unnecessary auth calls.
- **Exponential backoff on 429 responses.** Amazon SP-API rate limits are notoriously spiky. Backoff pattern: 1s, 2s, 4s, 8s, then fail with clear error logged to a monitoring tab.
- **Pagination in a single loop with a fixed max iteration count.** Prevents infinite loops if a paginated endpoint misbehaves.

## Tech Stack

- **Runtime:** Google Apps Script (V8 engine)
- **APIs:** Shopify GraphQL Admin API, Amazon SP-API (Reports and Inventory endpoints), ShipHero GraphQL, TikTok Shop API
- **Storage:** Google Sheets (both source-of-truth data and dashboards)
- **Auth:** OAuth 2.0 client credentials grant (Shopify), IAM roles (Amazon), API key (ShipHero)
- **Trigger management:** Apps Script installable triggers (hourly for inventory, daily for sales reports)
- **Monitoring:** Structured error logging to a dedicated Sheets tab

## Impact

- **7 to 10 hours per week reclaimed** for the operations team, immediately usable for higher-value work
- **Unified dashboard** with cross-platform revenue, inventory, and fulfillment KPIs updated hourly
- **Zero manual reconciliation** since deployment (previously the report had known-bad data ~30 percent of the time)
- **Foundation for automation roadmap.** The reliable data model enabled subsequent projects (fraud detection, address validation, creator gifting) to consume this pipeline as a source of truth.

## Lessons Learned

**1. Small companies rarely need cloud infrastructure for internal reporting.**
Apps Script inside the team's existing Google Workspace outperformed a Lambda-based alternative for this use case: no infrastructure cost, no deployment complexity, native trigger management. The right tool is often the boring one.

**2. Rate limit strategy differs meaningfully by platform.**
Shopify's REST API is lenient. Shopify's GraphQL API uses a cost-based bucket model. Amazon SP-API applies per-endpoint rate limits with token bucket refill. ShipHero uses a 400 credits per minute model. A generic "back off on 429" implementation would have worked poorly. Each platform got its own strategy.

**3. Documentation is part of the deliverable.**
Every function has inline documentation. Every workflow has a run book. Every error has a resolution path documented. When I move on, the next person can operate this without me.

## What's Next

- Migrate TikTok integration once the `shop_id invalid` issue resolves
- Extend the ShipHero integration to include operator hold and address hold status (already partially built for the [Address Validation Pipeline](../address-validation-pipeline))
- Consider migrating to a proper cloud function once volume outgrows Apps Script quotas

## References

- Shopify GraphQL Admin API: [https://shopify.dev/docs/api/admin-graphql](https://shopify.dev/docs/api/admin-graphql)
- Shopify client credentials grant: [https://shopify.dev/docs/apps/build/authentication-authorization/client-secrets](https://shopify.dev/docs/apps/build/authentication-authorization/client-secrets)
- Amazon SP-API: [https://developer-docs.amazon.com/sp-api](https://developer-docs.amazon.com/sp-api)
