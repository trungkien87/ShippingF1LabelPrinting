# Spec Review – Shipping F1 Packing Label Printing

| Mục | Nội dung |
|---|---|
| Tài liệu được review | `System_Development_Requirement_20261116.xlsx` (6 sheet + ảnh nhúng) |
| Ngày review | 2026-11-16 |
| Phiên bản | Rev01 |
| Phạm vi | Khai báo (Admin/DE/PMC) → In tem → Kiểm tra sau in → Đối chiếu chéo sau dán → Lưu lịch sử |

Ký hiệu dùng trong tài liệu:

- **[Dữ kiện]**: đọc trực tiếp từ spec, ảnh nhúng hoặc ảnh chụp màn hình trong file.
- **[Suy luận]**: rút ra từ dữ kiện, chưa ai xác nhận.
- **[Đề xuất]**: khuyến nghị của người review.

---

## 1. Kết luận chính

1. **Spec mô tả kỹ luồng nghiệp vụ nhưng chưa xử lý nguyên nhân gốc của lỗi trùng Reel ID.** Spec chỉ thêm bước *phát hiện* trùng (validate trước khi in, scan sau khi in). Bản thân **công thức ID hiện tại vẫn có thể sinh ra trùng ngay cả khi người nhập đúng hết**. Xem §3.
2. **Có 5 điểm blocker** cần chốt trước khi thiết kế: Q-01 đến Q-05 trong §6.
3. **Mô hình dữ liệu trong Ref 01/03 có lỗi**: Outer QTY và Inner QTY cùng lấy `[Input-4]`, và chưa mô hình hóa trường hợp một thùng chứa nhiều lot. Xem §4.2.
4. **Chưa rõ quan hệ với hệ thống hiện có** (PlusA, SaoMai, eMap). SaoMai đang lưu sẵn liên kết Invoice ↔ Outer ↔ Inner ↔ AlBag. Spec mới làm lại phần này mà không nói sẽ thay thế hay tích hợp. Xem §4.4.
5. **Yêu cầu WYSIWYG designer tự xây** là hạng mục tốn chi phí và rủi ro nhất, trong khi phần này không giúp giải quyết lỗi trùng ID. Cần cân nhắc build hay buy. Xem §5.

---

## 2. Công thức nhãn giải mã từ ảnh nhúng [Dữ kiện]

Các ô `#VALUE!` ở cột *Data Rules* của sheet Ref 02 và Ref 03 thực chất là ảnh trong cell. Nội dung các ảnh đó:

### 2.1 Inner label (túi nhôm)

| Field | Công thức | Ví dụ |
|---|---|---|
| LOT NO (8 ký tự) | Vendor code (1) + Year (1) + Month (1) + Lot No. (5) | `G6I00043` |
| ID (28 ký tự) | ICT P/N (16) + D/C `yymmdd` (6) + **Lot No. 3 digits** (3) + **Al bag number** (3, bắt đầu từ 001) | `821-05582-MP-QSF` `260925` `043` `004` |
| SEEV CODE CONTROL (Code39) | `*IN-` + Vendor + Year + Month + Lot(5) + Internal control code (1) + **Packing box number** (3) + `*` | `*IN-G6I00043Q001*` |
| QR CODE 1 (Model 2) | `ICT P/N\|D/C\|QTY\|V/C\|PO\|LOT NO\|CAVITY\|BIN\|X TYPE\|X QTY\|MSD`, phân cách bằng `\|` | `821-05582-MP-QSF\|20260925\|8015\|1003674\|4503582352\|G6I00043\|NA\|NA\|NA\|NA\|MSD5` |
| QR CODE 2 (Model 2) | = ID | |

### 2.2 Outer label (thùng)

| Field | Công thức | Ví dụ |
|---|---|---|
| L/C | Vendor + Year + Month + Lot(5), giống Inner LOT NO | `G6I00043` |
| ID (17 ký tự) | Vendor code of SEEV in ICT (7) + Internal control project (1, DE fix) + Internal control code (1, PMC) + Lot No. (5) + Packing box number (3) | `1003674` `E` `Q` `00043` `001` |
| SEEV CODE CONTROL (Code39) | `*OUT-` + Vendor + Year + Month + Lot(5) + Internal control code + Packing box number + `*` | `*OUT-G6I00043Q001*` |
| QR CODE 1 | `ID$V/C$VENDOR NAME$ICT P/N$CONFIG$L/C$D/C$QTY$CAVITY$BIN$X QTY$SECONDARY SUPPLIER$MSD`, phân cách bằng `$` | |

### 2.3 Dữ liệu PlusA / SaoMai quan sát được trong ảnh [Dữ kiện]

- **Tem PlusA dán trên thùng**: PARTS NO `69DRCT(25Y)`, Item code `4M002303000B`, LOT NO `G6D00492-20861`, Quantity `20,861 PCS`, `CASE 1`, `Label 1/1`, có QR.
- **Tem PlusA nhỏ**: Lot `G6D00492`, `7,940 PCS`, ngày `14 Apr 2026`, `Pack 001`, dấu QA PASS.
- **PlusA *Mass Products Shipping Entry***: `Item Tag BC` = Item code + Lot (`53067100000B00825`), có Shipping Instruction No `IDS265090`, Case No.
- **SaoMai**:
  - OuterBarcode dạng `IDS265030/27/4M002403000B/G6300243/20934`. [Suy luận] Cấu trúc là Shipping Instruction / Case / Item code / Lot / Qty.
  - InnerBarcode cùng cấu trúc, có thêm số túi.
  - Có AlBagID và OperatorID.
- **Shipping mark trên thùng**: `INV NO. IV-SEEV-26-5156`, `C/NO. 27/27`. Đây là **loại tem thứ ba** trên thùng mà spec chưa nhắc tới.
- **File gộp lot của PMC**: nhiều lot sản xuất (ví dụ `4M002402000B` lot `00470` và `00478`) được **gộp vào một combine lot** (`4M002403000B` lot `00224`). Mỗi lot gốc có Túi 1, Túi 2, Túi 3.

---

## 3. Nguyên nhân có thể gây trùng Reel ID [Suy luận – cần xác minh]

Reel ID bị khách hàng claim: `821-05582-MP-QSF260925043004` = P/N + D/C `260925` + Lot `043` + Bag `004`.

Theo công thức ở §2.1, Inner ID **không chứa đủ thông tin để duy nhất**. Các kịch bản dẫn đến trùng, sắp theo khả năng:

| # | Kịch bản | Cơ chế | Mức khả năng |
|---|---|---|---|
| H1 | **Gộp lot (combine lot)** | Hai lot sản xuất gốc gộp vào cùng một combine lot. Số túi đánh lại từ 001 cho từng lot gốc nhưng ID dùng chung lot và D/C → hai túi khác nhau cùng mang `...043004`. | Cao: file PMC cho thấy mỗi lot gốc có Túi 1–3 riêng. |
| H2 | **Cắt Lot 5 → 3 digits** | Lot `00043` và `01043` (hoặc `10043`) đều thành `043`. Lot hiện tại đã khoảng 00492–00825, nên chạm ngưỡng 1000 là chuyện sớm muộn. | Trung bình, sẽ tăng dần theo thời gian. |
| H3 | **Nhập tay Serial / Al bag number** | Input-11 và Input-12 do PMC gõ tay vào Excel. Copy dòng, quên tăng số, hoặc in lại đều có thể sinh trùng. | Trung bình. |
| H4 | **Nhiều Item code chung một ICT P/N và cùng D/C** | Hàng MP và Semi, hoặc hai item code (`4M002402000B`, `4M002403000B`) map cùng một P/N; lot trùng 3 số cuối. | Thấp–trung bình. |
| H5 | **In lại tem** | Không có kiểm soát reprint, nên tem cũ và tem mới cùng lưu hành. | Chưa rõ. |

**[Đề xuất]**

1. Lấy lại dữ liệu hai túi bị trùng (item code, lot gốc, lot gộp, ngày in, người in) để xác định H nào đúng **trước khi chốt thiết kế**.
2. ID phải do **hệ thống sinh ra**, không cho người nhập. Ràng buộc duy nhất phải nằm ở DB, áp dụng trên toàn bộ lịch sử, gồm cả các ID đã phát hành bằng Excel trước đây (cần import vào).
3. Nếu ICT cho phép đổi format ID, nên đưa lot 5 digits hoặc một số tuần tự duy nhất vào ID. Nếu ICT không cho đổi, hệ thống phải cấp số Al bag theo phạm vi **P/N + D/C + Lot(3)**, không theo lot gốc.

---

## 4. Phản biện theo từng phần

### 4.1 Định danh và chống trùng

| # | Vấn đề | Tác động |
|---|---|---|
| A1 | Spec viết "kiểm tra dữ liệu trùng lặp" nhưng không nói **trùng theo key nào, trong phạm vi nào** (Invoice? Lot? Toàn thời gian? Theo khách hàng?). | Không thiết kế được constraint. |
| A2 | Inner SEEV CODE CONTROL dùng **Packing box number** chứ không dùng Al bag number. Mọi túi trong cùng một thùng có cùng barcode. | Barcode này không định danh được từng túi. Cần xác nhận đây là chủ ý. |
| A3 | Outer ID không chứa Year/Month. Nếu Lot No. reset theo năm, lot `00043` năm 2025 và năm 2026 sẽ có cùng Outer ID. | Trùng ID giữa các năm. |
| A4 | Year chỉ có 1 digit nên chu kỳ lặp là 10 năm (2026 và 2036 cho cùng `G6I...`). | Rủi ro thấp, nhưng phải ghi nhận là giới hạn đã biết. |
| A5 | Internal control code (1 ký tự) do PMC "tự quy định" nhưng nằm trong Outer ID và barcode. | Không có quy tắc thì không đảm bảo được tính duy nhất. |
| A6 | Packing box number: 3 digits (Ref 01 mới) hay 5 digits ("Carton box serial 5 digits" trong form DE năm 2025)? Đánh số theo lot, theo invoice hay theo shipment? | Ảnh hưởng ID và barcode. |

### 4.2 Mô hình dữ liệu và quy cách đóng gói

| # | Vấn đề | Tác động |
|---|---|---|
| B1 | **Outer QTY = [Input-4] = Inner QTY.** Thực tế thùng có nhiều túi: tem PlusA thùng ghi 20,861, còn các túi là 8623, 8773, 3145… | Outer QTY phải là tổng số lượng các túi trong thùng (hoặc của lot trong thùng). Đây là lỗi spec. |
| B2 | Packing spec ghi: *"If multiple product lots are packed in one box, the label of each lot should be pasted on the box"*. Spec chưa mô hình hóa một thùng nhiều lot: mỗi lot một Outer label? Box number đánh thế nào? | Thiếu case, dễ in sai. |
| B3 | D/C của Outer: nếu các lot trong thùng khác DOM thì lấy giá trị nào? | |
| B4 | Cấu trúc Aluminum Bag → Box → **Pallet** được nhắc tới nhưng không có định nghĩa tem Pallet, cũng không có quy tắc gán thùng vào pallet. Tiêu đề lại nói về "Inner Label of **Packing Pallet**". | Phạm vi Pallet chưa rõ. |
| B5 | Combine lot: tem dùng lot gộp hay lot gốc? PlusA hiển thị lot nào? | Ảnh hưởng trực tiếp tới đối chiếu chéo (C2). |
| B6 | Định nghĩa DOM/D/C mâu thuẫn. Ref 01 E15 ghi "ENIG plating date", còn D15 và ảnh công thức ghi "Batch stock in date". | Sai nguồn dữ liệu thì sai cả ID. |
| B7 | Ví dụ trong Ref 01: DOM `20261022` (tháng 10, đáng ra là `J`) nhưng Month là `I` (tháng 9). Ngược lại, tem mẫu thực tế (D/C `20260925`, L/C `G6I`) thì khớp nhau. | Year/Month phải lấy cùng nguồn với DOM, hoặc spec phải nói rõ hai nguồn khác nhau. |
| B8 | Quy tắc Month: Jan→A … Dec→L, có bỏ chữ I/O không? Ví dụ `I` = tháng 9 cho thấy là không bỏ. Year ghi "2020->1, 2021->1", nhiều khả năng là lỗi gõ. | Cần một bảng quy đổi chính thức do Admin quản lý. |

### 4.3 Kiểm tra trước, trong và sau khi in

| # | Vấn đề | Tác động |
|---|---|---|
| C1 | Bước 7 yêu cầu: *"Với form mới, PMC phải điền khớp hoàn toàn với dữ liệu khai báo chuẩn từ Admin và DE"*. Nhưng các trường PMC nhập (P/N, Qty, PO, Lot) là dữ liệu biến động. "Khớp" ở đây so với cái gì? [Suy luận] Có thể ý là nhập hai lần (double-entry) các trường fix để đối chiếu với tờ chỉ thị giấy. | Không implement được nếu không làm rõ. |
| C2 | Bước 12 "cắt chuỗi, so chéo PlusA và tem hệ thống" chưa có: (a) nội dung QR/barcode của tem PlusA, (b) danh sách cặp field cần so (Lot? Qty? Item code ↔ P/N?), (c) cách xử lý combine lot và thùng nhiều lot. | Blocker cho chức năng đối chiếu. |
| C3 | Mục 2.2 viết "nhãn đầu tiên mỗi Invoice", sơ đồ viết "first **lot** of each invoice". Nếu chỉ kiểm tra lot đầu, các lot sau (lot, qty, DOM khác) không được kiểm tra. | Lỗ hổng kiểm soát. |
| C4 | Scan QR rồi so với DB chỉ chứng minh **máy in mã hóa đúng dữ liệu đã nhập**, không chứng minh **dữ liệu đúng thực tế**. Dữ liệu đúng hay sai phải so với chứng từ (PO, tờ chỉ thị, PlusA). | Kỳ vọng của bước 9 có thể sai. |
| C5 | Bước 11–12: scan 100% hay lấy mẫu? Scan tại trạm nào, ai scan, chặn luồng (interlock) hay chỉ cảnh báo? | Ảnh hưởng thiết kế trạm scan và thiết bị. |
| C6 | Shipping mark (Invoice / C/NO) không có trong đối chiếu, trong khi lỗi "nhầm thùng" thường xảy ra chính ở cấp này. | Cân nhắc đưa vào. |
| C7 | Khi lỗi thì spec ghi "xóa dữ liệu nhãn đã in". Thiếu quy trình **void** (hủy tem vật lý, ai được phép, có ghi log không) và **reprint** (giữ nguyên ID hay cấp ID mới, có giới hạn số lần, có cần phê duyệt không). | Nếu reprint không kiểm soát thì đây chính là nguồn sinh trùng (H5). |

### 4.4 Tích hợp và nguồn dữ liệu

| # | Vấn đề | Tác động |
|---|---|---|
| D1 | **SaoMai** đang lưu Invoice, Inner barcode, Outer barcode, AlBag ID (Ref 04, bước 8). Hệ thống mới thay thế SaoMai, ghi dữ liệu sang SaoMai, hay chạy song song? | Nếu chạy song song thì PMC phải nhập hai lần, rủi ro lệch dữ liệu. |
| D2 | **PlusA**: hệ thống mới đọc dữ liệu qua DB/API hay chỉ qua scan tem? Nếu đọc được Shipping Instruction, Lot, Qty từ PlusA thì có thể tự điền Year/Month/Lot/Qty thay vì nhập tay. | Bỏ được nguồn lỗi nhập tay lớn nhất. |
| D3 | **eMap** (Qty) và **QC summary**: spec ghi "mong muốn lấy trực tiếp từ eMap logfile". Có thuộc phạm vi phase 1 không? Định dạng log ra sao? | |
| D4 | **PMC Shipping Plan** (bước 5): chưa có file mẫu, chưa có quy tắc Auto-link (key là Item code + Customer + Lot?). | |
| D5 | **PO và ICT P/N** đang đến qua email của Sales. P/N của hàng Semi đổi theo Build. Ai cập nhật, cập nhật lúc nào, có version không? | P/N sai là sai toàn bộ ID. |

### 4.5 Cấu hình (Master Format / Template) và phân quyền

| # | Vấn đề |
|---|---|
| E1 | Key chống "khai báo trùng" Master Format/Template: MI + Label type? Item code + Label type + Revision? |
| E2 | Template có cần **phê duyệt** (DE tạo → ai approve) trước khi dùng in thật không? Có hiệu lực theo thời gian (effective date) không? |
| E3 | Đổi Template giữa chừng một Invoice: các tem đã in giữ revision cũ? Có in lại không? |
| E4 | Danh sách role chưa đầy đủ: SF Admin, DE, PMC, Operator, QC, và ai xem lịch sử? Xác thực bằng gì (AD/LDAP hay tài khoản riêng)? |
| E5 | Trong `[String]` công thức, độ dài cố định (P/N 16 ký tự) là bắt buộc hay chỉ là ví dụ? Có padding không? |

### 4.6 Phi chức năng (chưa có trong spec)

- **Thiết bị**: model máy in (Zebra/TSC/…), DPI, ngôn ngữ in (ZPL hay driver Windows), kích thước tem (mm), model máy scan 2D.
- **Hạ tầng**: on-prem hay cloud, số người dùng đồng thời, số trạm in và trạm scan, có cần chạy khi mất mạng không.
- **Hiệu năng**: thời gian phản hồi mỗi lần scan (đề xuất ≤ 1 s, cần xác nhận), số tem in mỗi ca.
- **Lưu trữ**: lịch sử in và scan giữ bao nhiêu năm, nhu cầu audit của khách hàng.
- **Ngôn ngữ UI**: VI, EN, hay cả hai.
- **Acceptance criteria / KPI**: ví dụ "0 Reel ID trùng sau go-live", "100% thùng được đối chiếu trước khi xuất". Đo bằng gì?
- **Migration**: import các ID đã phát hành từ Excel/SaoMai để kiểm tra trùng với lịch sử.
- **Đa khách hàng**: spec nói theo MI (ICT, UTAC), nhưng chỉ có tài liệu tham chiếu của ICT. Phase 1 chỉ làm ICT?

---

## 5. Brainstorm: phương án cho phần thiết kế và in tem

Đây là quyết định lớn nhất về chi phí. Spec yêu cầu một **WYSIWYG designer tự xây**.

| Tiêu chí | A. Tự xây WYSIWYG designer (như spec) | B. Dùng phần mềm thiết kế tem thương mại (BarTender / NiceLabel), hệ thống chỉ quản lý dữ liệu, validate, ID, lịch sử | C. Template cố định theo code (ZPL hoặc HTML), DE chỉ cấu hình field mapping và validation |
|---|---|---|---|
| Ưu điểm | Linh hoạt, không mất phí license | Designer đã hoàn thiện, hỗ trợ nhiều loại máy in, barcode/QR chuẩn | Rẻ, kiểm soát chặt, dễ test |
| Nhược điểm | Tốn công nhất, nhiều edge case (font, DPI, barcode) | Phí license, phải tích hợp qua SDK hoặc Integration Builder | Thêm format mới cần developer |
| Chi phí | Cao | Trung bình (license, chưa tra giá) | Thấp |
| Rủi ro | Cao (trễ tiến độ, lỗi render) | Thấp–trung bình | Thấp |
| Độ phức tạp triển khai | Cao | Trung bình | Thấp |
| Bảo trì | Khó | Trung bình | Dễ, nếu số format ít |

**[Đề xuất]** Tách hai mục tiêu:

1. **Phase 1**: dùng phương án C hoặc B, tập trung giải quyết lỗi trùng ID (sinh ID tự động, unique constraint, đối chiếu chéo, void/reprint, lịch sử).
2. **Phase 2**: mới làm designer, nếu số khách hàng và format thực sự nhiều.

Điều kiện để khuyến nghị này đúng: số format tem trong 12 tháng tới ít (ví dụ dưới 10). Con số này cần xác nhận.

### Ý tưởng khác cần cân nhắc

- **Scan tem PlusA để tự điền** Year/Month/Lot/Qty/Item code, không cho gõ tay. Dữ liệu PlusA đã có cấu trúc `G6D00492`.
- **Cấp ID tập trung** (sequence service) có khóa (lock) giữa các trạm in để tránh hai trạm cấp trùng khi in đồng thời.
- **Trạng thái tem** (state machine): `Generated → Printed → Verified → Applied → Shipped`, cùng các nhánh `Voided` / `Reprinted`. Mỗi lần chuyển trạng thái đều ghi log.
- **Interlock** ở trạm đóng gói: chưa đối chiếu OK thì không cho đóng thùng hoặc xác nhận lên pallet.

---

## 6. Danh sách câu hỏi cần làm rõ

Mức ưu tiên:

- **P1**: blocker, phải chốt trước khi thiết kế.
- **P2**: phải chốt trước khi code phần liên quan.
- **P3**: có thể chốt sau.

| ID | Ưu tiên | Câu hỏi | Hỏi ai | Liên quan |
|---|---|---|---|---|
| Q-01 | P1 | Hai túi mang ID trùng `821-05582-MP-QSF260925043004` thuộc item code, lot gốc, lot gộp nào? In lúc nào, ai in? Kịch bản nào trong H1–H5 đã xảy ra? | PMC, QC | §3 |
| Q-02 | P1 | ICT có cho phép thay đổi format Inner ID (ví dụ dùng Lot 5 digits) không? Format ID do ICT quy định hay do SEEV tự định nghĩa? | Sales, DE (hỏi ICT) | §3, A1 |
| Q-03 | P1 | Phạm vi duy nhất của từng mã (Inner ID, Outer ID, SEEV CODE CONTROL Inner/Outer) là gì: theo P/N, theo khách hàng, theo năm, hay vĩnh viễn? | DE, SF | A1–A3 |
| Q-04 | P1 | Nội dung QR/barcode của tem PlusA (trên túi và trên thùng) là gì? Cần so những cặp field nào với tem hệ thống? Combine lot xử lý thế nào? | PMC, IT PlusA | C2, B5 |
| Q-05 | P1 | Hệ thống mới thay thế, tích hợp hay chạy song song với SaoMai? Có đọc được dữ liệu PlusA qua DB/API không? | SF, IT | D1, D2 |
| Q-06 | P2 | Al bag number đánh theo lot gốc hay combine lot? Packing box number đánh theo lot, invoice hay shipment? Dài 3 hay 5 digits? | PMC | H1, A6 |
| Q-07 | P2 | Inner SEEV CODE CONTROL dùng Packing box number (không dùng Al bag number) là chủ ý? | DE | A2 |
| Q-08 | P2 | Lot No. có reset theo năm hoặc theo item code không? | PE, PMC | A3, H2 |
| Q-09 | P2 | Quy tắc chọn Internal control code (A, B, C…) là gì? Mỗi giá trị có ý nghĩa gì? | PMC | A5 |
| Q-10 | P2 | Outer QTY = tổng QTY các túi trong thùng (hoặc tổng của lot trong thùng)? Thùng nhiều lot thì mỗi lot một Outer label? D/C lấy theo lot nào? | DE, PMC | B1–B3 |
| Q-11 | P2 | DOM/D/C là ENIG plating date hay Batch stock in date? Year/Month lấy cùng nguồn với DOM? | DE | B6, B7 |
| Q-12 | P2 | Bảng quy đổi Year/Month chính thức (có bỏ I/O không)? | DE, Admin | B8 |
| Q-13 | P2 | Bước 7 "PMC điền khớp tuyệt đối với dữ liệu chuẩn" cụ thể là so những field nào, với nguồn nào? | SF | C1 |
| Q-14 | P2 | Kiểm tra sau in là nhãn đầu tiên mỗi Invoice hay lot đầu tiên mỗi Invoice? Các lot sau có cần kiểm tra không? | QC, SF | C3, C4 |
| Q-15 | P2 | Đối chiếu sau dán: scan 100% hay lấy mẫu? Chặn luồng hay chỉ cảnh báo? Có đưa Shipping mark (Invoice, C/NO) vào đối chiếu không? | QC, PMC | C5, C6 |
| Q-16 | P2 | Quy trình void và reprint: reprint giữ nguyên ID hay cấp ID mới? Ai được phép reprint, có cần phê duyệt không? | QC, SF | C7, H5 |
| Q-17 | P2 | Phạm vi Pallet: có tem Pallet không? Quy tắc gán thùng vào pallet? | PMC | B4 |
| Q-18 | P2 | Có import ID lịch sử (từ Excel, SaoMai) để kiểm tra trùng không? Dữ liệu có sẵn từ thời điểm nào? | IT, PMC | §4.6 |
| Q-19 | P2 | Phase 1 chỉ làm ICT hay gồm cả UTAC và các MI khác? | SF | §4.6 |
| Q-20 | P2 | WYSIWYG designer có bắt buộc trong phase 1 không? Chấp nhận dùng BarTender/NiceLabel hoặc template cố định không? | SF (quản lý) | §5 |
| Q-21 | P3 | Key chống trùng Master Format/Template? Template có cần quy trình approve, effective date không? Đổi template giữa chừng Invoice thì sao? | SF, DE | E1–E3 |
| Q-22 | P3 | Danh sách role và phương thức xác thực (AD/LDAP)? | IT | E4 |
| Q-23 | P3 | Model máy in, DPI, kích thước tem (mm), ngôn ngữ in; model máy scan; số trạm in và scan? | SF, IT | §4.6 |
| Q-24 | P3 | Hạ tầng (on-prem/cloud), yêu cầu khi mất mạng, thời gian lưu lịch sử, ngôn ngữ UI? | IT | §4.6 |
| Q-25 | P3 | Định dạng PMC Shipping Plan, eMap log, QC summary? Lấy Qty từ eMap có thuộc phase 1 không? | PMC, QC, IT | D3, D4 |
| Q-26 | P3 | KPI nghiệm thu (ví dụ 0 ID trùng, 100% thùng được đối chiếu) và kế hoạch UAT? | SF, QC | §4.6 |

---

## 7. Lỗi nhỏ trong file spec cần sửa

- Ref 01, E9: "2020->1, 2021->1" → nhiều khả năng đúng là "2020->0, 2021->1".
- Ref 01, E21: ví dụ Vendor code of SEEV ghi `6000023178`. Giá trị này là PO trong ví dụ cũ; vendor code đúng là `1003674`.
- Ref 01, E5: ví dụ P/N `001-101H-0257` khác với cột ví dụ `821-05582-MP-QSF`. Đây không phải lỗi, nhưng nên thống nhất một ví dụ.
- Ref 03: cột No bị lặp số `21` (Outer-21 và Outer-22).
- Ref 02 Inner-16 (Tem HF, RoHS) có `#VALUE!` dù chỉ là ảnh logo. Nên đổi thành text mô tả.
- Địa chỉ: packing spec cũ ghi "Dong Anh District", Ref 03 ghi "Thien Loc Commune, Ha Noi City". Cần xác nhận địa chỉ hiện hành và cách quản lý version của text fix.
- Sheet 02 B5: "găn chặn" → "ngăn chặn".
