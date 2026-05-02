# 📋 BÁO CÁO CHI TIẾT CÁC MODULE — ANTI-RUG INTELLIGENCE LAYER

**Dự án:** Migration Sniper Bot  
**Ngày:** 2026-05-02  
**Tổng số modules:** 5 modules + 3 hệ thống hỗ trợ

---

## MODULE TỔNG QUAN (Core)

### Nhiệm vụ
Module Core đóng vai trò **trung tâm điều phối**, kết nối tất cả 5 modules anti-rug lại với nhau. Nó quyết định token nào được phép mua, token nào bị chặn, và quản lý cấu hình chung cho toàn bộ hệ thống.

### Các file code thuộc module Core

#### 📄 `src/modules/anti_rug/mod.rs` (28 dòng)
- **Nhiệm vụ:** File gốc (root) của toàn bộ Anti-Rug layer. Đây là file "mục lục" — nó khai báo và export tất cả sub-modules để các phần khác của bot có thể sử dụng.
- **Chi tiết:**
  - Dòng 15-22: Khai báo 8 sub-modules (`config`, `filter_result`, `holder_analyzer`, `dev_wallet_profiler`, `genesis_detector`, `metadata_checker`, `panic_sell`, `pre_buy_filter`)
  - Dòng 24-27: Re-export các types quan trọng nhất (`AntiRugConfig`, `FilterVerdict`, `evaluate_token`, `PanicSellContext`, v.v.) để code bên ngoài gọi trực tiếp mà không cần đường dẫn dài

#### 📄 `src/modules/anti_rug/config.rs` (70 dòng)
- **Nhiệm vụ:** Định nghĩa **toàn bộ tham số cấu hình** của hệ thống Anti-Rug. Mỗi module có thể bật/tắt độc lập, mỗi ngưỡng có thể điều chỉnh.
- **Chi tiết:**
  - `enabled` (bool): Công tắc tổng — tắt toàn bộ anti-rug
  - `warn_only` (bool): Chế độ cảnh báo — log nhưng KHÔNG chặn mua (dùng khi calibration)
  - `max_top10_holder_pct` (f64): Ngưỡng % holder tối đa (mặc định 30%)
  - `min_dev_tx_count` (u64): Số TX tối thiểu ví dev cần có (mặc định 10)
  - `max_genesis_buy_pct` (f64): % supply tối đa mua trong genesis block (mặc định 50%)
  - `max_clustered_wallets` (u32): Số ví tối đa được phép mua cùng block (mặc định 3)
  - `panic_sell_jito_tip_lamports` (u64): Phí tip Jito cho panic sell (mặc định 1,000,000 = 0.001 SOL)
  - `filter_timeout_ms` (u64): Timeout cho mỗi RPC call (mặc định 1500ms)

#### 📄 `src/modules/anti_rug/filter_result.rs` (72 dòng)
- **Nhiệm vụ:** Định nghĩa **kiểu dữ liệu kết quả** mà tất cả modules trả về. Đây là "hợp đồng" chung giữa các modules.
- **Chi tiết:**
  - `FilterVerdict` (enum): 3 trạng thái — `Pass` (an toàn), `Fail(reason)` (nguy hiểm), `Warn(reason)` (đáng ngờ)
  - `AntiRugFilterResult` (struct): Chứa tất cả kết quả từ 5 modules: `top10_holder_pct`, `dev_tx_count`, `genesis_buy_pct`, `genesis_bundle_detected`, `has_metadata_uri`, `metadata_uri`, `token_name`, `filter_duration_ms`
  - `disabled_pass()`: Tạo result Pass khi anti-rug bị tắt hoàn toàn

#### 📄 `src/modules/anti_rug/pre_buy_filter.rs` (184 dòng)
- **Nhiệm vụ:** **Orchestrator** — điều phối chạy tất cả 4 pre-buy modules (M1, M3, M4, M5) song song và tổng hợp kết quả thành 1 verdict duy nhất.
- **Chi tiết:**
  - `evaluate_token()` (dòng 25-101): Entry point duy nhất — nhận `mint`, `dev`, `creation_slot` → trả về `AntiRugFilterResult`
  - Dòng 38-68: Dùng `tokio::join!` chạy 3 modules song song (M1 + M3 + M5), M4 chạy riêng vì cần tham số extra
  - `build_verdict()` (dòng 103-175): Thu thập tất cả kết quả → nếu bất kỳ module nào trả Err → FAIL, nếu metadata thiếu → WARN, còn lại → PASS
  - `get_total_supply()` (dòng 177-183): Helper lấy total supply qua RPC cho M4

---

## MODULE 1: HOLDER CONCENTRATION ANALYZER

### Nhiệm vụ
Phân tích **phân bố sở hữu** của token trước khi mua. Nếu top 10 ví lớn nhất nắm giữ quá nhiều supply (>30%), token có nguy cơ bị dump → FAIL.

**Ví dụ:** Token XYZ có 1,000,000 supply. Top 10 ví nắm 450,000 (45%) → FAIL. Bot sẽ KHÔNG mua token này.

### Các file code thuộc Module 1

#### 📄 `src/modules/anti_rug/holder_analyzer.rs` (132 dòng)
- **Nhiệm vụ:** Kết nối Solana RPC để lấy danh sách holders lớn nhất và tính % concentration.
- **Chi tiết:**
  - `HolderAnalysis` (struct): Kết quả phân tích gồm `top10_holder_pct` (%) và `holder_count` (số holders)
  - `analyze_holders()` (dòng 22-43): Hàm chính — gọi RPC với timeout, xử lý timeout gracefully (trả None thay vì crash)
  - `fetch_holder_concentration()` (dòng 46-83): Logic thực tế:
    1. Gọi `get_token_largest_accounts()` → lấy top 20 ví lớn nhất
    2. Gọi `get_token_supply()` → lấy tổng supply
    3. Tính: `top10_pct = (top10_sum / total_supply) * 100`
  - `check_holder_concentration()` (dòng 88-115): Wrapper cho orchestrator — so sánh kết quả với ngưỡng `max_top10_pct`, trả `Err(reason)` nếu vượt ngưỡng
  - **Unit test** (dòng 117-131): Test logic tính % concentration (32% > 30% → fail)

---

## MODULE 2: DYNAMIC PANIC-SELL VIA JITO BUNDLE

### Nhiệm vụ
Sau khi bot đã mua token, Module 2 **theo dõi liên tục** ví dev và whale. Nếu phát hiện họ bán >20% lượng token → bot tự động bán TRƯỚC họ thông qua **Jito Bundle** (giao dịch ưu tiên) để giảm thiểu tổn thất.

**Ví dụ:** Bot mua token ABC. 5 giây sau, dev bán 40% token → Module 2 phát hiện → gửi Jito bundle bán toàn bộ token ABC ngay lập tức.

### Các file code thuộc Module 2

#### 📄 `src/modules/anti_rug/panic_sell.rs` (399 dòng)
- **Nhiệm vụ:** Toàn bộ logic monitoring + Jito bundle submission cho panic sell.
- **Chi tiết:**
  - `JITO_TIP_ACCOUNTS` (dòng 51-60): 8 địa chỉ Jito tip accounts — chọn ngẫu nhiên 1 cái mỗi bundle
  - `JITO_BUNDLE_ENDPOINT` (dòng 62): URL Frankfurt Jito Block Engine (gần VPS nhất)
  - `PANIC_SELL_HANDLES` (dòng 35): **[Fix BUG-1]** Global DashMap lưu trữ monitor handles — giữ monitors sống
  - `store_panic_sell_handle()` (dòng 39-41): Lưu handle vào global map
  - `cancel_panic_sell_monitor()` (dòng 44-49): Cancel + xóa monitor khi token đã bán
  - `PanicSellMonitorHandle` (struct, dòng 64-79): Handle để cancel monitor từ bên ngoài, tự động cancel khi Drop
  - `PanicSellContext` (struct, dòng 81-92): Thông tin cần thiết: mint, keypair, balance, watched wallets
  - `start_panic_sell_monitor()` (dòng 98-107): Spawn task async riêng để monitoring
  - `run_monitor()` (dòng 109-187): Vòng lặp chính:
    1. Khởi tạo baseline balances cho từng wallet
    2. Poll mỗi 500ms: lấy balance hiện tại
    3. Nếu balance giảm >20%: trigger panic sell + gửi Telegram alert
  - `get_token_balance_for_wallet()` (dòng 190-199): Lấy token balance qua Associated Token Account (ATA)
  - `trigger_jito_panic_sell()` (dòng 202-273): Xây dựng và gửi Jito bundle:
    1. Build sell instructions (ATA + sell + close WSOL)
    2. Thêm Jito tip instruction
    3. Submit bundle qua REST API
    4. Fallback: nếu Jito fail → gửi bằng normal transaction
  - `submit_jito_bundle()` (dòng 285-321): HTTP POST đến Jito API với JSON-RPC payload
  - `log_panic_sell_alert()` (dòng 324-333): Log + gửi Telegram alert
  - **Unit tests** (dòng 335-397): 5 tests — cancel flag, auto-cancel on drop, Frankfurt endpoint, tip accounts validation, 20% threshold

---

## MODULE 3: DEV WALLET PROFILER

### Nhiệm vụ
Kiểm tra **lịch sử ví của developer** tạo token. Ví dev mới tạo (ít giao dịch, tuổi ngắn) có nguy cơ rug cao → FAIL. Ví dev có lịch sử dài → đáng tin hơn.

**Ví dụ:** Dev wallet có 3 TX, tuổi 2 giờ → FAIL (min cần 10 TX). Dev wallet có 50 TX, tuổi 90 ngày → PASS.

### Các file code thuộc Module 3

#### 📄 `src/modules/anti_rug/dev_wallet_profiler.rs` (179 dòng)
- **Nhiệm vụ:** Truy vấn lịch sử giao dịch của dev wallet qua RPC và đánh giá.
- **Chi tiết:**
  - `DevWalletProfile` (struct): Kết quả gồm `tx_count`, `oldest_tx_timestamp`, `estimated_age_hours`
  - `analyze_dev_wallet()` (dòng 29-83): Hàm chính:
    1. Gọi `get_signatures_for_address_with_config()` với limit 50 TX
    2. Áp dụng timeout protection
    3. TX cũ nhất = phần tử cuối mảng (RPC trả về mới→cũ)
    4. Tính tuổi: `age_hours = (now - oldest_tx_timestamp) / 3600`
  - `check_dev_wallet()` (dòng 91-116): Wrapper cho orchestrator:
    - Nếu `tx_count < min_tx_count` → trả `Err(reason)` → FAIL
    - Nếu đủ TX → trả `Ok(Some(tx_count))` → PASS
  - **Unit tests** (dòng 118-178): 4 tests — struct creation, empty wallet, TX threshold logic, age calculation

---

## MODULE 4: GENESIS BUNDLE DETECTOR

### Nhiệm vụ
Phát hiện kỹ thuật rug phổ biến: dev tạo **nhiều ví giả**, mua hàng loạt token trong block đầu tiên (genesis block) để chiếm phần lớn supply → dump sau khi giá lên.

**Ví dụ:** Block đầu tiên của token có 5 ví khác nhau mua tổng 60% supply → BUNDLE DETECTED → FAIL.

### Các file code thuộc Module 4

#### 📄 `src/modules/anti_rug/genesis_detector.rs` (235 dòng)
- **Nhiệm vụ:** Quét toàn bộ transactions trong genesis block để tìm bundled buy patterns.
- **Chi tiết:**
  - `GenesisAnalysis` (struct): Kết quả gồm `genesis_buy_pct`, `unique_buyers`, `bundle_detected`
  - `analyze_genesis_block()` (dòng 31-129): Hàm chính:
    1. Gọi `get_block_with_config()` lấy toàn bộ block data
    2. Quét từng transaction: tìm `post_token_balances` có mint trùng token
    3. Thu thập buyer wallet + amount vào HashMap
    4. Tính: `genesis_buy_pct = (total_bought / total_supply) * 100`
    5. Detect bundle: nếu `unique_buyers > max_clustered_wallets` VÀ `genesis_buy_pct > max_genesis_buy_pct` → `bundle_detected = true`
  - `check_genesis_bundles()` (dòng 136-169): Wrapper cho orchestrator — trả `Err(reason)` nếu bundle detected
  - **Unit tests** (dòng 171-234): 4 tests — struct creation, bundle detection logic, % calculation, safe genesis scenario

---

## MODULE 5: METADATA CHECKER

### Nhiệm vụ
Kiểm tra **Metaplex on-chain metadata** của token. Token hợp lệ thường có name, symbol, và URI (link website/social). Token không có metadata → đáng ngờ (WARN).

**Ví dụ:** Token có metadata URI `https://ipfs.io/abc` → PASS. Token không có metadata → WARN (cảnh báo nhưng không chặn mua).

### Các file code thuộc Module 5

#### 📄 `src/modules/anti_rug/metadata_checker.rs` (187 dòng)
- **Nhiệm vụ:** Derive Metaplex PDA, fetch account data, parse binary metadata.
- **Chi tiết:**
  - `METAPLEX_METADATA_PROGRAM` (dòng 17): Program ID của Metaplex Token Metadata
  - `MetadataCheckResult` (struct): Gồm `metadata_account_exists`, `has_uri`, `uri`, `name`
  - `derive_metadata_pda()` (dòng 31-43): Tính PDA address từ seeds = ["metadata", program_id, mint]
  - `check_metadata()` (dòng 46-65): Entry point với timeout protection
  - `fetch_metadata()` (dòng 67-97): Gọi RPC `get_account()` lấy raw account data
  - `parse_metadata_uri()` (dòng 101-137): Parse binary Metaplex data thủ công:
    - Skip: 1 byte (key) + 32 bytes (update_authority) + 32 bytes (mint) = offset 65
    - Parse: name (4 + N bytes) → symbol (4 + N bytes) → URI (4 + N bytes)
    - Trim null bytes và whitespace
  - `read_length_prefixed_string()` (dòng 139-153): Helper đọc chuỗi length-prefixed từ binary data
  - **Unit tests** (dòng 167-186): 2 tests — parse empty data, PDA derivation

---

## HỆ THỐNG HỖ TRỢ 1: DATABASE LOGGING

### Nhiệm vụ
Lưu **toàn bộ kết quả filter** vào PostgreSQL để phân tích hiệu suất, tinh chỉnh ngưỡng, và audit trail.

### Các file code

#### 📄 `src/modules/postgresql/entities/anti_rug_filter_log.rs` (36 dòng)
- **Nhiệm vụ:** Định nghĩa **SeaORM entity** — mapping giữa Rust struct và bảng PostgreSQL `anti_rug_filter_log`.
- **Chi tiết:** Mỗi record gồm: `id`, `token_mint`, `created_at`, `verdict`, `reject_reason`, `top10_holder_pct`, `dev_tx_count`, `genesis_buy_pct`, `genesis_bundle_detected`, `has_metadata_uri`, `filter_duration_ms`

#### 📄 `src/modules/postgresql/migration/m20260430_000003_create_anti_rug_log_table.rs` (129 dòng)
- **Nhiệm vụ:** **Database migration** — tự động tạo bảng `anti_rug_filter_log` khi bot chạy lần đầu.
- **Chi tiết:**
  - Tạo bảng với 10 cột (data types: string, timestamp, double, boolean, bigint)
  - Tạo 2 indexes: `idx-anti-rug-log-token-mint` (tìm nhanh theo token) và `idx-anti-rug-log-created-at` (query theo thời gian)
  - Có rollback function `down()` để xóa bảng khi cần

#### 📄 `src/modules/postgresql/db.rs` — phần thêm mới (50 dòng)
- **Nhiệm vụ:** Cung cấp hàm `log_anti_rug_filter_result()` và **shared DB connection pool** (Fix BUG-2).
- **Chi tiết:**
  - `SHARED_DB_POOL` (static OnceCell): Connection pool khởi tạo 1 lần, tái sử dụng
  - `get_shared_db()`: Trả về reference đến shared connection — tránh mở connection mới mỗi lần log
  - `log_anti_rug_filter_result()`: Insert 1 record vào bảng `anti_rug_filter_log` với tất cả dữ liệu filter

---

## HỆ THỐNG HỖ TRỢ 2: TELEGRAM ALERT

### Nhiệm vụ
Gửi **thông báo real-time** qua Telegram khi phát hiện token nguy hiểm hoặc khi panic sell được kích hoạt.

### Các file code

#### 📄 `src/modules/telegram_ui/alert_sender.rs` (107 dòng)
- **Nhiệm vụ:** Global Telegram alert system — gửi message fire-and-forget từ bất kỳ module nào.
- **Chi tiết:**
  - `ALERT_BOT` (static OnceCell): Global bot instance, khởi tạo 1 lần
  - `init_alert_bot()` (dòng 23-49): Đọc `TELEGRAM_BOT_TOKEN` + `ALLOWED_TELEGRAM_USER_ID` từ .env → tạo bot
  - `send_telegram_alert()` (dòng 53-68): Gửi message async (tokio::spawn), không block luồng chính
  - `alert_token_filtered()` (dòng 71-81): Format message khi token bị skip: "🛡️ Anti-Rug Alert — Token SKIPPED"
  - `alert_panic_sell_triggered()` (dòng 84-94): Format message khi panic sell: "🚨 PANIC SELL TRIGGERED"
  - `alert_buy_success()` (dòng 97-106): Format message khi mua thành công

---

## HỆ THỐNG HỖ TRỢ 3: TRADE INTEGRATION

### Nhiệm vụ
Tích hợp toàn bộ Anti-Rug layer vào **luồng mua/bán chính** của bot mà KHÔNG thay đổi logic gốc.

### Các file code

#### 📄 `src/features/handle_sniper/execute_trade.rs` — phần thêm mới (80 dòng)
- **Nhiệm vụ:** Chèn Anti-Rug filter VÀO GIỮA luồng trade: detect migration → **[FILTER]** → buy/skip.
- **Chi tiết:**
  - Bước 1 (dòng 21-45): Thu thập tokens từ DashMap vào 2 Vec (buy + sell)
  - Bước 2 (dòng 48-67): Xử lý SELL — cancel panic-sell monitor trước khi bán **(Fix BUG-1)**
  - Bước 3 (dòng 69-196): Xử lý BUY:
    1. Gọi `evaluate_token()` → chạy M1+M3+M4+M5 song song
    2. Log kết quả ra console + PostgreSQL (dùng shared pool — **Fix BUG-2**)
    3. Nếu FAIL + không phải warn_only → skip, gửi Telegram alert, đánh dấu `RugDetected`
    4. Nếu PASS → thực hiện buy (logic gốc không đổi)
    5. Sau khi buy → start M2 panic-sell monitor, lưu handle vào global map **(Fix BUG-1)**

#### 📄 `src/modules/telegram_ui/run_state.rs` — phần thêm mới (3 dòng)
- **Nhiệm vụ:** Thêm field `anti_rug: AntiRugConfig` vào `TelegramBotRunState` để bot có thể đọc cấu hình anti-rug.

#### 📄 `src/modules/token_db/token_db_schema.rs` — phần thêm mới (2 dòng)
- **Nhiệm vụ:** Thêm variant `RugDetected` vào enum `SniperTradeStatus` để đánh dấu token bị filter block.

#### 📄 `src/entry_point/migration_sniper_mode.rs` — phần thêm mới (3 dòng)
- **Nhiệm vụ:** Gọi `init_alert_bot()` khi bot khởi động để kích hoạt Telegram alerts.

---

## TỔNG KẾT SỐ LIỆU

| Hạng mục | Số lượng |
|----------|---------|
| Files mới tạo | 11 |
| Files sửa đổi (chỉ thêm) | 6 |
| Tổng dòng code thêm mới | ~1,547 |
| Dòng code gốc bị xóa | 0 |
| Unit tests | 16 |
| Bug fixes | 2 (BUG-1: handle drop, BUG-2: DB pool) |

| Module | File chính | Dòng code | Tests |
|--------|-----------|----------|-------|
| Core | `mod.rs`, `config.rs`, `filter_result.rs`, `pre_buy_filter.rs` | 354 | 0 |
| M1 - Holder | `holder_analyzer.rs` | 132 | 1 |
| M2 - Panic Sell | `panic_sell.rs` | 399 | 5 |
| M3 - Dev Profiler | `dev_wallet_profiler.rs` | 179 | 4 |
| M4 - Genesis | `genesis_detector.rs` | 235 | 4 |
| M5 - Metadata | `metadata_checker.rs` | 187 | 2 |
| DB Logging | `anti_rug_filter_log.rs`, migration, `db.rs` | 215 | 0 |
| Telegram Alert | `alert_sender.rs` | 107 | 0 |
| Trade Integration | `execute_trade.rs` (phần mới) | ~80 | 0 |
