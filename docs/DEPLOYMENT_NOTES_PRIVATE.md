# 🔐 GHI NHỚ QUAN TRỌNG — TRIỂN KHAI DỰ ÁN RIÊNG

**Ngày ghi:** 2026-05-02  
**Mục đích:** Khi triển khai bot cho bản thân, cần thay đổi các thông tin sau

---

## 1. VÍ ẨN CỦA DEV CŨ (CẦN XÓA/THAY)

**File:** `src/features/confirm_tx/send_zero_slot_tx.rs` — dòng 41
```rust
let tip_receiver = Pubkey::from_str("TpdxgNJBWZRL8UXF5mrEsyWxDWx9HQexA9P1eTWQ42p").unwrap();
```
- Mỗi giao dịch chuyển **0.001 SOL** vào ví này
- **Hành động:** Xóa hoặc thay bằng ví của mình

## 2. API KEY 0-SLOT (CỦA DEV CŨ)

**File:** `src/features/confirm_tx/send_zero_slot_tx.rs` — dòng 78
```rust
.post("http://de2.0slot.trade?api-key=335e371309b6492584368e9dc553622d")
```
- Dịch vụ gửi TX nhanh của bên thứ 3
- **Hành động:** Đăng ký 0-slot riêng hoặc dùng Jito thay thế

## 3. PHÍ MẶC ĐỊNH

**File:** `src/config/standard_config.rs` — dòng 19
```rust
default_third_party_fee: 0.001, // 0.001 SOL mỗi giao dịch
```
- **Hành động:** Giảm về 0 nếu không dùng dịch vụ 0-slot

## 4. THÔNG TIN .ENV CẦN THAY

| Mục | Hiện tại | Cần thay |
|-----|---------|---------|
| `RPC_ENDPOINT` | Shyft key `Bs3GIL0Q3NdkzFNc` (của khách) | Đăng ký Shyft/Helius riêng |
| `GRPC_TOKEN` | `b3df13df-...` (của khách) | Đăng ký gRPC riêng |
| `TELEGRAM_BOT_TOKEN` | `8666425064:AAG...` (của mình ✅) | Giữ nguyên |
| `ALLOWED_TELEGRAM_USER_ID` | `8281428035` (của mình ✅) | Giữ nguyên |

## 5. CHECKLIST TRIỂN KHAI

- [ ] Thuê VPS riêng (Frankfurt, ~$10-20/tháng)
- [ ] Đăng ký RPC/gRPC endpoint riêng
- [ ] Xóa/thay ví ẩn `Tpdxg...42p` trong `send_zero_slot_tx.rs`
- [ ] Đăng ký 0-slot API key riêng hoặc xóa
- [ ] Set `default_third_party_fee: 0.0` nếu không dùng 0-slot
- [ ] Tạo wallet mới + nạp SOL
- [ ] Build + deploy + test với 0.01 SOL
