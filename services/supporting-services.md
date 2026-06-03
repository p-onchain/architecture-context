# Supporting Services

> Grouped reference for smaller/supporting services that don't need individual deep-dive docs.

---

## order-responder
- **Repo:** p-blackswan/order-responder
- **Purpose:** Aggregates match and status events from Kafka, builds order response for fast-cancel flow
- **Consumes:** `order.events.match` (all markets), `order.events.status` (all markets)
- **Redis:** DB 7 (shared order cache with order-api)
- **Notes:** Scales horizontally via shared Kafka consumer group. Cooperative-sticky rebalance.

## match-forge
- **Repo:** p-blackswan/match-forge
- **Purpose:** Event persistence — consumes match engine events from Redpanda, writes as append-only event store to PostgreSQL
- **DB:** PostgreSQL (`match_forge`), with S3 archival (partitioned, daily rotation)
- **Notes:** Internal HTTP API gated by token for commission-cost-tracker. Archiver feature for old data → S3 + Glue.

## conditional-order
- **Repo:** p-blackswan/conditional-order
- **Purpose:** Stop-loss, take-profit, and other conditional order triggers
- **Port:** gRPC :50052
- **Consumes:** `orderbook.match_price` (trigger evaluation), `order.events.status` (order tracking)
- **Calls:** order-api (gRPC :60060) to submit triggered orders
- **DB:** PostgreSQL (`conditional_orders`), Redis DB 6

## config-service
- **Repo:** p-blackswan/config-service
- **Purpose:** Dynamic configuration management (market params, limits, feature toggles)
- **Produces:** `config-service-events` to Kafka
- **Deployed as:** config-service + config-service-worker
- **Notes:** HTTP REST API (not gRPC). Consumed by order-api and read-mono.

## notification / notif2
- **Repo:** p-blackswan/notification, p-blackswan/notif2
- **Purpose:** Multi-channel notification delivery (push, SMS, email, in-app)
- **notif2:** Newer notification service replacing legacy notification
- **Deployed as:** notification-api + notification-worker (or notify-api + notify-iys-worker)
- **Notes:** notification-command handles DLQ + bulk push campaigns

## alarm-service
- **Repo:** p-blackswan/alarm-service
- **Purpose:** User-defined price alarms and custom alert rules
- **Deployed as:** alarms-server, alarm-query, quick-alarms-worker, custom-alarms-worker

## campaign-service
- **Repo:** p-blackswan/campaign-service
- **Purpose:** Promotions, coupon campaigns, referral rewards
- **Deployed as:** campaign-service, campaign-projection, campaign-query, campaign-outbox-worker
- **Events:** campaign created/updated/status changed, coupon claimed/uploaded/withdrawn

## staking-service
- **Repo:** p-blackswan/staking-service
- **Purpose:** Pool-based crypto staking (create pools, stake, redeem, distribute rewards)
- **Deployed as:** staking-service, staking-query, staking-distributor
- **Proto:** v1 and v2 versions (v2 adds APR management, more event types)

## invoice-service
- **Repo:** p-blackswan/invoice-service
- **Purpose:** Invoice generation for trades and transactions
- **Deployed as:** invoice-service-api, invoice-service-worker

## bank-integration
- **Repo:** p-blackswan/bank-integration
- **Purpose:** Bank API integration for fiat deposits/withdrawals
- **Deployed as:** bank-integration, bank-integration-query, bank-integration-proxy, bank-integration-mock

## custody-integration
- **Repo:** p-blackswan/custody-integration
- **Purpose:** HSM/custody signing bridge for crypto transactions
- **Deployed as:** custody-integration, custody-integration-query

## reconciliation
- **Repo:** p-blackswan/reconciliation
- **Purpose:** Balance reconciliation between wallet, ledger, and external sources
- **Deployed as:** reconciliation-server, reconciliation-job

## sanctions / elliptic-screener
- **Repo:** p-blackswan/sanctions, p-blackswan/elliptic-screener
- **Purpose:** AML/compliance screening, sanctions list checking
- **Deployed as:** sanctions, elliptic-screener-scheduler, elliptic-screener-worker

## commission-update-worker
- **Repo:** p-blackswan/commission-update-worker
- **Purpose:** Consumes CommissionEvent from Kafka, writes commission rates to Redis hash
- **Consumes:** `user-commission-events`
- **Redis:** DB 2 (commission_rates hash, read by wallet)

## commission-cost-tracker
- **Repo:** p-blackswan/commission-cost-tracker
- **Purpose:** Tracks commission costs per trade for financial reporting

## global-price-tracker
- **Repo:** p-blackswan/global-price-tracker
- **Purpose:** Fetches and serves global crypto prices from external sources
- **Deployed as:** global-price-tracker-api, global-price-tracker-worker

## heimdall
- **Repo:** p-blackswan/heimdall (Rust)
- **Purpose:** Domain-agnostic external data hub. Fetches, validates, stores, caches, and serves data from external sources (CoinGecko, sentiment indices, etc.)
- **Port:** gRPC :50099
- **Notes:** Written in Rust. API-key authentication required in non-local environments.

## ws-hub
- **Repo:** p-blackswan/ws-hub
- **Purpose:** WebSocket hub for streaming real-time data to web/mobile clients (legacy)
- **Notes:** JWT-based auth. Being supplemented by wapi for API-key clients.

## wapi
- **Repo:** p-blackswan/wapi
- **Purpose:** Low-latency WebSocket service for API-key trading clients
- **Architecture:** Consumes directly from internal Redpanda (not via MSK bridge)
- **Auth:** API-key on /v1/user (KrakenD validates), anonymous on /v1/stream
- **Notes:** Independent of ws-hub. Hexagonal architecture.

## gopanel
- **Repo:** p-blackswan/gopanel
- **Purpose:** Back-office admin panel
- **Deployed as:** gopanel-backend, gopanel-frontend, gopanel-example-service
- **Also:** gopanel-marketing-backend, gopanel-marketing-frontend

## market-api
- **Repo:** p-blackswan/market-api
- **Purpose:** Market metadata API (generates market-configs YAML)

## feature-flags
- **Repo:** p-blackswan/feature-flags
- **Purpose:** YAML-based feature flag definitions (consumed by flagd/OpenFeature)

## input-validator
- **Repo:** p-blackswan/input-validator
- **Purpose:** gRPC service for validating user inputs (addresses, amounts, etc.)

## recurring-api
- **Repo:** p-blackswan/recurring-api
- **Purpose:** Recurring buy (DCA) service — CQRS + Event Sourcing

## partner-integration
- **Repo:** p-blackswan/partner-integration
- **Purpose:** External partner integrations (Besiktas, Mohikan)

## kep-service
- **Repo:** p-blackswan/kep-service
- **Purpose:** KEP (legal registered email) integration for the legal department

## mkk
- **Repo:** p-blackswan/mkk
- **Purpose:** MKK (Merkezi Kayıt Kuruluşu) integration for regulatory reporting
