# Báo cáo Khắc phục và Vận hành Config Server - FoodX

## 1. Phân tích lỗi và Cách sửa đổi

### Lỗi 1: Thiếu annotation `@EnableConfigServer` ở main class
- **Nguyên nhân**: Khi thiếu annotation `@EnableConfigServer`, Spring Boot sẽ khởi chạy như một ứng dụng Web thông thường dựa trên auto-configuration có sẵn trong classpath. Nó không kích hoạt các Component nội bộ của Spring Cloud Config (như `Marker` bean cần thiết để bật `ConfigServerAutoConfiguration`). Điều này dẫn tới việc các endpoint cấu hình như `/{name}/{profile}` không được đăng ký. Khi `restaurant-service` gọi tới port `8888`, server sẽ trả về lỗi HTTP 404 (hoặc 500 nếu có lỗi internal phát sinh).
- **Khắc phục**: Thêm `@EnableConfigServer` ngay phía trên class `ConfigServerApplication` cùng với `@SpringBootApplication`.

### Lỗi 2: Sai đường dẫn Git URI (`git.uri` thiếu ".git")
- **Nguyên nhân**: Thiếu hậu tố `.git` ở cuối URI có thể làm cho một số thư viện Git client nội bộ của JGit không thể xác định chính xác giao thức git thô và parse repository metadata, dẫn tới lỗi kết nối hoặc cloning fail.
- **Khắc phục**: Đổi giá trị thành `https://github.com/foodx/config-repo.git`.

### Lỗi 3: Sai nhánh mặc định (`default-label` là `master` thay vì `main`)
- **Nguyên nhân**: Các kho lưu trữ (repository) hiện đại ngày nay thường sử dụng nhánh `main` làm mặc định thay vì `master`. Việc cấu hình sai nhánh dẫn đến việc Config Server cố gắng checkout nhánh không tồn tại và ném ra lỗi `GitAPIException` (lỗi 500 về phía client).
- **Khắc phục**: Thay đổi `default-label: master` thành `default-label: main`.

---

## 2. Mô tả luồng xử lý yêu cầu (Request Flow)

Khi `restaurant-service` gửi request `GET /restaurant-service/prod` tới Config Server, luồng xử lý diễn ra như sau:

1. **Tiếp nhận request**: Config Server nhận HTTP request tại endpoint `/{name}/{profile}`. Ở đây `name = restaurant-service` và `profile = prod`.
2. **Định vị Repository**: Server sử dụng cấu hình Git được khai báo trong `application.yml` (`uri: https://github.com/foodx/config-repo.git` và nhánh mặc định `main`).
3. **Đồng bộ hóa (Fetch/Pull)**: Config Server tiến hành clone repository về một thư mục tạm (temp directory) trên ổ cứng máy chủ (nếu chạy lần đầu), hoặc thực hiện thao tác `git pull` để đồng bộ hóa commit mới nhất trên nhánh `main`.
4. **Tìm kiếm file cấu hình**: Config Server tìm kiếm các file cấu hình tương ứng trong repo theo thứ tự ưu tiên:
    - `restaurant-service-prod.yml` (hoặc `.properties`)
    - `restaurant-service.yml`
    - `application-prod.yml`
    - `application.yml`
5. **Hợp nhất và Biên dịch cấu hình**: Nó đọc tất cả các file cấu hình tìm thấy, phân tích cú pháp và hợp nhất chúng lại thành một cấu trúc phân cấp (PropertySources). Các thuộc tính có độ ưu tiên cao hơn (ví dụ trong file cụ thể cho profile `prod`) sẽ override các thuộc tính chung.
6. **Phản hồi Client**: Config Server định dạng dữ liệu thành một cấu trúc JSON chuẩn của Spring Cloud Config và trả về mã trạng thái `200 OK` cho `restaurant-service`.

---

## 3. Cách thức kiểm thử độc lập (Independent Testing)

Bạn hoàn toàn có thể kiểm thử Config Server một cách độc lập mà không cần khởi chạy service client thật (`restaurant-service`) bằng các công cụ sau:

### Cách 1: Sử dụng Web Browser hoặc Command-line (cURL)
Do Config Server cung cấp endpoint dạng REST API tiêu chuẩn, bạn có thể gửi HTTP GET trực tiếp:
```bash
curl -X GET http://localhost:8888/restaurant-service/prod
```
**Kết quả mong đợi (JSON thành công):**
```json
{
  "name": "restaurant-service",
  "profiles": ["prod"],
  "label": "main",
  "version": "a1b2c3d4...",
  "propertySources": [
    {
      "name": "https://github.com/foodx/config-repo.git/restaurant-service-prod.yml",
      "source": {
        "spring.datasource.url": "jdbc:mysql://prod-db:3306/restaurant_db",
        "app.feature.discount": true
      }
    },
    {
      "name": "https://github.com/foodx/config-repo.git/restaurant-service.yml",
      "source": {
        "server.port": 8081
      }
    }
  ]
}
```

### Cách 2: Sử dụng Postman / Insomnia
- Tạo một request `GET` đến địa chỉ `http://localhost:8888/restaurant-service/prod`.
- Kiểm tra Response Status Code phải là `200 OK` và Response Body chứa các cấu hình mong muốn từ GitHub repo của bạn.