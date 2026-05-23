# PHẦN NÀY SẼ NÂNG CẤP BÀI WORFPRESS_WEB-SITE CŨ, KHAI THÁC N8N ĐỂ TỰ ĐỘNG ĐĂNG BÀI LÊN WORDPRESS

Đầu tiên gõ lệnh dọn dẹp ổ cứng này:

```bash
docker system prune -a -f
```
<img width="1919" height="1079" alt="Screenshot 2026-05-22 170412" src="https://github.com/user-attachments/assets/c9846cb8-1729-4ba9-a3c7-90005f04985e" />

Lệnh này sẽ xóa sạch các bản cài đặt cũ, các Image không dùng tới của các bài Lab trước, giúp giải phóng ngay vài gb trống để chạy `n8n` mượt mà.

# BƯỚC 1: CẤU HÌNH FILE docker-compose.yml 5 SERVICE

Di chuyển vào thư mục dự án:

```bash
cd ~/wordpress_project
```

và dùng lệnh:

```bash
nano docker-compose.yml
```

để cập nhật lại toàn bộ file thành cấu hình chuẩn dưới đây (Đã tối ưu cho tên miền `sunning.id.vn`):

```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:latest
    container_name: mariadb
    environment:
      TZ: "Asia/Ho_Chi_Minh"
      MARIADB_ROOT_PASSWORD: rootpassword
      MARIADB_DATABASE: wordpress_db
      MARIADB_USER: wp_user
      MARIADB_PASSWORD: wp_password
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    environment:
      PMA_HOST: mariadb
      PMA_ARBITRARY: 1
    depends_on:
      - mariadb
    restart: unless-stopped

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    environment:
      WORDPRESS_DB_HOST: mariadb
      WORDPRESS_DB_NAME: wordpress_db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_password
    volumes:
      - wp_data:/var/www/html
    depends_on:
      - mariadb
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    environment:
      - WEBHOOK_URL=https://n8n.sunning.id.vn/
      - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - mariadb
    restart: unless-stopped

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel --no-autoupdate run --token <MÃ_TOKEN_CLOUDFLARE_CỦA_BẠN>
    restart: unless-stopped

volumes:
  db_data:
  wp_data:
  n8n_data:
```
<img width="1919" height="1079" alt="Screenshot 2026-05-22 171648" src="https://github.com/user-attachments/assets/98d4d415-c9be-4091-b318-ecac612972eb" />

( Lưu ý nhớ thay chỗ `<MÃ_TOKEN_CLOUDFLARE_CỦA_BẠN>` bằng cái đoạn `Token` dài loằng ngoằng lấy từ trang Dashboard Cloudflare Tunnel của vào ).

## Chạy kích hoạt hệ thống:

Sau khi lưu file, gõ lệnh thần thánh:

```bash
docker compose up -d
```
<img width="1919" height="291" alt="Screenshot 2026-05-22 175741" src="https://github.com/user-attachments/assets/c7fe04c8-d95c-43c5-8249-e50857865db2" />

Chờ hệ thống tải và chạy xong, gõ:

```bash
docker compose ps
```
<img width="1906" height="198" alt="image" src="https://github.com/user-attachments/assets/002dcaa7-ae87-43ce-9864-3bad55b2912f" />

để kiểm tra. Đảm bảo cả `5` dịch vụ đều báo chữ `Up` (hoặc `running`), không cái nào bị `Exit` hay `Restarting`.

---

# BƯỚC 2: ĐỊNH TUYẾN (ROUTE DNS) TRÊN CLOUDFLARE DASHBOARD

Đăng nhập vào giao diện web `Cloudflare Zero Trust -> Networks -> Tunnels -> Chọn Tunnel và bấm Edit`.

Tại tab `Public Hostname`, tiến hành `Add` thêm `3` đường dẫn chuẩn:

## Sub-domain 1 (WordPress):
## Sub-domain 2 (phpMyAdmin):
## Sub-domain 3 (n8n):

<img width="1917" height="1023" alt="Screenshot 2026-05-22 180644" src="https://github.com/user-attachments/assets/f124432b-621c-4fa8-9aa6-c1fe3a83401a" />


---

# BƯỚC 3: KIỂM TRA DATABASE VÀ CÀI ĐẶT WORDPRESS THỦ CÔNG

## Quan sát DB ban đầu:

Truy cập vào `https://pma.sunning.id.vn`.

Đăng nhập bằng tài khoản `wp_user / wp_password`.

Bấm vào database `wordpress_db`, sẽ thấy chưa có bảng dữ liệu nào cả.
<img width="1918" height="1022" alt="Screenshot 2026-05-22 193219" src="https://github.com/user-attachments/assets/dc5a39e9-49fa-4e42-992b-d4827ea6a748" />

## Cài đặt WordPress:

Truy cập `https://wp.sunning.id.vn` và tiến hành các bước thiết lập ban đầu (`Chọn ngôn ngữ Tiếng Việt, đặt tên Web, tạo tài khoản Admin cho WordPress`).
<img width="1064" height="838" alt="Screenshot 2026-05-22 193551" src="https://github.com/user-attachments/assets/1499480e-cd2e-4169-8b0b-4c05b2c8b460" />

## Quan sát DB sau khi cài:

Quay lại trang phpMyAdmin lúc nãy và ấn `F5`.

Sẽ thấy hệ thống tự động sinh ra mười mấy bảng dữ liệu dạng `wp_posts`, `wp_users`, `wp_comments`... 
<img width="1807" height="860" alt="image" src="https://github.com/user-attachments/assets/e144c8f1-6ac7-4939-8f0d-11a89eae14cd" />

## Đăng 2 bài viết thủ công:

Vào `https://wp.sunning.id.vn/wp-admin`, tạo và đăng `2` bài viết (test wordpress nếu cần):

---

# BƯỚC 4: KÍCH HOẠT LICENSE CHO N8N

Truy cập vào `https://n8n.sunning.id.vn` và đăng ký tài khoản Admin bằng Email thật của mình.

`N8n` sẽ hiện một bảng yêu cầu gửi `License key` bản Community miễn phí. Điền đầy đủ thông tin của bạn vào đó để `n8n` gửi email key về.

Mở hòm thư Email của bạn, tìm thư từ `n8n` và sao chép đoạn mã kích hoạt.

Trên giao diện `n8n`:

`Settings (biểu tượng bánh răng ở góc dưới bên trái) -> Usage and plan -> Bấm vào Enter activation key -> Dán đoạn mã từ Email vào -> Chọn Activate`.

Nhìn xuống góc dưới bên phải thấy thông báo:

```text
Your Registered Community Edition has been successfully activated
```
<img width="491" height="229" alt="Screenshot 2026-05-22 201413" src="https://github.com/user-attachments/assets/6b104426-7136-40cb-93f6-668cc506b262" />

là xong.

---

# BƯỚC 5: XÂY DỰNG LUỒNG TỰ ĐỘNG HÓA (WORKFLOW) TRÊN N8N

Tại trang chủ `n8n`, chọn `Create workflow`.
<img width="507" height="299" alt="Screenshot 2026-05-22 202207" src="https://github.com/user-attachments/assets/af219900-ddd5-4aec-8dd3-fcd50eee5d4e" />

Sẽ xây dựng một chuỗi gồm `4 Node` liên kết với nhau như sau:
<img width="1919" height="1023" alt="Screenshot 2026-05-22 202254" src="https://github.com/user-attachments/assets/54326887-5ba1-4210-b8b1-1223f9b3678d" />

## Node 1: Telegram Trigger (Điểm bắt đầu)

Thêm node mới, tìm từ khóa `Telegram`, chọn `OnMessage (Telegram Trigger)`.

Đăng nhập vào ứng dụng `Telegram` trên điện thoại, tìm con bot `@BotFather`.

Gõ lệnh `/newbot` để tạo một con bot mới của riêng. Đặt tên hiển thị và tên username cho bot.

`@BotFather` sẽ cấp cho một chuỗi `HTTP API Token`. Hãy sao chép nó.
<img width="1919" height="1079" alt="Screenshot 2026-05-22 203639" src="https://github.com/user-attachments/assets/423fcbbc-7855-4372-b007-0666584f61bc" />

### Mẹo quan trọng:

Tìm đúng tên con bot  vừa đẻ ra, nhấn `Start` và chát một câu bất kỳ (`Ví dụ: "hello bot"`) vào đó để kích hoạt bot.
<img width="1919" height="1079" alt="Screenshot 2026-05-22 204249" src="https://github.com/user-attachments/assets/2f56a97c-7e8d-4e1d-a9cf-aba5ea779afc" />

Quay lại giao diện `n8n`:

Tại phần `Credential`, chọn `Create New Credential`, dán chuỗi `API Token` vừa lấy vào đây.
<img width="1919" height="1028" alt="Screenshot 2026-05-22 203849" src="https://github.com/user-attachments/assets/19da9f26-e2b2-42a5-bd45-e877d8069418" />

---

## Node 2: Google Gemini (Não bộ AI)

Truy cập vào trang `Google AI Studio` để tạo một project mới và bấm lấy `API KEY`.
<img width="1919" height="1023" alt="Screenshot 2026-05-22 204731" src="https://github.com/user-attachments/assets/c75858f4-0dd1-4fe1-872e-80e5ba19c58f" />

Trên `n8n`, kéo một mũi tên từ Node Telegram ra để tạo Node tiếp theo:

Tìm `Google Gemini`, chọn hành động `Message a model`.
<img width="1918" height="1022" alt="Screenshot 2026-05-22 204605" src="https://github.com/user-attachments/assets/f7df0898-5aa4-450d-ba10-29615bd54585" />

### Credential:

Chọn tạo mới và dán đoạn `API Key` của Gemini bạn vừa lấy được vào.
<img width="1919" height="1021" alt="Screenshot 2026-05-22 204831" src="https://github.com/user-attachments/assets/1c40f2b8-eba4-404b-9750-deaae069493a" />

### Cấu hình phần Prompt:

Tại ô `Text`, bạn hãy kéo thả biến từ Node Telegram ở cột bên trái sang hoặc gõ chính xác dòng lệnh sau:

```plaintext
{{ $json.message.text }}. Kết quả sinh ra ở định dạng HTML+CSS để tôi dùng HTML+CSS này tạo bài viết cho wordpress. Hãy định dạng kết quả trả về là một chuỗi JSON thuần túy chứa đúng 2 trường: "post_title" (Tiêu đề bài viết) và "post_content" (Nội dung bài viết chứa mã HTML). Tuyệt đối không bọc chuỗi trong ký tự markdown code block (không dùng ```json).
```
<img width="1919" height="1020" alt="Screenshot 2026-05-22 205547" src="https://github.com/user-attachments/assets/9f536398-04ac-457c-9023-59ee865b6aaf" />

### Bật tính năng JSON:

Kéo xuống dưới, gạt công tắc `Turn on Output Content as JSON` sang màu xanh.
<img width="1919" height="1025" alt="Screenshot 2026-05-22 205552" src="https://github.com/user-attachments/assets/9b047874-cb09-4318-88da-6e3b6a6e6c27" />

### Mục Options:

Bấm `Add Option -> Chọn System Message` và điền nội dung:

```text
"Bạn là một chuyên gia viết blog chuẩn SEO, hành văn cuốn hút, các thẻ tiêu đề H2, H3 mạch lạc, bôi đậm từ khóa quan trọng."
```
<img width="1918" height="1023" alt="Screenshot 2026-05-22 205652" src="https://github.com/user-attachments/assets/e5c411d4-81e7-4288-8358-20aae2553952" />

Nhận xét: `Option` này rất đáng dùng để ép AI viết bài đúng form mẫu, không bị trả về văn bản thừa thãi.

---

## Node 3: Code in JavaScript (Bộ lọc dữ liệu)

Nối tiếp sau node Gemini, thêm node `Code in JavaScript`.
<img width="539" height="861" alt="Screenshot 2026-05-22 210036" src="https://github.com/user-attachments/assets/62875dda-2699-4f77-9ab2-d36f7f4b707e" />

Xóa sạch các đoạn code mặc định có sẵn trong khung và dán chính xác đoạn code JavaScript của bài tập vào:

```javascript
// 1. lấy dữ liệu gốc từ Gemini trả về
const rawText = $input.first().json.content.parts[0].text;

// 2. Chuyển đổi chuỗi thành Object trong JavaScript
const cleanData = JSON.parse(rawText);

// 3. Trả về kết quả định dạng lại gọn gàng cho n8n sử dụng
return {
  title: cleanData.post_title,
  content: cleanData.post_content
};
```
<img width="1919" height="1026" alt="Screenshot 2026-05-22 210110" src="https://github.com/user-attachments/assets/fd804088-28f4-4e35-b115-85fcfa43e883" />

---

## Node 4: WordPress (Đích đến - Xuất bản bài viết)

### Lấy mật khẩu ứng dụng:

Vào trang admin WordPress của bạn (`https://wp.sunning.id.vn/wp-admin`) -> Chọn mục `Thành viên (Users)` -> `Hồ sơ của bạn (Profile)` -> Cuộn xuống dưới cùng tìm mục `Mật khẩu ứng dụng (Application Passwords)`.

Nhập tên là `n8n` rồi ấn nút `Thêm mật khẩu ứng dụng mới`.

Hệ thống sẽ hiện ra một chuỗi gồm `24 ký tự chữ cái`. Hãy copy chuỗi này.
<img width="1919" height="1025" alt="Screenshot 2026-05-22 211042" src="https://github.com/user-attachments/assets/015e9815-e77f-40b9-9cb3-07fc77890618" />

### Tại n8n:

Thêm node cuối cùng là `WordPress`, chọn hành động là `Create a Post`.
<img width="1919" height="1027" alt="Screenshot 2026-05-22 210731" src="https://github.com/user-attachments/assets/5d5408b9-ae74-4d5c-a3a6-fbf958dc6173" />

### Credential:

Chọn tạo mới.

- Username: Điền tài khoản Admin đăng nhập web WordPress của bạn.
- Password: Dán chuỗi mật khẩu ứng dụng `24 ký tự` vừa copy ở trên vào.
- Wordpress URL: Điền chính xác `https://wp.sunning.id.vn/`

Gạt công tắc bật tính năng `Ignore SSL Issues (Insecure)` sang `ON`.
<img width="1919" height="1020" alt="Screenshot 2026-05-22 211315" src="https://github.com/user-attachments/assets/497d0a5e-87e2-4948-aee7-dbd43ca727df" />

### Cấu hình Node:

Bấm nút `Execute previous nodes` ở phía trên để `n8n` chạy thử và lấy dữ liệu mẫu. Sau đó:

- Tại ô `Title`: Kéo thả trường dữ liệu `title` từ Node JS ở cột bên trái sang.
- Tại ô `Content`: Kéo thả trường dữ liệu `content` từ Node JS ở cột bên trái sang.
<img width="1919" height="1024" alt="Screenshot 2026-05-22 211417" src="https://github.com/user-attachments/assets/8cf20c5b-6c96-4b8b-8332-73b9c14107be" />

### Đăng bài trực tiếp:

Bấm vào mục `Add Field -> Chọn thuộc tính Status -> Chuyển giá trị của nó từ Draft (Bản nháp) thành Publish (Xuất bản)` để bài viết được hiển thị công khai ngay lập tức.
<img width="1919" height="1030" alt="Screenshot 2026-05-22 213620" src="https://github.com/user-attachments/assets/d844f6c8-b51c-462f-b952-87182e8b6d66" />

---

# BƯỚC 6: KÍCH HOẠT HỆ THỐNG VÀ KIỂM TRA THÀNH QUẢ

Tại góc trên bên phải giao diện `n8n`, bạn bấm gạt công tắc từ `Inactive` sang `Active` (`hoặc nút Publish/Save workflow`) để kích hoạt hệ thống chạy tự động.

## Thử nghiệm thực tế:

Lấy điện thoại hoặc mở telegram web ra, vào con bot vừa tạo và gửi đúng tin nhắn yêu cầu bài tập:

```text
👉 "Tạo bài viết khen thầy Đỗ Duy Cốp dạy môn Phát triển ứng dụng với mã nguồn mở"
```
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/04ffe8dc-dacc-4ef9-86ae-fc3a3b90f7f8" />

Chờ khoảng `15 - 20 giây` để `n8n` chuyển tiếp tin nhắn qua Gemini xử lý cấu trúc JSON và đẩy về WordPress.

Mở trang chủ `https://wp.sunning.id.vn` lên và `F5` lại và tận hưởng thành quả <3
<img width="1919" height="1026" alt="image" src="https://github.com/user-attachments/assets/3a90c029-e597-449a-9400-4203855601a8" />
# Nhận xét:
# Hệ thống đã vận hành trơn tru theo quy trình

`Telegram (Input) -> Google Gemini (AI xử lý nội dung & cấu trúc HTML) -> JavaScript (Xử lý, làm sạch dữ liệu JSON) -> WordPress (Đăng tải tự động)`

Hệ thống đã loại bỏ được thao tác thủ công, đảm bảo định dạng văn bản chuyên nghiệp và tự động hóa toàn bộ quy trình sản xuất nội dung.

---

# 1. Nhận xét thành quả

## Hạ tầng

Cài đặt và chạy thành công `5 container`:

- `MariaDB`
- `phpMyAdmin`
- `WordPress`
- `n8n`
- `Cloudflared`

trên Ubuntu qua Docker Compose. Các dịch vụ kết nối nội bộ ổn định.

## Mạng & Domain

Cấu hình thành công `Cloudflare Tunnel` để đưa web `WordPress` và `n8n` ra ngoài Internet bằng tên miền riêng (`sunning.id.vn`) qua HTTPS an toàn mà không cần mở port mạng nhà.

## Tự động hóa

Thiết lập luồng `n8n` chạy mượt:

```text
Gõ lệnh bằng tiếng Việt trên Telegram
        ↓
Gemini AI tự viết bài chuẩn HTML
        ↓
Code JS lọc dữ liệu
        ↓
Tự động đăng lên WordPress sau 15 giây
```

---

# 2. Các lỗi gặp phải và cách xử lý

## Lỗi 1: Bay màu toàn bộ tài khoản và bài viết cũ

### Nguyên nhân

Do lỡ tay chạy lệnh:

```bash
docker compose down -v
```

(`tham số -v xóa sạch ổ đĩa ảo Volume`)

### Xử lý

Vào `phpMyAdmin` tạo lại database `wordpress_db` từ đầu và setup lại tài khoản admin mới cho `WordPress`.

---

## Lỗi 2: Lỗi mạng sập nguồn 1033 từ Cloudflare

### Nguyên nhân

Do đổi database và cài lại `WordPress` làm kẹt cấu hình cũ trong hầm `Cloudflared`.

### Xử lý

Chạy lệnh:

```bash
docker logs --tail 20 cloudflared
```

để soi log lỗi.

Sau đó lên web `Cloudflare Zero Trust` xóa `Hostname` cũ đi, tạo cái mới trỏ chuẩn về:

```text
wordpress:80
```

---

## Lỗi 3: Node Code JavaScript bị lỗi crash dữ liệu

### Nguyên nhân

Cấu trúc dữ liệu chữ (`text`) của `Gemini` trả về nằm sâu hơn bình thường nên đoạn code cũ không tìm thấy đường dẫn (`bị báo undefined`).

### Xử lý

Nhìn vào bảng cấu hình `INPUT` bên trái của `n8n`, sửa lại dòng code đầu tiên thành:

```javascript
json.content.parts[0].text
```

để bóc tách dữ liệu chuẩn.

---

## Lỗi 4: Bài viết bị lặp 2 cái tiêu đề to đùng

### Nguyên nhân

Do `Prompt` ban đầu chưa chặt chẽ làm AI tự tiện chèn thêm thẻ `<h1>` tiêu đề vào đầu phần nội dung bài viết.

### Xử lý

Thêm `System Message` định hình vai trò chuyên gia cho AI và sửa lại `Prompt` ra lệnh cấm tuyệt đối dùng thẻ `<h1>` trong nội dung, bắt đầu viết bài thẳng bằng thẻ `<p>` và `<h2>`.

---

## Lỗi 5: Lỗi nghẽn mạng "Service unavailable" của Gemini

### Nguyên nhân

Server của Google bị quá tải tạm thời do nhiều người dùng cùng lúc.

### Xử lý

Bật tính năng `Retry On Fail` trong tab `Settings` của node `Gemini` để `n8n` tự động gửi lại lệnh khi bị ngắt kết nối.
