# HANDOFF CHO CLAUDE AI — iLD Process Flow Traceability

Ngày bàn giao: 08/10/2026. Đây là tài liệu hiện hành. `HANDOFF_LEGACY.md` chỉ dùng tham khảo lịch sử của app offline.

## 1. Yêu cầu người dùng đã chốt

- Process Flow là nơi trung tâm để thực hiện truy xuất tại nhà máy cà phê iLD.
- Mỗi đại diện dùng máy riêng, truy cập web server nội bộ, đăng nhập và nhập dữ liệu công đoạn được giao.
- Một công đoạn có nhiều mẻ/PO. Một mẻ có nhiều đầu vào và nhiều đầu ra, bao gồm rework.
- Kết nối đầu ra công đoạn trước với đầu vào công đoạn sau; tính mass balance và truy ngược/xuôi từ vị trí đã có thông tin.
- Giao diện được gộp trong **một file HTML duy nhất** để bàn giao và tiếp tục chỉnh sửa.
- Một HTML không thay thế backend: giữ server cho đăng nhập, phân quyền và dữ liệu chung. Không biến đăng nhập thành kiểm tra mật khẩu bằng JavaScript hoặc LocalStorage.
- Ngôn ngữ giao diện tiếng Việt, nhận diện iLD Crafted nâu/kem. Không yêu cầu chuyển framework hay dùng dịch vụ cloud.

## 2. Bàn giao những gì

Chỉ cần gửi cho Claude **`index.html` và `HANDOFF.md`** để đọc toàn bộ nguồn hiện tại. HTML chứa cả giao diện legacy, template giao diện web và gói mã nguồn backend/tài liệu/tests nhúng trong JSON. Không chứa dữ liệu nhà máy, DB hay thông tin đăng nhập thực tế.

| Thành phần trong HTML | Vai trò và cách sửa |
|---|---|
| HTML/CSS/JS ngoài template | Các tab legacy: dashboard, exercise, email, checklist, trace, mass, report, settings và sơ đồ tham chiếu |
| `<template id="ild-lan-template">` | Nguồn giao diện Process Flow Web; CSS/JS nội bộ được cách ly trong iframe |
| `BEGIN ILD LAN FRAGMENT` / `END ILD LAN FRAGMENT` | Marker server dùng để trích giao diện web. Giữ nguyên, mỗi marker xuất hiện một lần ở dạng literal |
| `<script id="ild-backend-bundle" type="application/json">` | Gói nguồn bổ trợ: `files` là mapping đường dẫn tương đối → nội dung UTF-8. Không phải script thực thi |
| `downloadServerBundle()` | Nút Tải bộ server tạo ZIP từ nguồn nhúng qua ZIP writer có sẵn, không lấy DOM đang chạy hoặc dữ liệu LocalStorage |

**Không còn dùng `lan/app.html`.** Khi làm việc trong repo, sửa UI tại template trong `index.html`; sửa backend ở `lan/server.py`. Sau khi sửa file bổ trợ, chạy `python tools/sync_handoff_bundle.py` để cập nhật bản nhúng. Không sửa độc lập cả backend nhúng lẫn file server gây lệch phiên bản. Script chỉ đóng gói danh sách file nguồn cố định, không quét/thêm thư mục data.

## 3. Khôi phục repo từ HTML bàn giao

Cách dễ nhất: mở HTML bằng trình duyệt, bấm **Tải bộ server**, giải nén ZIP, rồi đặt **file index.html gốc** cạnh `START_WEB.cmd`. Giữ tên `index.html` vì server đọc đúng tên này. Đọc `lan/README.md`.

Nếu Claude có môi trường xử lý file, có thể trích JSON giữa thẻ script `ild-backend-bundle`, parse bằng JSON rồi ghi từng `files[name]` dưới một thư mục đầu ra. Trước khi ghi, kiểm tra đường dẫn đã resolve vẫn nằm trong thư mục đó; không ghi đè repo khác hoặc file người dùng mà chưa kiểm tra. Không dùng regex lấy toàn bộ mọi script làm JavaScript vì có script JSON.

Gói đi kèm gồm server, hai file CMD, tài liệu nghiệp vụ/handoff, tests và script đồng bộ gói. Sau khi trích, dùng file HTML nhận được làm `index.html`. DB được tạo khi setup, không được gửi kèm gói.

## 4. Chạy và các URL

Yêu cầu Python 3.12+, thư viện chuẩn. Không cần npm/framework để vận hành.

```powershell
python lan/server.py --init-admin
python lan/server.py
```

Hoặc chạy `SETUP_ADMIN.cmd` rồi `START_WEB.cmd`. Không có mật khẩu mặc định. Tạo tài khoản đầu tiên bằng terminal, nhập mật khẩu ẩn ít nhất 12 ký tự. Nếu DB đã có người dùng, không khởi tạo lại.

- `/`, `/lan`, `/lan/`: server trích template trong index.html và phục vụ giao diện Process Flow Web độc lập.
- `/legacy`: toàn bộ giao diện cũ; tab Process Flow dùng iframe srcdoc từ cùng template, không fetch một HTML thứ hai.
- Mở index.html trực tiếp bằng file://: các tab offline vẫn hoạt động; Process Flow Web chỉ xem trước màn hình đăng nhập, có thông báo cách chạy server và không cho gửi đăng nhập qua file://. Nút Sơ đồ tham chiếu offline mở sơ đồ kéo thả cũ.
- Localhost mặc định: http://127.0.0.1:8080.
- Dùng mạng nội bộ cần HTTPS/reverse proxy hoặc `--host 0.0.0.0 --cert ... --key ...`. Server không cho bind LAN HTTP thuần. Xem README cho proxy Host và backup.

Chưa cấu hình DNS, HTTPS, firewall, dịch vụ Windows hay triển khai cho máy trạm nhà máy. Không tự xem pilot là đã triển khai production.

## 5. Backend và mô hình dữ liệu hiện có

`lan/server.py` dùng ThreadingHTTPServer và SQLite WAL trên đĩa cục bộ server. Ghi qua BEGIN IMMEDIATE để kiểm tra phiên bản và tổng phân bổ trong cùng transaction. Không đặt SQLite trên SMB hoặc mở nhiều server từ các máy khác nhau vào một file mạng.

Bảng DB:

- `users`: id, username, name, role (`admin`, `qa`, `operator`), stages (JSON), password scrypt, active.
- `sessions`: hash token, user_id, expires. Token bearer ngẫu nhiên, phiên 8 giờ. UI chỉ giữ token trong bộ nhớ.
- `cases`: id, name, version, status (`open`, `closed`), created, snapshot.
- `events`: id, case_id, version, status, author, data JSON, updated.
- `history`: thời gian, người, action, entity, before/after JSON. Không giới hạn 500 dòng như legacy.
- `login_attempts`: giới hạn thử đăng nhập sai theo địa chỉ và username.

Phiếu `event.data`:

```text
stage, title, po, pic, equipment, start, end,
evidence, note, customer, limit, limitReason,
inputs[], outputs[], contexts[]
```

Mỗi dòng vật liệu: `id, material, batch, sscc, location, qty, unit, basis, factor, factorEvidence, kind`. Input liên kết thêm `sourceEvent, sourceLine`. Input kind: external/linked/opening. Output kind: product/rework/reject/sample/loss/stock. `contexts` là các ID phiếu Utility/CIP được đại diện chọn.

Stages: incoming, tipping, roasting, extraction, evaporation, liquid, drying, packing, warehouse, packaging, ingredients, rework, utility. Khi thêm stage phải cập nhật STAGES/STAGE_IDS backend và positions/route trong template UI.

ID nội bộ mới là định danh liên kết. Không tự nối lô bằng chuỗi batch; nhiều mã giống nhau có thể thuộc nguồn khác nhau. Dữ liệu sự kiện hiện được giới hạn trong từng đợt, chưa dùng lại genealogy giữa các đợt.

## 6. API chính

Tất cả trả JSON; request ghi dùng Content-Type application/json. Ngoài login, API yêu cầu Authorization: Bearer token. Origin nếu có phải phù hợp Host. Không có CORS mở rộng.

| Endpoint | Chức năng |
|---|---|
| POST /api/login | username/password → token/user |
| POST /api/logout | Xóa session |
| GET /api/state?case=ID | User, stages, cases, case, users, events kèm balance |
| POST /api/users | Admin tạo người dùng và công đoạn |
| POST /api/cases | QA/admin tạo đợt |
| POST /api/events | `{id?, caseId, version, data}`; kiểm quyền, phiên bản, nguồn và phân bổ |
| POST /api/transition | `{id, version, action, reason?}`; submit/verify/reopen |
| POST /api/close | `{id, version}`; snapshot và khóa đợt |
| GET /api/export?case=ID | JSON đợt hoặc snapshot đã đóng |
| GET /api/audit | QA/admin xem 200 thao tác gần nhất; DB vẫn giữ toàn bộ |

## 7. Quy tắc đang thực thi

- Operator sửa công đoạn trong stages được giao; QA/admin sửa toàn bộ, xác minh và đóng đợt. Tên PIC là người được phân công, khác người thực sự thao tác trong audit.
- Nháp → gửi xác nhận → verified. Sửa verified phải QA reopen có lý do. Cập nhật/reopen nguồn làm phiếu phụ thuộc đã gửi/xác minh chuyển needs_review.
- Kiểm `version` khi lưu/chuyển trạng thái; stale version trả 409, không ghi đè.
- Liên kết nguồn chỉ nhận output product/rework/stock, giữ material/batch/SSCC/unit. Tổng cấp cho mọi phiếu không vượt lượng output. Phiếu đang nháp cũng giữ phần phân bổ.
- Không xóa output đang có người nhận, đổi định danh của nguồn đang dùng hoặc tạo vòng giao dịch. Rework nhiều thế hệ phải là sự kiện mới dù mã batch có thể giữ nguyên.
- Truy bằng `traceSet()`: BFS/visited ID theo edges(). Cả hai = hợp của truy ngược và truy xuôi riêng từ cùng mẻ gốc; không duyệt đồ thị vô hướng làm lan sang mọi nhánh cùng tổ tiên.
- Search Batch/PO/SSCC neo vào phiếu/mẻ chứa thông tin tìm được. Chưa truy chính xác riêng từng phần của một output trong mẻ trộn.
- Balance: Σ(qty × factor) input/output từng basis. Thiếu factor/căn cứ hoặc thiếu một phía → incomplete. Từng basis kiểm sai lệch riêng, không bù chéo nhóm. Utility → not_applicable. QA verify cần balance ok, căn cứ ngưỡng và nguồn/context verified.
- Input external là ranh giới truy xuất/chứng từ ngoài app; không có nghĩa đã chứng minh đủ nguồn gốc.
- Poll 15 giây khi không đang nhập. Dirty form giữ nguyên; người dùng lưu rõ ràng. Refresh có xác nhận bỏ bản chưa lưu.
- Đóng đợt cần mọi phiếu verified, lưu snapshot và cấm thay đổi. Chưa có reopen đợt đã đóng.

## 8. Tình trạng legacy và các lỗi nghiệp vụ còn tồn tại

Legacy dùng `ild_trace_v1` trong LocalStorage, khác DB server. Các tab trace/mass/report legacy không tự tổng hợp dữ liệu Process Flow mới. Không xóa/migrate tự động dữ liệu này.

Các phát hiện đã có trong DE_XUAT_CAI_TIEN_TRUY_XUAT.md, **chưa được sửa trong logic legacy**:

- evalExercise coi đủ hồ sơ khi có ít nhất 1 yes và không có no, bỏ qua ô chưa đánh dấu.
- Báo cáo coi liên kết công đoạn đạt chỉ dựa vào cs.done > 0.
- Unknown deviation = |1 − N1×N2| có thể che hai sai lệch bù trừ.
- Chưa kiểm đủ dữ liệu cầu nối GC/Solid. Ngày receipt/production có chỗ lấy ngày đợt.
- Balance phụ thuộc dòng đầu/cuối trong khi cho thêm/xóa; mảng rỗng có thể lỗi.
- Quy tắc GC khác nhau giữa slide training và batch coding; không tự chuẩn hóa mã lịch sử khi chưa có SOP hiện hành.
- `reworkAddSolid` chú thích là dry solid nhưng công thức nhân lại fgSolid; phải kiểm Excel gốc/định nghĩa trước khi đổi.

Tài liệu 4 PPTX có tổng 63 slide đã được đọc trong phiên trước, nhưng không nhúng file PPTX vào HTML. Các Excel/Word/PDF được handoff cũ nhắc tới chưa được kiểm chứng ở vòng triển khai này. Không tuyên bố đạt tiêu chuẩn/audit dựa trên phiên bản ghi trong legacy.

## 9. Việc Claude cần ưu tiên hoàn thiện

1. Quản trị tài khoản: đổi/reset mật khẩu, vô hiệu hóa tài khoản, thu hồi phiên, quy trình cấp quyền và kiểm thử phân quyền. Hiện UI mới có tạo tài khoản.
2. Làm rõ phạm vi đợt và các công đoạn không áp dụng; mọi phiếu hiện verified chưa đủ chứng minh không bỏ sót công đoạn.
3. Bàn giao hai bên: lượng xuất và lượng thực nhận độc lập, trạng thái xác nhận và lý do chênh lệch. Hiện mới kiểm tổng phân bổ/tiêu hao.
4. Quản lý file chứng cứ an toàn, phiên bản, gắn event/lot; hiện chỉ mã/tham chiếu file.
5. Chuẩn hóa mass balance với QA: units, basis, solid/moisture/yield, tồn đầu/cuối, nhánh liquid/powder, PM, rework, hệ số có phê duyệt. Không áp dụng 0.84/0.97 mù quáng.
6. Cải thiện đồ thị: drill-down từ công đoạn tới mẻ và output cụ thể, xem lượng/chứng cứ trên cạnh, tránh đường chồng và không dùng vị trí mũi tên làm dữ liệu nguồn gốc.
7. Cân nhắc genealogy dùng chung giữa đợt, import SAP/CSV chống trùng, migration legacy có xem trước/map/đối chiếu và backup.
8. Báo cáo chính thức QA.F.029/030 từ DB mới. Giữ báo cáo cũ để đối chiếu; chưa tuyên bố hai hệ thống dữ liệu đã hợp nhất.
9. Triển khai và nghiệm thu với IT: HTTPS, backup/restore, vận hành dịch vụ, đo tải và thử từ nhiều máy thực. Chưa triển khai production.

Không cần làm tất cả trong một lần. Trước mỗi thay đổi, kiểm mã đang có, chọn phạm vi theo yêu cầu tiếp theo của người dùng và giữ migration dữ liệu cũ.

## 10. Kiểm thử và bàn giao lại

```powershell
python -m unittest discover -s tests -p test_lan.py -v
python tests/serve_fixture.py
# Trong terminal khác, nếu có Playwright và Chrome:
node tests/browser_lan.cjs
python tools/sync_handoff_bundle.py
```

Có thể đặt PLAYWRIGHT_MODULE là đường dẫn module Playwright sẵn có và BROWSER_CHANNEL là chrome/edge phù hợp. Fixture ở localhost:8097, DB test mới trong .test-output, không dùng tài khoản fixture cho dữ liệu thật. Dừng fixture sau kiểm thử. Không cần Playwright để vận hành ứng dụng.

Đã kiểm sau gộp: 11 test server đạt; tất cả JavaScript thực thi kiểm cú pháp đạt; Chrome kiểm login/save/verify/liên kết/truy hai chiều/report/audit/tài khoản/phân quyền/mobile; thêm login thật trong iframe srcdoc, 9 tab legacy file://, trạng thái preview offline và chuyển sang sơ đồ tham chiếu.

Khi sửa thêm cần kiểm export ZIP nguồn nhúng, parse JSON bundle và so từng file với source repo. Không đưa data/, .test-output/, mật khẩu hoặc TLS key vào file bàn giao. Sau thay đổi backend/docs/tests, chạy sync bundle rồi mới gửi HTML.

## 11. Prompt tiếp tục cho Claude

> Đọc HANDOFF.md này trước và kiểm tra index.html. Người dùng muốn Process Flow làm trung tâm truy xuất đa người dùng trên web nội bộ, mỗi đại diện đăng nhập và nhập phiếu công đoạn, nối nhiều đầu vào/đầu ra, mass balance và truy ngược/xuôi. Giao diện chỉ có một index.html; backend Python/SQLite được nhúng dưới dạng gói nguồn JSON và cũng có thể trích ra để chạy. Giữ phân quyền thực ở server, không giả lập bằng LocalStorage. Xác định phần nào đã có, nêu rõ giới hạn pilot, rồi tiếp tục theo yêu cầu mới của người dùng. Ưu tiên giữ dữ liệu, kiểm phiên bản đồng thời, chứng cứ và cân bằng đúng basis. Không tuyên bố đã triển khai LAN hoặc đạt yêu cầu QA khi chưa nghiệm thu. Đồng bộ gói nguồn trong HTML trước khi bàn giao lại.
