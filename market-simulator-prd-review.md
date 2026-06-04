# Market Simulator PRD — Doğrulama & İyileştirme Raporu

**Tarih:** 2026-06-05
**Yöntem:** Canlı phoenix-test EKS cluster (ReadOnly) + gerçek repo'lar (order-api, match, wallet, transaction, config-service, user-service, market-configs, read-mono, proto-hub) + **exchange-automation-tests** (ampirik ground truth) çapraz kontrolü.
**Fact hiyerarşisi:** canlı cluster Service objeleri (bağlantı: DNS/port) > gerçek repo (proto/method/DTO) > exchange-automation-tests (gerçek uçtan uca çağrı dizisi).

> ⚠️ **Önemli:** `~/architecture-context` context pack'inin **portları ve namespace'leri sistematik olarak yanlış** çıktı (order-api 9000→aslında 3000, wallet 50058→50051, match 50059→prod'da 50051, match-core "aktif"→arşivli, config/user/ticker servisleri "blackswan ns"→aslında shelby/saul). PRD'nin ilk incelemesinde bu pack'e güvenip yaptığım "düzeltmeler" hatalıydı; **PRD'nin port/iddialarının çoğu doğruymuş.** Aşağıdaki her şey canlı kanıta dayanıyor.

---

## 1. Doğrulanmış Bağlantı Gerçekleri (canlı cluster)

| Servis | Namespace | Port | Not |
|---|---|---|---|
| **order-api** | blackswan | **HTTP :3000**, gRPC :50051 | `POST /order` HTTP'de. gRPC sadece conditional-order callback'i, order-create DEĞİL. `ENVIRONMENT=test`. |
| **wallet** | blackswan | gRPC :50051 | |
| **match-{cur}-{pay}** | blackswan | gRPC :50051 | Örn. `match-btc-try:50051`. Health: `grpc.health.v1.MatchEngine`. (Repo/local default 50059; **prod 50051**.) |
| **config-service** | **shelby** | HTTP :8080 | REST-only (gRPC yok). `config-service.shelby.svc.cluster.local:8080` |
| **user-state-query** | shelby | gRPC :50051 | CanTrade buradan okunur |
| **user-service** | shelby | (command, doğrulanamadı) | UUID'ler server-side üretilir |
| **ticker-query** | saul | gRPC :50051 | |
| **global-price-tracker-api** | saul | gRPC :50051 | Harici fiyat referansı |
| **krakend-gateway** | blackswan | :8080 / :9090 | **Rate-limit SADECE burada** (x-user-id/JWT keyed, Redis-backed) |

order-api env'leri: `MATCH_SERVICE_ADDR_BASE=match-%s:50051` (market = `{currency}-{payment}`), `WALLET_SERVICE_ADDR=wallet:50051`, `CONFIG_URL=http://config-service.shelby.svc.cluster.local:8080`.

**Market sayısı (phoenix-test):** Bu env'de **~27 market gerçekten canlı** (= `markets/test/*.yaml` = 27 ≈ 27 match Deployment ≈ 29 ready endpoint). Cluster'da 253 `match-*` Service objesi görünür ama çoğu **stale placeholder** — gerçek sinyal *ready endpoint'i olan* Service'ler. (Prod'da 259 market.) "Tradeable" pratik kuralı: config-service'te `unlisted!=true && suspended yok` **VE** match Service'inin ready endpoint'i var.

**Payment kodu = `try` (DOĞRULANDI):** order body'de `payment` alanı **`try`** olmalı. Kanıt: `match-btc-try` configMap'i `PAYMENT=try, CURRENCY=btc, MARKET=btc-try`; ve geçen automation-test order örneği `payment=try`. config-service seed blob'unun `btc_tl` / `payment:tl` kullanması yalnızca **config-service'in iç katalog keying'i** (legacy/display) — order-api/match `try` kullanır.

---

## 2. Beş Açık Sorunun Cevabı

### Q1 — Test user oluşturma / KYC bypass → **Pre-seed (DB) gerekli; API ile sabit UUID İMKANSIZ**
- user-service **tüm** create yollarında UUID'yi server-side üretir (`uuid.NewV7()`). Yani API ile **istediğin sabit UUID'yi veremezsin**.
- Yeni user `kyc_status=unverified` başlar; **test env'de KYC bypass YOK** (gerçek `pkyc` provider — `kyc-test.int.blackswan.run`).
- **`CanTrade` user-service'te değil:** read-mono `user-state` projection'ında hesaplanır (`permission_evaluator.go`): `active` + `kyc verified|courier_process` + sözleşme imzalı + **fiat deposit var** + market/fraudbank lock yok. Kafka `user-state-events` ile yayınlanır, order-api cache'ler, `CanTrade=false` ise **403**.
- **Cevap/öneri:** Sabit UUID'li ~10 user'ı **doğrudan DB'ye seed et** (verified/active/contract/fiat deposit state'iyle) — bu zaten **exchange-automation-tests'in yaptığı şey**: `data/users/test-users.ts` içinde sabit UUID'li `NAMED_USERS` registry'si ("Must exist in test env"). **En pratik yol: simülatör bu mevcut test user'larını yeniden kullansın.** Trade-eligibility'yi `LockCommandService.CreateLock/DeleteLock(market)` ile toggle edebilirsin.

### Q2 — Bakiye yükleme (AddAvailableFund external mi?) → **Evet, wallet'ı DOĞRUDAN çağır**
- wallet gRPC server'da **auth interceptor YOK** (sadece panic-recovery), NetworkPolicy/mTLS yok. `wallet:50051`'e erişen herkes çağırabilir.
- **Cevap:** Simülatör `wallet.AddAvailableFund`'ı **doğrudan** çağırsın, `operation_type=DEPOSIT_FIAT (=2)` ile — **tüm assetler için (crypto dahil)**. automation-tests aynen bunu yapıyor. `transaction` üzerinden gitmek daha ağır/async ve sonunda yine aynı method'u çağırıyor.
- ⚠️ `DEPOSIT_CRYPTO_FINALIZE (=1)` AddAvailableFund yolu **değildir** (o `ReleaseFund`). Crypto fonlama için bile `DEPOSIT_FIAT` kullan.
- WalletOperation mesajı: `{user_id, tx_id (üretilen UUID), asset_code (lowercase), amount (string), operation_type=2, event_time, commission:"0"}`.
- (Trade reservation order-api içinde `wallet.ReserveTradeFund` / `RESERVE_TRADE=10` ile yapılır — simülatörün uğraşması gerekmez, sadece `POST /order` atar.)

### Q3 — Boş orderbook'ta seed/initial price → **HİÇBİR YERDE yok; simülatör kendi tanımlamalı**
- match'te seed price yok: boş market boş orderbook'la başlar, **ilk trade'e kadar `orderbook.match_price` yayınlamaz** (`PublishMatchPrice` sadece eşleşme olunca).
- config-service market'lerinde sadece `precisions`+`steps` var, **fiyat yok**. market-configs YAML'lerinde de fiyat yok. read-mono read-side (ticker `first/open`'ı clickhouse'tan 24s aggregate hesaplar).
- **Cevap:** Seed/reference price **simülatörün kendi config'inde** olmalı (automation-tests'teki `data/prices/market-prices.ts` gibi: btc-try 3766000, eth-try 126000, sol-try 5300, btc-usdt 95000, usdt-try 43...). Simülatör market'i bu fiyat etrafında ilk BID/ASK ile bootstrap eder. İstersen daha gerçekçi başlangıç için `global-price-tracker-api`'den (saul) harici fiyat okunabilir — ama bu orderbook fiyatı değil, sadece referans.

### Q4 — Rate limiting + topoloji → **In-cluster order-api:3000 KrakenD'yi bypass eder, limit YOK**
- Rate-limit **sadece KrakenD'de** (per-endpoint, Redis, x-user-id/JWT keyed). order-api'de **hiç** rate-limit yok.
- **Cevap & tasarım kararı:** Simülatör pod'u in-cluster `order-api:3000`'e doğrudan vurursa **KrakenD'yi tamamen atlar → throttle olmaz** (throughput için iyi). Ama bu, **gateway rate-limit code path'ini test ETMEZ**. Eğer rate-limit'i de egzersiz etmek istiyorsan `krakend-gateway:8080`'e JWT ile gitmen gerekir. PRD bu seçimi net belirtmeli — **öneri: v1'de doğrudan order-api:3000 (basit, throttle yok), rate-limit testi kapsam dışı.**

### Q5 — mahmut-try → **Test-only sandbox; ana akışta KULLANMA**
- Canlı (`match-mahmut-try`, 35g). `markets/test/mahmut-try.yaml` (prod'da yok). Özelliği: `CONFIG_URL` override'ı özel bir **merged config UUID**'sine (`/merged/019c760c-6b66-7cf3-a860-f8d293d402c4`) bakıyor + kısıtlı kaynak (350m cpu/400Mi). `mahmut` gerçek bir asset değil.
- **Cevap:** Simülatör **gerçek test pair'lerine** odaklansın (btc-try, eth-try, sol-try, btc-usdt, usdt-try...). mahmut-try'ı dışarıda bırak (özel merged-config'li sandbox, temsili değil).

---

## 3. PRD'deki Diğer Düzeltmeler

| PRD iddiası | Gerçek (kanıtlı) |
|---|---|
| order-api :3000 | ✅ **Doğru** (canlı Service :3000). |
| `match-{market}:50051` | ✅ Doğru (prod :50051; repo default 50059). |
| match-core matching engine | ❌ **match-core ARŞİVLİ**; aktif repo `p-blackswan/match`. |
| match WAL → `order.requests` | ⚠️ Topic **market-suffixed**: `order.requests.{market}` (örn. `order.requests.btc-try`). Düz `order.requests` yok. |
| match Kafka consume etmez | ❌ **Eder**: kendi WAL'ını replay eder (crash recovery) + `config-service-events` consume eder. |
| Event topic'leri `*.{market}` suffix'li | ❌ **Shared/suffix'siz**: `order.events.match`, `order.events.status`, `orderbook.state`, `orderbook.match_price`. Consumer'lar payload'tan market'e göre filtreler. |
| order body `POST /order` | ✅ **Singular `/order`** (order-api). KrakenD/BFF ve forwarder ise `/orders` (plural). |
| X-User-ID header | ✅ Doğru — ama order-api **header UUID == body.user_id** şartını koşar (yoksa 403). KrakenD path'i JWT raw token kullanır. |
| config GET /default market listesi | ⚠️ Düz `/default` market vermez; **`GET /default?basic=true`** verir. Market'lerde `precisions`+`steps` var, **limit/price yok**. |
| ~27 active market | ✅ TEST env = 27 (doğru). Prod = 259. Cluster'da 253 Service ama 29 ready. |
| read-mono "30+ projection" | ❌ **15 projection** (+21 query, 25 domain, 43 deployable unit). |
| order-api 30+ projection besler | Doğru sayı: yukarıdaki gibi 15 projection / 21 query. |

**Order-create contract (ground truth):** `POST http://order-api:3000/order`, header `X-User-ID: <uuid>`, JSON body:
```json
{ "user_id":"<uuid>", "currency":"btc", "payment":"try", "price":"3766000",
  "quantity":"0.001", "order_side":"ORDER_SIDE_BID", "order_type":"ORDER_TYPE_LIMIT",
  "time_in_force":"GTC", "total_price":"..." }
```
Enum'lar: `order_side ∈ {ORDER_SIDE_BID, ORDER_SIDE_ASK}`, `order_type ∈ {ORDER_TYPE_LIMIT, ORDER_TYPE_MARKET, ORDER_TYPE_LIMIT_MAKER}`, `time_in_force ∈ {GTC, IOC, FOK}` (market order GTC olmalı). Market = ayrı `currency`+`payment` (combined symbol değil). Cancel: `DELETE /order?order_id=&currency=&payment=` + `X-User-ID`.

---

## 4. EN BÜYÜK Plan İyileştirmesi: Sıfırdan yazma, mevcut iki repo'yu kullan

**`p-blackswan/exchange-automation-tests`** simülatörün istediğinin **~%80'ini zaten yapıyor:**
- order-api HTTP client (`POST /order`, X-USER-ID) — `adapters/http/order.client.ts`
- wallet `AddAvailableFund` ile fonlama — `services/wallet/wallet-deposit.service.ts`, eşik bazlı (`easy-trade.service.ts`)
- **orderbook seed script'i** — `scripts/seed-orderbooks.ts` (market başına 20 level/side)
- seed fiyatlar — `data/prices/market-prices.ts`
- sabit test user registry — `data/users/test-users.ts`
- config'ten dinamik market discovery + ready-match filtresi — `easy-trade.service.ts:fetchActiveMarkets`
- event doğrulama (Kafka `order.events.status`, `ORDER_COMPLETED`)

**`p-blackswan/order-event-forwarder`** (Go) ise daha alt seviye yolu gösteriyor: `wallet.ReserveFund` + `match.CreateOrder` doğrudan gRPC (order-api'yi atlayarak), market başına `match-{cur}-{pay}:50051`.

> **Öneri:** PRD §11'in **Go + yeni repo** kararı doğru — koru. Ama "sıfırdan" yazma; iki repo'yu **farklı amaçlarla** kullan:
> 1. **(Önerilen) Servisi `order-event-forwarder` iskeleti üzerine kur.** O zaten *deploy edilebilir, uzun-süre çalışan bir Go servisi* ve `wallet` + `match.CreateOrder` gRPC çağrılarını, market client manager'ı, wallet reserve'ü yapıyor — PRD §11'in Go/zerolog/proto-hub standardıyla birebir uyumlu. Üstüne sürekli MM döngüsü + senaryolar + Prometheus + seed manager ekle.
> 2. **`exchange-automation-tests`'i SADECE reçete/sabit kaynağı olarak madenle** — onu productionize ETME (o bir Playwright/Cucumber E2E test suite'i, daemon değil). Ondan al: funding çağrısı (`AddAvailableFund/DEPOSIT_FIAT`), sabit user UUID'leri (`NAMED_USERS`), seed fiyatları (`market-prices.ts`), order contract'ı, event topic'leri, market-discovery filtresi. Bu suite beş açık sorunun hepsini ampirik olarak yanıtladı.
> Her iki durumda da: user sorununu (Q1) çöz çünkü ikisi de **var olan seed'li user'lara güveniyor** — simülatör de aynısını yapsın.

---

## 5. İyileştirilmiş Faz Planı (somut)

**Faz 0 — Önkoşul netleştirme (PRD'de eksikti):**
- Test user'ları: exchange-automation-tests'in `NAMED_USERS`'ını yeniden kullan **veya** DB-seed script'i yaz (verified+active+contract+fiat+lock'suz). API register **işe yaramaz** (Q1).
- Seed fiyat config'i hazırla (Q3) — market başına reference price.

**Faz 1 — MVP (btc-try):**
- wallet `AddAvailableFund(DEPOSIT_FIAT)` ile seed user'ları fonla (doğrudan gRPC `wallet:50051`).
- `POST http://order-api:3000/order` + `X-User-ID` ile symmetric MM (config-driven spread/level/tick).
- Boş kitabı seed fiyat etrafında bootstrap et (Q3).
- Prometheus metrikleri. K8s deploy **sadece phoenix-test**, `ENVIRONMENT != prod` guard.

**Faz 2 — Multi-market + discovery:**
- `GET http://config-service.shelby.svc.cluster.local:8080/default?basic=true` ile market listesi → `unlisted!=true && suspended yok` filtrele → **ayrıca match Service ready endpoint kontrolü** (253 Service'in sadece ~29'u canlı).
- 27 test market'i için parallel MM. Senaryolar (normal, volatility spike).

**Faz 3 — Projection & validation:**
- E2E latency: `POST /order` → Kafka `order.events.status` (`ORDER_COMPLETED`) arası.
- Smoke-test modu (CI). Gelişmiş senaryolar (cancel storm, batch, cross-market).

**Güvenlik/safeguard:** order-api:3000'e in-cluster doğrudan erişim KrakenD'yi bypass eder (rate-limit yok) — kabul edilen tasarım. Prod'a deploy engeli: namespace pin + `ENVIRONMENT` guard + sabit/seed user UUID'leri prod'da yok.

---

## 6. Kalan / yeni açık noktalar
- **Payment kodu:** ✅ **Çözüldü → `try`** (bkz. §1; `match-btc-try` configMap + geçen automation-test order'ı). config-service'in iç `tl` keying'i order contract'ını etkilemez.
- **NAMED_USERS yeniden kullanımı (build-time doğrulanacak varsayım):** Simülatörü bu sabit UUID'lere bağlamadan önce, `user-state-query.shelby:50051`'den birinin `CanTrade=true` olduğunu sorgula. State drift olmuşsa kendi seed user'larını oluştur.
- **match prod port'u 50051 nasıl set ediliyor:** repo default 50059; prod Service 50051. Deployment'ın `GRPC_PORT=50051` override'ı shallow clone'da görünmedi (sadece minikube/base 50059). Bağlantı için canlı `:50051` kesin; kod tarafı bilinçli not.
- **exchange-automation-tests wallet client kısmen stale:** `ReserveFund`/`CompleteFund` çağırıyor (mevcut proto'da trade-reserve = `ReserveTradeFund`, `CompleteFund` yok). Reuse ederken trade-reserve method adını canlı proto/reflection ile doğrula. (AddAvailableFund kısmı güncel.)
