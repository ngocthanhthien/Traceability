# HAND-OFF — iLD Coffee Traceability App

Tài liệu bàn giao để một AI/model khác (hoặc chat mới) tiếp tục chỉnh sửa ứng dụng mà không cần lịch sử hội thoại. Đọc hết phần này trước khi sửa code.

---

## 1. Tổng quan

- **Sản phẩm:** Một ứng dụng web **1 file HTML duy nhất**, chạy **offline**, lưu dữ liệu bằng **LocalStorage**, phục vụ hệ thống **Truy xuất nguồn gốc (Traceability)** cho nhà máy cà phê hòa tan sấy thăng hoa **iLD Coffee Vietnam**.
- **File chính (deliverable):** `D:\14_TRACEABILITY\Traceability App\iLD_Traceability_App.html`
- **Người dùng:** 1 QA coordinator, dùng trên PC (Windows, Chrome). Không cần server, không cần internet.
- **Chuẩn tham chiếu:** FSSC 22000 v5.1 · BRC issue 9 · IFS Food v7.
- **Ngôn ngữ UI:** Tiếng Việt (có song ngữ Anh ở báo cáo).
- **Phong cách giao diện:** "Industrial Dashboard — Light", màu chủ đạo xanh dương `--brand:#0b5cab`.

### Ràng buộc kỹ thuật BẮT BUỘC giữ nguyên
1. **Chỉ 1 file HTML** — toàn bộ CSS + JS inline, KHÔNG tách file, KHÔNG thư viện ngoài, KHÔNG CDN.
2. **Offline tuyệt đối** — không fetch mạng. Lưu trữ = LocalStorage (key `ild_trace_v1`).
3. Vanilla JS thuần (ES6+), không framework. Vẫn chia module rõ bằng comment vùng mã.
4. Dùng được trên Chrome/Edge hiện đại. `localStorage`, `btoa`, `Blob`, `URL.createObjectURL` đều OK.

---

## 2. Cách chạy & kiểm thử

- **Chạy:** double-click file HTML → mở bằng Chrome. Script đặt cuối `<body>`, boot ngay (`if(document.readyState==='loading') addEventListener('DOMContentLoaded',boot); else boot();`).
- **Kiểm thử (không mở trình duyệt):** dùng **jsdom** trong môi trường Node (sandbox Linux).
  - Cài: `cd /tmp && npm install jsdom` (lưu ý: `/tmp` reset giữa các lần chạy bash → cài lại nếu mất).
  - Nạp file, `runScripts:'dangerously'`, rồi `w.eval('boot()')` (jsdom không tự fire DOMContentLoaded đúng lúc — gọi boot() tay để test).
  - Biến/hàm **khai báo bằng `function`** là global (gọi được qua `w.eval`); biến `const`/`let` top-level KHÔNG lên window — test qua `dom.window.eval('...')`.
  - **Kiểm cú pháp nhanh:** trích `<script>` ra file .js rồi `node --check`.
  - **Kiểm .xlsx:** ghi bytes ra file, mở bằng `openpyxl` (python) để xác nhận Excel đọc được.
  - **Kiểm .eml:** parse bằng `email` (python) để xác nhận header + body UTF-8 giải mã đúng.
- Sau mỗi chỉnh sửa: chạy lại smoke test điều hướng 8 tab + `node --check`.

---

## 3. Kiến trúc & bố cục code (theo thứ tự trong `<script>`)

Các vùng mã đánh dấu bằng comment `[TÊN]`:

- `[CONFIG]` — `CFG` (KEY, MAX_HOURS=4, DEV_LIMIT=0.5, RETAIN_YEARS=5) + `defaultSender()`.
- `[SEED]` — dữ liệu hạt giống:
  - `SEED_CONTACTS` (To), `SEED_CC` (Cc) — danh bạ PIC/email.
  - `SEED_BACKWARD`, `SEED_FORWARD` — danh mục biểu mẫu Annex 1 theo công đoạn/PIC.
  - `TRACE_SPEC`, `TRACE_ORDER`, `newTrace()`, `newTraceRow()` — tab Chi tiết truy xuất.
  - `MB_STAGES` (cũ, còn khai báo nhưng KHÔNG dùng nữa — có thể xóa).
  - Mass balance: `mkStageRow()`, `newMass()`, `stageSummary()`, `massCalc()`.
  - `evalExercise()` — đánh giá đạt/không.
- `[STORE]` — `Store` (save/load/defaults/exportJSON/importJSON).
- `[STATE]` — `DB` (toàn bộ dữ liệu), `State`, `currentEx()`, `logAct()`.
- `[UTILS]` — `$ $$ esc uid fmtDate fmtDay toast flashSaved confirmBox`.
- `[LOGIC]` — helpers: `buildChecklist checklistStats picProgress timeStatus fmtDur massCalc evalExercise`.
- `[UI]` — `Nav` (điều hướng tab) + các `render*()` cho từng tab + module hành vi.
- `[XLSX]` — module `Xlsx` tự sinh file .xlsx (zip + OOXML) không cần thư viện.
- `[BOOT]` — `boot()` + migration + đăng ký sự kiện.

### 8 Tab (sidebar)
`dashboard, exercise, email, checklist, trace, mass, report, settings`
Map trong `Nav.go()`: `{dashboard:renderDashboard, exercise:renderExercise, email:renderEmail, checklist:renderChecklist, trace:renderTrace, mass:renderMass, report:renderReport, settings:renderSettings}`.

Mỗi tab có `<section id="tab-XXX" class="page">`. Mỗi module hành vi là một object: `Ex` (đợt), `Cl` (checklist), `Tr` (chi tiết truy xuất), `Mb` (mass balance), `Email`, `Rep`, `Set`, `Xlsx`.

---

## 4. Data model (LocalStorage `DB`)

```
DB = {
  exercises: [ Exercise ],   // danh sách đợt truy xuất
  contacts:  [ {name,email,dept} ],      // To
  ccContacts:[ {name,email,dept} ],      // Cc
  catalog:   { backward:[{stage,pic,forms:[...]}], forward:[...] }, // khuôn checklist
  settings:  { maxHours:4, devLimit:0.5, sender:{...} },
  current:   "<exerciseId>" | null,      // đợt đang chọn
  log:       [ {t,msg} ]                 // audit log
}

settings.sender = { name, short, title, mobile, email, company, address }
  // 'short' = tên gọi dùng ở câu mở đầu email ("<short> xin gửi yêu cầu...")

Exercise = {
  id, createdAt, status:'draft'|'running'|'closed',
  batch, material, matNo, purpose, scenario,
  dir:'backward'|'forward',
  startAt, finishAt,          // epoch ms; đồng hồ đếm ngược 4h
  checklist: [ {stage,pic,items:[{id,form,status:''|'yes'|'no'|'na',receivedAt,remark}]} ],
  mass:  Mass,                // xem mục 6
  trace: Trace,               // xem mục 5
  capa, conclusion
}
```

Migration (trong `boot()`): tự thêm `trace`, `mass` (model mới), `ccContacts`, `settings.sender`, và merge liên hệ seed còn thiếu (Grasso/WTP) cho DB cũ. **Khi thêm field mới vào model, PHẢI thêm migration tương ứng ở boot() để DB cũ không vỡ.**

---

## 5. Tab "Chi tiết truy xuất" (trace) — theo mẫu QA.F.029 / QA.F.030

Nguồn: `D:\14_TRACEABILITY\QA.F.029 Forward Traceability Report.xlsx`, `QA.F.029-030 Traceability report,v2.xlsx`, `Traceability Report - 29.05_42080028F1.xlsx`.

```
Trace = {
  reason, participants:[{name,dept}] (8 dòng),
  incoming:[row], tipping:[row], process:[row], packing:[row], fgs:[row]
}
```
- `TRACE_SPEC` định nghĩa cột mỗi section. Có 2 loại:
  - `kind:'flat'` (incoming, fgs): bảng phẳng (Material No, Supplier, Batch, SSCC, dates, Qty, Customer...).
  - `kind:'io'` (tipping, process, packing): có nhóm cột **INPUT / OUTPUT** + cột **Gap tự tính = inQty − outQty**.
- `TRACE_ORDER.forward = [incoming,tipping,process,packing,fgs]`; `backward` đảo ngược. Report & tab tự đảo theo `ex.dir`.
- Báo cáo (`renderReport`) layout chính thức **8 mục**: 1 Thời gian · 2 Người tham gia · 3 Loại · 4 Lý do · 5 Thông tin truy xuất · 6 Kết quả (các section + Mass Balance) · 7 Kết luận (4 tiêu chí BRC) · 8 Điểm cải thiện + ký duyệt.

---

## 6. Tab "Mass Balance" — LOGIC ILD (quan trọng, bám sát file thực tế)

Nguồn logic: `D:\14_TRACEABILITY\ORGANIC MASSBALANCE SUMMARY_ILD.xlsx` (sheet **"ILD summary"** chứa công thức gốc) và `..._Olam.xlsx` (tham khảo logic cơ bản).

### Mô hình (`newMass()`)
```
Mass = {
  factors:{ roastYield:0.84, fgSolid:0.97 },
  solidFromExtraction:'',   // M9 — Tổng Solid từ Extraction thực tế (kg)
  reworkAddSolid:'',        // cộng thêm vào Output thực tế FGs (basis solid)
  unknownLimit:0.5,         // % ngưỡng
  stage1:[ row(Nhập hàng,roasted:false), row(Tipping,false), row(Roaster,roasted:true), row(Grinder,roasted:true) ],
  stage2:[ row(Extraction,isFG:false), row(Evaporation,false), row(FMT,false), row(FGs,isFG:true) ],
  note:''
}
row = {stage,batch,nsx,po,input,downgrade,loss,sampling,instock,actualOut,record,remark, roasted?|isFG?}
```

### Công thức (`massCalc()` — đã kiểm khớp 100% với sheet ILD)
Mỗi dòng có **hệ số quy đổi về basis nhóm**:
- Stage 1 quy về **GC (Green Coffee)**: dòng `roasted:true` (Roaster, Grinder) nhân `1/roastYield` (tức ÷0.84).
- Stage 2 quy về **Total Solid**: dòng `isFG:true` (FGs) nhân `fgSolid` (×0.97).

Cho mỗi nhóm:
```
Output lý thuyết (theo) = INPUT(dòng đầu) − Σ(downgrade·k) − Σ(loss·k) − Σ(sampling·k) − Σ(instock·k)
Output thực tế (actual) = actualOut(dòng cuối) · k(dòng cuối)   // stage2 cộng thêm reworkAddSolid trước khi ×k
Tỉ lệ thu hồi (recovery)= actual / theo
Gap = 1 − recovery
```
Nối 2 stage:
```
N9 (GC→Solid) = solidFromExtraction / s1.actual
Hệ số chuyển đổi tổng (overall) = s1.recovery × N9 × s2.recovery   // range tham khảo 25–53%
Unknown deviation (%) = |1 − s1.recovery × s2.recovery| × 100      // mục tiêu < 0.5%
hasData = (stage1[0].input>0 && grinder.actualOut>0)
```
`evalExercise().massOk = hasData && unknownDev <= unknownLimit`.

**Số liệu demo trong sheet ILD để đối chiếu khi test** (phải ra đúng): INPUT=30000, Tipping instock=9700, Grinder downgrade=252 & actualOut=8116, M9≈14002.857, Extraction input=3624, FGs downgrade=100/sampling=1/actualOut=3475, reworkAddSolid=380 → **N1=48.31%, N9=144.93%, N2=106.05%, overall=74.25%, unknownDev=48.77%** (demo cố ý không cân bằng).

> Lưu ý: model mass balance hiện FIXED 2 stage đặc thù cà phê (coffee-specific). Hệ số 0.84/0.97 chỉnh được trong tab Cài đặt-mass.

---

## 7. Tab "Soạn email" — khởi động quy trình truy xuất

Nguồn mẫu: `D:\14_TRACEABILITY\QA_DIEN_TAP_TRUY_XUAT_NGUON_GOC_BATCH_52610103F1.txt`, `QA_TRUY_XUAT_NGUON_GOC_CHO_ORGANIC_BATCH_52800075F1.txt`. Mẫu file .eml đích: `...Email-OE-Overview-ILD0016 (3).eml` (trong uploads).

- `buildEmail(ex)` sinh nội dung bám verbatim mẫu QA (câu mở đầu, bảng Batch/Material, phân công từng bộ phận, Note 3 mục, chữ ký). Câu mở đầu + chữ ký lấy từ `DB.settings.sender` (đã cho cấu hình được).
- `emailSubject(ex)` đổi tiêu đề theo `purpose`: Organic / Recall / mặc định "DIỄN TẬP".
- **Tạo email Outlook (.eml):** `Email.eml()` → `Email.buildEML()` sinh file RFC822 với:
  - Header **`X-Unsent: 1`** (Outlook mở ra = thư nháp CHƯA gửi, chỉ bấm Send).
  - Subject mã hóa `=?UTF-8?B?<base64>?=`; body `Content-Transfer-Encoding: base64`, `charset=UTF-8` (tiếng Việt chuẩn).
  - To/Cc lấy từ textarea `#emTo`/`#emCc` (người dùng sửa được trước khi tạo).
  - Tải xuống dạng `message/rfc822`, tên `QA_TruyXuat_<batch>.eml`.
- `Email.mailto()` giữ làm phương án dự phòng.
- **Chưa làm (gợi ý tiếp):** nhúng sẵn attachment (Annex 1 hoặc file .xlsx của đợt) vào .eml — hiện phải đính kèm tay.

---

## 8. Module `Xlsx` — xuất Excel .xlsx offline (không thư viện)

- Tự build **ZIP (store, no compression) + CRC32** + các part OOXML (`[Content_Types].xml`, `_rels`, `workbook.xml`, `styles.xml`, `worksheets/sheet1.xml`).
- Cell dùng `inlineStr` (không shared strings); hỗ trợ `mergeCells`, `cols` width, vài style (bold/title/header/section) qua `styles.xml`.
- `Xlsx.report(id)` xuất báo cáo 1 đợt theo layout QA.F.029/030 (gồm cả Mass Balance ILD). Tên file `QA.F.029_Forward_<batch>.xlsx` / `QA.F.030_Backward_...`.
- Helper nội bộ: `Sheet`, `cell()`, `sheetXML()`, `buildSection()` (trace), `buildMassStage()` (mass), `bytes(ex)` (đóng gói), `report(id)` (tải).
- Đã verify mở được bằng openpyxl. Khi sửa: giữ CRC32 + cấu trúc zip chuẩn, test lại bằng openpyxl.

---

## 9. Trạng thái hiện tại (ĐÃ xong)

- [x] 8 tab đầy đủ, style Industrial Light, auto-save LocalStorage, backup/restore JSON.
- [x] Dashboard KPI + nhắc việc (2 đợt/năm, quá hạn 4h, thiếu hồ sơ).
- [x] Đợt truy xuất: thông tin lô, chọn chiều, đồng hồ đếm ngược 4h, nhân bản/xóa, đóng & đánh giá.
- [x] Soạn email: template bám mẫu QA, To/Cc, tạo .eml (X-Unsent), người gửi cấu hình được.
- [x] Checklist hồ sơ: seed Annex 1 (Backward ~94 form / Forward ~13), Có/Không/N/A, % theo bộ phận, xuất CSV.
- [x] Chi tiết truy xuất: 5 section theo QA.F.029/030, Gap tự tính, đảo chiều.
- [x] Mass Balance: logic ILD 2-stage, hệ số chỉnh được, khớp công thức file ILD.
- [x] Báo cáo: layout chính thức 8 mục + in/PDF + xuất .xlsx.
- [x] Cài đặt: thông số, người gửi email, danh bạ To/Cc, danh mục Annex 1, audit log, reset.

## 10. Gợi ý việc tiếp theo (chưa làm)
- Nhúng attachment (Annex 1 / .xlsx) trực tiếp vào .eml.
- Xuất Excel 2 sheet (Forward + Backward) hoặc khung viền đậm hơn cho giống biểu mẫu in.
- QR/Barcode cho batch; đa ngôn ngữ EN/VI; PWA cài như app.
- Xóa `MB_STAGES` (code chết). Cân nhắc cho phép cấu hình số stage/hệ số của mass balance linh hoạt hơn (hiện fixed 2 stage).

---

## 11. Quy ước khi sửa code
- Giữ comment vùng `[TÊN]` và phong cách module-object (`Ex`, `Mb`, `Tr`...).
- Mỗi lần đổi dữ liệu → gọi `Store.save()`; thao tác đáng ghi → `logAct()`.
- Input số: lưu `oninput`, rerender `onchange` (tránh mất focus khi gõ).
- Thêm field vào model → thêm migration ở `boot()`.
- Escape mọi chuỗi đưa vào HTML bằng `esc()`.
- Tiếng Việt: dùng số định dạng `fmtNum` (vi-VN, dấu `.` ngăn nghìn) nhưng LƯU số thực.
- Sau sửa: `node --check` + smoke test 8 tab (+ openpyxl nếu đụng Xlsx).

## 12. File nguồn tham chiếu (trong D:\14_TRACEABILITY)
- Quy trình: `QA.P.00X TRACEABILITY PROCEDURE - 002.docx`, `6589.QA.P.009 TRACEABILITY PROCEDURE.pdf`
- Training: `iLD_Traceability System Training.pptx`
- Danh mục hồ sơ: `Annex 1 List of records, documents for traceability sytem.xlsx`
- Report mẫu: `QA.F.029 Forward Traceability Report.xlsx`, `QA.F.029-030 Traceability report,v2.xlsx`, `Traceability Report - 29.05_42080028F1.xlsx`
- Mass balance: `ORGANIC MASSBALANCE SUMMARY_ILD.xlsx` (logic gốc + công thức), `..._Olam.xlsx`
- Email mẫu: 2 file `QA_..._BATCH_*.txt` + `...Email-OE-Overview-ILD0016 (3).eml`

---
*Hand-off tạo ngày 2026-10-07. App do QA iLD Coffee đặt làm; mọi chỉnh sửa giữ nguyên ràng buộc 1-file / offline / LocalStorage.*
