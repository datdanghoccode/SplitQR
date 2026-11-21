# DATA MODEL
- bills: id, code (unique 8-char nanoid, used in share link /b/:code), title, tax, service_fee, discount, fee_mode ('proportional' | 'equal'), version, created_at
- members: id, bill_id, name, bank_bin, account_no, account_name (bank info optional)
- items: id, bill_id, name, unit_price, qty
- item_shares: item_id, member_id, weight (default 1; weight 2 = eats double)
- payers: bill_id, member_id, amount_paid (supports multiple payers)
- transfers: id, bill_id, from_member, to_member, amount, status ('pending' | 'confirmed'), created_at
- Indexes on every bill_id. Row Level Security: access only through the bill code via RPC; anonymous users can read/write only bills they joined.

# CORE FEATURES (each needs acceptance criteria met)
1. Create bill in under 3 taps: title, add members by name (quick-add chips, paste a list of names to add many at once).
2. Add items fast: name + price + qty. Price input accepts "50k" = 50,000 and "1tr2" = 1,200,000, displayed as 1.200.000đ.
3. Split modes per item: equal among selected members, by weight/shares, by exact amount, by percentage. Default = equal among everyone. Tap member avatars on an item to include or exclude them.
4. Extras: tax/VAT %, service fee, tip, discount (amount or %). Allocate proportionally to each person's subtotal or equally (user's choice).
5. Rounding: use the largest-remainder method so that the sum of all shares equals the bill total EXACTLY. Optional "round each person up to nearest 1.000đ" toggle, with the surplus assigned to the payer.
6. Multiple payers: record who paid how much. Compute each member's net balance = paid - owed.
7. Settlement: minimize number of transfers (greedy match: largest creditor with largest debtor, repeat). Show "A chuyển cho B: X đ".
8. VietQR per transfer: generate the VietQR (EMVCo, NAPAS 247) payload CLIENT-SIDE with CRC16-CCITT and render it with a tiny QR lib, so there is no network round trip. Include amount and transfer note (e.g. "Chia bill <title> - <name>"). Bundle a static JSON of Vietnamese banks (name, short name, BIN). Fallback to img.vietqr.io only if local generation fails. Validate the QR against at least 3 real banking apps.
9. Payment tracking: debtor taps "Đã chuyển" -> status pending; creditor taps "Xác nhận" -> confirmed. Remaining settlement is always recomputed as (net balance - confirmed/pending transfers), so editing the bill later never corrupts history.
10. Share: link /b/:code, native Web Share API, copy button, and QR of the link. Opening the link asks "Bạn là ai?" (pick an existing member or add yourself) and remembers the choice locally.
11. Realtime: all participants see edits and payment status live (Supabase Realtime channel per bill). Conflict handling: last-write-wins per row, bill.version for optimistic checks.
12. Local history: list of bills the device has opened, stored in IndexedDB, works offline. Remember my own bank info locally (never sent unless I choose to attach it to a bill).
13. Edit/delete items and members safely (warn if someone already has confirmed transfers).

14. # LATENCY REQUIREMENTS (treat as hard budgets, verify and report numbers)
- LCP < 1.5s on throttled 4G, Lighthouse Performance >= 95 on mobile.
- Initial JS < 150KB gzip. Route-level code splitting. Lazy-load the QR and share modules.
- All local interactions respond < 100ms (INP < 100ms). Split and settlement recalculation < 10ms for 50 members x 100 items (pure functions, memoized, run in the main thread unless it exceeds budget, then a Web Worker).
- Opening a known bill renders instantly from IndexedDB (stale-while-revalidate), then syncs in the background.
- One round trip to load a bill: use a single Postgres RPC `get_bill_bundle(code)` that returns bill + members + items + shares + payers + transfers as JSON.
- Writes: optimistic update immediately, batched/debounced network sync (<= 1 request per 300ms), rollback with a small toast on failure.
- Realtime edit propagation to other devices < 500ms.
- Preconnect to the Supabase host, system font stack (no web font blocking), inline critical CSS, long-cache hashed assets, Brotli.
- No layout shift: skeletons with fixed dimensions.
