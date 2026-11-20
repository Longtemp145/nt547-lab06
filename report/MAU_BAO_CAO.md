# NT547 - LAB 06 - BÁO CÁO (TỰ ĐIỀN)

**MSSV:** 24521005  | **Họ tên:** Nguyễn Hoàng Long | **Lớp:** NT547.R11.ANTT
**Session code tự sinh:** SC-015C17 | **Tên contract:** StakingPool_1005
**Solidity / Slither:** 0.8.24 / 0.11.6 | **Ngày hoàn thành:** 20/11/2026
**URL GitHub repository:** https://github.com/Longtemp145/nt547-lab06

> `lab_identity.json`: baseline_sha256 = `d1883023e98a0d63b293c8fce1d3df499befdd9ec2443c7237c2a79b831b901f`, generated_utc = 2026-11-20T01:33:20Z.
> Môi trường: WSL2 Ubuntu, Python 3.12.7 (venv), solc 0.8.24 (solc-select), Graphviz 2.43.0; `tools/preflight.py` báo OK toàn bộ.

## 1. Static scan (2,0 điểm)

- Lệnh raw: `python tools/scan.py --mode raw --tag before`
  (thực chất: `slither AuditTarget.sol --config-file config/slither.raw.json --fail-none --json evidence/before_raw/slither.json`, process_exit_code = 0)
- Số lượng finding raw từ `evidence/before_raw/summary.json` (tổng **11**):
  - High: **2**; Medium: **3**; Low: **1**; Informational: **5**; Optimization: **0**
- Chạy configured: `python tools/scan.py --mode configured --tag before` → tổng **5** (High 2, Medium 3, Low 0, Informational 0).
- So sánh raw/configured: `slither.config.json` chỉ bật `exclude_low`, `exclude_informational` và loại detector `solc-version`, nên 6 finding bị ẩn đi đều là Low/Informational (1 `timestamp`, 4 `low-level-calls`, 1 `naming-convention`). **Toàn bộ 2 High + 3 Medium vẫn còn nguyên** ở chế độ configured, tức là config chỉ giảm nhiễu chứ không "che" lỗi nghiêm trọng; muốn CI GREEN bắt buộc phải sửa code hoặc suppress có lý do ở đúng dòng.
- 3 detector **khác tên** (chọn để phân tích chi tiết): `reentrancy-eth`, `tx-origin`, `unchecked-lowlevel`.

| ID finding | Detector | Impact | Confidence | Source (file:dòng) | Luồng dữ liệu: đầu vào -> thao tác -> tác hại |
|---|---|---|---|---|---|
| F01 | reentrancy-eth | High | Medium | AuditTarget.sol:36-42 (`pullFunds`; call ở dòng 39, ghi `shares` ở dòng 41) | `msg.sender` (có thể là contract) gọi `pullFunds()` → `amount = shares[msg.sender]` → `msg.sender.call{value: amount}` chuyển ETH và trao quyền điều khiển cho `receive()` của kẻ gọi **trước khi** `shares[msg.sender] = 0` → `receive()` gọi lại `pullFunds()`, `shares` vẫn còn nguyên nên rút lặp nhiều lần → rút cạn ETH của người gửi khác. |
| F02 | tx-origin | Medium | Medium | AuditTarget.sol:57-61 (`updateOwner`; kiểm tra ở dòng 58) | Owner bị dụ gọi một contract độc hại bất kỳ → contract đó gọi `updateOwner(attacker)` → trong lời gọi này `tx.origin == owner` (EOA ký giao dịch) dù `msg.sender` là contract độc hại → `owner = attacker` → kẻ tấn công chiếm quyền `flush`/`ownerWithdraw`/`vestStep`, rút toàn bộ quỹ. |
| F03 | unchecked-lowlevel | Medium | Medium | AuditTarget.sol:44-47 (`flush`; call ở dòng 46) | Owner gọi `flush(to)` → `to.call{value: balance}("")` trả về `(false, …)` khi `to` revert/hết gas → kết quả bị bỏ qua, giao dịch vẫn "thành công" → owner tưởng đã chuyển tiền nhưng ETH vẫn nằm trong contract, log/off-chain kế toán sai, không có tín hiệu lỗi để xử lý. |

## 2. Triage và sửa lỗi (2,5 điểm)

Đã đọc toàn bộ AuditTarget.sol (baseline): **4 mẫu lỗi thật** = reentrancy (`pullFunds`), selfdestruct không phân quyền (`shutdown`), `tx.origin` authorization (`updateOwner`), unchecked low-level call (`flush`); **2 đoạn cần đánh giá ngữ cảnh** = so sánh bằng tuyệt đối `address(this).balance == 0` (`isEmpty`) và `block.timestamp` cho timelock (`vestStep`). Cả 6 mẫu đều có detector báo, nên không có mục "Manual review / Not reported".

| Detector + vị trí | TP / FP / Chưa kết luận | Bằng chứng kiểm soát/khả năng khai thác | Biện pháp (Fix / Accept có lý do) | Sau sửa |
|---|---|---|---|---|
| `reentrancy-eth` – `pullFunds` L36-42 | **TP** | Hàm `external` không có modifier, ai cũng gọi được; call ra `msg.sender` (do kẻ tấn công kiểm soát) xảy ra trước khi ghi `shares = 0` (L41); `shares` còn được dùng chéo trong `save()` → khai thác được (xem kịch bản bên dưới). | **Fix**: áp dụng Checks-Effects-Interactions, chuyển `shares[msg.sender] = 0;` lên trước `call`, giữ `require(ok)`. | Hết finding; chỉ còn Info `low-level-calls` (chấp nhận). |
| `suicidal` – `shutdown` L53-55 | **TP** | `selfdestruct(payable(msg.sender))` không có kiểm soát truy cập: bất kỳ ai gọi cũng nhận toàn bộ ETH của pool (theo EIP-6780, sau Cancun selfdestruct vẫn chuyển hết ETH dù không xoá code). | **Fix**: xoá hẳn hàm `shutdown()`; nghiệp vụ không cần hủy contract. | Hết finding. |
| `tx-origin` – `updateOwner` L57-61 | **TP** | Kiểm tra quyền bằng `tx.origin` → bị phishing qua contract trung gian như F02; hậu quả là mất quyền owner. | **Fix**: `require(msg.sender == owner, "not owner")` + chặn `address(0)` + `emit OwnerUpdated(previousOwner, newOwner)` để có dấu vết. | Hết finding. |
| `unchecked-lowlevel` – `flush` L44-47 | **TP** | Giá trị trả về của `to.call` bị bỏ qua; dù chỉ owner gọi được, lỗi chuyển tiền trở nên "im lặng". | **Fix**: `(bool ok, ) = to.call{value: address(this).balance}(""); require(ok, "flush failed");` | Hết finding; còn Info `low-level-calls` (chấp nhận). |
| `incorrect-equality` – `isEmpty` L49-51 | **FP** (theo ngữ cảnh) | Hàm `view`, chỉ trả về thông tin, **không** được dùng làm điều kiện cho bất kỳ thao tác chuyển tiền/ghi state nào. Kẻ tấn công có thể ép ETH vào (selfdestruct/coinbase) để `isEmpty()` trả `false`, nhưng không có hàm nào phụ thuộc kết quả đó → không có tác hại. | **Accept**: `// slither-disable-next-line incorrect-equality` đặt ngay trên dòng `return address(this).balance == 0;`, chỉ tắt đúng detector đó ở đúng 1 dòng. | Không còn hiển thị; lý do ghi ở đây. |
| `timestamp` – `vestStep` L63-68 | **Chưa kết luận → Accept** (risk thấp) | `onlyOwner`; `block.timestamp` dùng cho cooldown **1 ngày**. Validator chỉ lệch được vài giây, không đủ để vượt cooldown 86 400 s; mỗi lần chỉ trả 10% số dư cho chính owner. | **Accept** (không sửa): độ chính xác tính bằng giây là đủ cho timelock theo ngày. Finding mức Low bị `exclude_low` ẩn ở CI nhưng vẫn có trong `after_raw`. | Vẫn còn trong raw (Low), có lý do. |

- **Kịch bản khai thác TP (F01 – reentrancy):** Kẻ tấn công deploy contract `Attacker` có `receive()` gọi lại `pool.pullFunds()` khi `address(pool).balance >= 1 ether` → `Attacker` gọi `save{value: 1 ether}()` (`shares[Attacker] = 1 ether`) → gọi `pullFunds()`: `amount = 1 ether`, `call` gửi 1 ETH cho `Attacker` → `receive()` chạy và gọi lại `pullFunds()`; lúc này `shares[Attacker]` **vẫn là 1 ether** vì dòng 41 chưa chạy → tiếp tục nhận 1 ETH… lặp đến khi pool cạn → **hậu quả:** toàn bộ ETH của người gửi khác bị rút, sau đó các `require(ok)` lồng nhau đều thành công nên giao dịch không revert. Sau khi sửa (CEI), lần gọi lại thứ hai gặp `shares == 0` → `require(amount > 0, "empty")` revert.
- **Finding có severity cao nhưng risk thực tế không tương ứng:** `incorrect-equality` được Slither xếp **Medium/High confidence** (nằm trong cổng fail-on medium) nhưng risk thực tế ≈ 0 vì `isEmpty()` chỉ là hàm `view` thông tin, không điều kiện hoá logic nào. Ngược lại, `timestamp` chỉ là **Low** và đúng là risk thấp vì cooldown tính theo ngày. Điều này cho thấy Impact của detector là theo *mẫu code*, còn risk phải đánh giá theo *ngữ cảnh sử dụng*.
- **Bằng chứng `git diff`, scan before/after:** `evidence/fix.diff` (commit `15a4c0b`) cho thấy đúng các thay đổi trên. Scan after: raw **6** (High 0, Medium 0, Low 1, Info 5) – `evidence/after_raw/summary.json`; configured **0** – `evidence/after_configured/summary.json`. Các cảnh báo còn lại (đều ngoài cổng CI):
  - `timestamp` (Low) ở `vestStep`: Accept như bảng trên.
  - `low-level-calls` (Info) ×4 ở `ownerWithdraw`, `pullFunds`, `flush`, `vestStep`: dùng `call` là cách khuyến nghị để chuyển ETH (không giới hạn 2300 gas như `transfer`); cả 4 chỗ đều đã kiểm tra `ok` và chạy sau khi cập nhật state.
  - `naming-convention` (Info): tên `StakingPool_1005` có hậu tố MSSV do bộ đề sinh ra, không phải CapWords; không ảnh hưởng bảo mật, giữ nguyên để đúng định danh bài lab.
- **Lý do của mọi `slither-disable-next-line`:** chỉ có **1** chỗ: `// slither-disable-next-line incorrect-equality` ở `isEmpty()` (ngay trên dòng `return`). Ghi đúng tên detector, không tắt toàn file/toàn detector trong config. Kiểm soát bù trừ: hàm chỉ đọc, không có code nào dùng kết quả để quyết định chuyển tiền; nếu sau này có logic dựa trên số dư thì phải dùng biến kế toán nội bộ (ví dụ tổng `shares`) thay vì `address(this).balance`.

## 3. Kiến trúc / Inheritance (1,5 điểm)

- Lệnh printer đã chạy: `python tools/print_arch.py`, script gọi
  `slither contracts/ArchitectureDemo.sol --print inheritance-graph --config-file config/slither.raw.json --fail-none` và
  `slither contracts/ArchitectureDemo.sol --print human-summary --config-file config/slither.raw.json --fail-none` (`evidence/architecture/printer_log.txt`, `human_summary.txt`).
  Slither 0.11.6 ghi file `contracts/ArchitectureDemo.sol.inheritance-graph.dot` nên script báo "Khong tim thay *.dot"; em render thủ công:
  `cp contracts/ArchitectureDemo.sol.inheritance-graph.dot evidence/architecture/` và `dot -Tpng evidence/architecture/ArchitectureDemo.sol.inheritance-graph.dot -o evidence/architecture/inheritance.png`.
- Ảnh `evidence/architecture/inheritance.png` (bản có chú thích: `evidence/architecture/inheritance_marked.png`): 3 contract, `StudentAsset` kế thừa (1) `AccessManaged` và (2) `BalanceLedger`. `human_summary.txt`: 3 contracts, SLOC 23, 0 High/Medium/Low, 1 optimization; `StudentAsset` có 4 functions.
- Access control nằm ở **`AccessManaged`** (biến `admin`, `modifier onlyAdmin()` kiểm tra `msg.sender == admin`); logic số dư nằm ở **`BalanceLedger`** (`mapping balances` internal, `_credit()` internal ghi số dư, `balanceOf()` chỉ đọc); contract triển khai nằm ở **`StudentAsset`** (entrypoint `mint(address,uint256)` có `onlyAdmin`, gọi `_credit()`).
- **Điểm audit cần bắt đầu:** Bắt đầu từ `StudentAsset` vì đó là contract được deploy và là nơi duy nhất có hàm `external` thay đổi state (`mint`). Từ `mint` lần theo hai nhánh: (1) `onlyAdmin` trong `AccessManaged`, vì mọi bảo vệ quyền đều phụ thuộc modifier này và `admin` chỉ được gán trong constructor, không có cơ chế chuyển/thu hồi quyền; (2) `_credit` trong `BalanceLedger`, nơi trực tiếp thay đổi `balances`. Cần xác nhận `_credit` là `internal` nên không thể gọi từ ngoài, và không có hàm nào khác ghi vào `balances`. Thứ tự kế thừa (C3 linearization) cũng cần kiểm tra để chắc không có hàm/modifier bị override ngoài ý muốn.

## 4. Red -> Green CI (2,5 điểm)

Workflow `.github/workflows/slither.yml`: `crytic/slither-action@v0.4.2`, `solc-version: 0.8.24`, `slither-version: 0.11.6`, `slither-config: slither.config.json`, `fail-on: medium`.

| Phase | Commit (full/short SHA) | Run URL | Kết quả | Detector khiến RED / lý do GREEN |
|---|---|---|---|---|
| Baseline vulnerable | `170fc198099b1cf52eae0a06009ee0400c55be39` / `170fc19` | https://github.com/Longtemp145/nt547-lab06/actions/runs/44658794841 | RED | `reentrancy-eth`, `suicidal` (High); `incorrect-equality`, `tx-origin`, `unchecked-lowlevel` (Medium) → "5 result(s) found" ≥ medium nên step Slither fail. |
| Patched | `15a4c0b674c2615fe8b4ee438a0c1125037364f4` / `15a4c0b` | https://github.com/Longtemp145/nt547-lab06/actions/runs/44660590351 | GREEN | 4 TP đã sửa, 1 FP suppress đúng dòng; Low/Info bị `exclude_low/informational` loại → "0 result(s) found". |

- **So sánh local `--fail-none` với CI `fail-on: medium`:** Ở local, `scan.py` chạy `--fail-none` để Slither **luôn trả exit 0** và ghi đủ JSON/CSV làm bằng chứng, kể cả khi có High (summary ghi `process_exit_code: 0`). Trên CI, `fail-on: medium` biến kết quả thành **cổng chặn**: chỉ cần ≥1 finding Medium/High là job đỏ. Vì vậy cùng một baseline cho "11 findings, exit 0" ở local nhưng RED trên GitHub.
- **`exclude_low/informational` thay đổi kết quả hiển thị:** trên baseline giảm từ 11 xuống 5; sau khi sửa giảm từ 6 xuống 0. Hai cờ này chỉ ẩn Low/Info khỏi output/cổng CI; chúng **không** ảnh hưởng High/Medium (bài không dùng `exclude_medium`/`exclude_high`). Do đó các finding Low/Info vẫn phải được triage ở bản raw (mục 2).

> Các run phụ trên Actions (không thuộc chuỗi chấm, đều GREEN): #3 `5c9c23a` architecture evidence (runs/44662193923); #4 `0d7689c` mutation trong modifier (runs/44664136136, xem mục 5); #5 `c280da8` revert mutation modifier (runs/44664589162).

## 5. Mutation RED -> GREEN (1,5 điểm)

- Mutation một dòng: `require(msg.sender == owner, "not owner")` -> `require(tx.origin == owner, "not owner")` trong `modifier onlyOwner`.
- Detector kỳ vọng: `tx-origin` (mức Medium); xác nhận của em từ CI:
  - **Theo đúng đề (đổi trong `modifier onlyOwner`, dòng 19):** local `python tools/scan.py --mode configured --tag mutation_modifier` → **issues = 0** (`evidence/mutation_modifier_configured/`); commit `0d7689c` → run https://github.com/Longtemp145/nt547-lab06/actions/runs/44664136136 vẫn **GREEN**. Đây là **false negative** của Slither 0.11.6: detector `tx-origin` chỉ duyệt `contract.functions` (các hàm), không duyệt `contract.modifiers`, nên `tx.origin` trong modifier không bị báo, dù lỗ hổng là thật (mọi hàm `onlyOwner` đều bị phishing). Em đã revert bằng `c280da8` (runs/44664589162).
  - **Mutation tương đương để kiểm chứng cổng CI:** đổi đúng 1 dòng kiểm tra quyền trong `updateOwner` (dòng 57), cùng bản chất `msg.sender` → `tx.origin`. Local `--tag mutation_check` → issues = 1 (`tx-origin`, Medium). CI log: `Detector: tx-origin`, `StakingPool_1005.updateOwner(address) (AuditTarget.sol#56-61) uses tx.origin for authorization … (AuditTarget.sol#57)`, "1 result(s) found" → **RED**. Revert → "0 result(s) found" → **GREEN**.

| Phase | Commit SHA | Run URL | Kết quả |
|---|---|---|---|
| Mutation | `ef55d34b0d8899c7310c61195dcee7016873a62b` | https://github.com/Longtemp145/nt547-lab06/actions/runs/44664931697 | RED |
| Revert mutation | `e1b2157b30517be37c4dbf1ee2f7735a8aaf3d9d` | https://github.com/Longtemp145/nt547-lab06/actions/runs/44665216157 | GREEN |

- **Giải thích sự thay đổi về nguồn định danh/authorization:** `msg.sender` là **địa chỉ gọi trực tiếp** hàm hiện tại (EOA hoặc contract trung gian), còn `tx.origin` là **EOA đã ký giao dịch gốc**, giữ nguyên xuyên suốt chuỗi call. Khi authorization chuyển từ `msg.sender` sang `tx.origin`, contract không còn kiểm tra "ai đang gọi mình" mà chỉ kiểm tra "ai bắt đầu giao dịch": nếu owner tương tác với một contract độc hại (airdrop, NFT mint giả…), contract đó có thể gọi tiếp `updateOwner(attacker)` và vượt qua kiểm tra vì `tx.origin == owner`. Kết quả là quyền owner bị chiếm mà owner không hề gọi trực tiếp `updateOwner`. Ngoài ra, `tx.origin` luôn là EOA nên còn làm hỏng tính tương thích với ví multisig / smart-contract wallet.

## 6. Bằng chứng, tự kiểm

- [x] `AuditTarget.sol`, `lab_identity.json`
- [x] `evidence/before_raw/` và `evidence/before_configured/`
- [x] `evidence/after_raw/` và `evidence/after_configured/` (+ `evidence/fix.diff`)
- [x] `evidence/architecture/` (`human_summary.txt`, `printer_log.txt`, `.dot`, `inheritance.png`, `inheritance_marked.png`)
- [x] 04 URL Actions run RED/GREEN/RED/GREEN + 04 commit SHA (mục 4 và 5)
- [x] Link repo và ảnh màn hình có terminal / URL / commit SHA; không cắt mất ngữ cảnh (file ảnh minh chứng kèm theo)
- [x] Các finding được triage, không chép lại mô tả detector thay cho suy luận
- [x] Tuyệt đối không có seed phrase, private key, credentials

Bằng chứng mutation bổ sung: `evidence/mutation_modifier_configured/` (0 finding, false negative) và `evidence/mutation_check_configured/` (1 finding `tx-origin`).

**Nhận xét cuối cùng (5–7 câu):** Slither giúp phát hiện nhanh 5 lỗi Medium/High trên baseline, và cổng `fail-on: medium` trong GitHub Actions biến kết quả đó thành một điều kiện chặn rõ ràng (RED → GREEN). Tuy vậy, output của tool không thay thế được triage: `incorrect-equality` là Medium nhưng thực tế vô hại, còn `timestamp` là Low và được chấp nhận có lý do. Bài mutation cho thấy Slither cũng có **false negative**: `tx.origin` trong modifier không bị phát hiện nên CI vẫn GREEN dù code đã có lỗ hổng. Vì vậy CI xanh chỉ có nghĩa "không có mẫu lỗi mà detector biết", không có nghĩa là contract an toàn. Cần kết hợp đọc code thủ công, unit test cho phân quyền (ví dụ test gọi qua contract trung gian) và có thể thêm công cụ khác như fuzzing. Suppression nên giới hạn ở đúng detector, đúng dòng và luôn có ghi chú lý do để người review sau kiểm chứng được.
