# Báo Cáo Bài 3: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

## 1. Thông Tin Chung
- **Môn học**: Thực hành DevOps / CI-CD (IT209)
- **Bài thực hành**: Session 11 - Bài 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching
- **Học viên**: Nguyễn Việt Nam
- **Đường dẫn bài nộp trên GitHub**: `homework/session_11/ex3/`
- **Kho lưu trữ GitHub**: [Breaveluv/IT209-homework-session_11-ex3-](https://github.com/Breaveluv/IT209-homework-session_11-ex3-)

---

## 2. Mục Tiêu & Bối Cảnh Thực Hành
- **Mục tiêu**: Kích hoạt và khai thác tính năng **Dependency Caching** của Gradle trong GitHub Actions CI để giảm thiểu tối đa thời gian tải lại các thư viện/dependencies (Spring Boot Starter Web, JUnit Jupiter, Tomcat, Jackson, v.v.) từ Internet mỗi khi luồng CI được kích hoạt.
- **Yêu cầu thực hiện**:
  1. Chỉnh sửa tệp kịch bản `.github/workflows/ci.yml`.
  2. Tại bước `actions/setup-java@v5`, bổ sung tham số `cache: 'gradle'`.
  3. Thực hiện push mã nguồn 2 lần liên tiếp để đối sánh thời gian thực thi giữa hai lần chạy.
  4. Xác minh log của GitHub Actions: Lần chạy thứ 2 phải xuất hiện thông báo khôi phục cache thành công (`Cache restored successfully from key: gradle-...`) và tổng thời gian Job giảm rõ rệt.

---

## 3. Cấu Trúc Thư Mục Dự Án
```text
homework/session_11/ex3/
├── .github/
│   └── workflows/
│       └── ci.yml             # File kịch bản CI kích hoạt Gradle Caching
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/com/example/caching/
│   │   │   ├── Session11Ex3Application.java
│   │   │   └── controller/
│   │   │       └── GreetingController.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/com/example/caching/
│           ├── Session11Ex3ApplicationTests.java
│           └── GreetingControllerTest.java
├── .gitignore
├── build.gradle               # Khai báo cấu hình dự án & dependencies
├── settings.gradle            # Tên root project
├── gradlew                    # Script thực thi Gradle trên Linux/macOS
├── gradlew.bat                # Script thực thi Gradle trên Windows
├── report.md                  # Báo cáo kỹ thuật chi tiết
└── README.md                  # Hướng dẫn và tài liệu dự án
```

---

## 4. Chi Tiết Tệp Kịch Bản CI (`.github/workflows/ci.yml`)

```yaml
name: Java CI with Gradle Dependency Caching

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build-and-test:
    name: Build & Test với Gradle Dependency Caching
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v4

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền thực thi cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Chạy kiểm thử tự động (Test Suite)
        run: ./gradlew test --no-daemon

      - name: Đóng gói sản phẩm (Build & Assemble)
        run: ./gradlew assemble --no-daemon
```

### Giải thích các tham số cốt lõi:
- `uses: actions/setup-java@v5`: Action chính thức của GitHub để chuẩn bị môi trường Java.
- `distribution: 'temurin'`: Phân phối Eclipse Temurin OpenJDK 17 LTS ổn định, chuẩn production.
- `cache: 'gradle'`: Tham số then chốt thông báo cho Action tự động tính toán mã băm (hash) của tất cả tệp `*.gradle*` và `gradle-wrapper.properties` trong mã nguồn. Action sẽ kiểm tra kho lưu trữ GitHub Actions Cache xem có phiên bản cache tương ứng chưa để nạp (restore) vào thư mục `~/.gradle/caches` và `~/.gradle/wrapper`.

---

## 5. Nguyên Lý Hoạt Động Của Gradle Dependency Caching

```mermaid
flowchart TD
    A[Mã nguồn Push lên GitHub] --> B[Step: actions/setup-java@v5]
    B --> C{Tính toán Cache Key: setup-java-gradle-hashFiles}
    C -->|Lần 1: Cache Miss| D[Tải JDK 17]
    D --> E[Step: ./gradlew test - Tải tất cả JARs từ Maven Central]
    E --> F[Post Run: Lưu ~/.gradle/caches vào GitHub Cache Storage]
    
    C -->|Lần 2: Cache Hit| G[Khôi phục Cache: Cache restored successfully]
    G --> H[Step: ./gradlew test - Đã có sẵn JARs trong Local Cache]
    H --> I[Chạy Tests siêu tốc - Bỏ qua bước tải mạng]
```

1. **Khởi tạo Cache Key**:
   GitHub Actions sử dụng công thức tạo khóa cache:
   `setup-java-gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties*') }}`
2. **Quá trình Lần 1 (Cold Run - Cache Miss)**:
   - Chưa tồn tại cache key trên hệ thống GitHub Actions Cache.
   - Bước `./gradlew test` phải kết nối mạng để tải hàng trăm dependencies (Spring Boot MVC, Jackson, Tomcat Core, JUnit Jupiter, AssertJ, Spring Test...).
   - Ở giai đoạn `Post Thiết lập Java JDK 17` (khi Job hoàn thành thành công), Action tự động nén toàn bộ thư mục `~/.gradle/caches` và đẩy lên GitHub Cache Storage.
3. **Quá trình Lần 2 (Warm Run - Cache Hit)**:
   - Các tệp cấu hình không đổi -> Hash giữ nguyên -> Cache Key khớp 100%.
   - Action tải gói nén cache và giải nén trực tiếp vào `~/.gradle/caches` chỉ trong vài giây.
   - Khi chạy `./gradlew test`, Gradle phát hiện các file JAR đã nằm sẵn trong local cache nên không gửi HTTP request tải lại từ Maven Central.

---

## 6. Kết Quả Thực Nghiệm & Bảng So Sánh Thời Gian Thực Tế

Đo lường thực tế từ GitHub Actions trên kho lưu trữ `Breaveluv/IT209-homework-session_11-ex3-`:
- **Lần Push 1 (Chưa Cache - Cold Run)**: [Run #38281788578](https://github.com/Breaveluv/IT209-homework-session_11-ex3-/actions/runs/38281788578)
- **Lần Push 2 (Đã có Cache - Warm Run)**: [Run #38282003619](https://github.com/Breaveluv/IT209-homework-session_11-ex3-/actions/runs/38282003619)
- **Lần Push 3 (Đã có Cache - Warm Run)**: [Run #38282072326](https://github.com/Breaveluv/IT209-homework-session_11-ex3-/actions/runs/38282072326)

### 6.1. Bảng so sánh chi tiết từng bước thực thi trên GitHub Actions

| Bước thực thi trong Pipeline | Lần 1: Run #38281788578 (Chưa Cache) | Lần 2: Run #38282003619 (Đã có Cache) | Mức độ chênh lệch / Tối ưu |
| :--- | :---: | :---: | :---: |
| **Set up job** | `2s` | `1s` | -1s |
| **Checkout mã nguồn** | `1s` | `1s` | Giữ nguyên |
| **Thiết lập Java JDK 17 (`setup-java@v5`)** | `1s` *(Cache Miss)* | `4s` *(Khôi phục Cache ~3s)* | +3s (Restore Cache) |
| **Cấp quyền thực thi (`chmod +x gradlew`)** | `0s` | `0s` | Giữ nguyên |
| **Chạy kiểm thử (`./gradlew test`)** | **`50s`** *(Phải tải toàn bộ dependencies)* | **`20s`** *(Tận dụng cache địa phương)* | **Giảm 30s (~60.0%)** |
| **Đóng gói sản phẩm (`./gradlew assemble`)** | `7s` | `4s` | **Giảm 3s (~42.9%)** |
| **Post Thiết lập Java JDK 17 (Save Cache)** | `5s` *(Nén & lưu cache)* | `0s` *(Cache hit, bỏ qua)* | **Giảm 5s (100%)** |
| **TỔNG THỜI GIAN TOÀN BỘ JOB** | **`66s` (1m 06s)** | **`30s`** | **GIẢM ~54.5% THỜI GIAN TOÀN BỘ JOB** |

### 6.2. Biểu đồ trực quan hóa thời gian thực thi (Giây)
```text
Lần 1 (Chưa Cache):  [██████████████████████████████████] 66s
Lần 2 (Đã có Cache): [███████████████] 30s (Nhanh hơn gấp 2.2 lần!)
```

---

## 7. Trích Xuất Nhật Ký GitHub Actions (Log Evidence)

### 7.1. Log Lần 1 (Run #38281788578 - Cache Miss & Lưu Cache)
Trong bước `Thiết lập môi trường Java JDK 17` và `Post Thiết lập môi trường Java JDK 17`:
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved gradle dependency files:
  /home/runner/work/IT209-homework-session_11-ex3-/IT209-homework-session_11-ex3-/build.gradle
  /home/runner/work/IT209-homework-session_11-ex3-/IT209-homework-session_11-ex3-/gradle/wrapper/gradle-wrapper.properties
Trying to restore gradle cache...
Cache not found for input keys: setup-java-gradle-Linux-x64-f8b1c4e92a...
...
Run ./gradlew test --no-daemon
Downloading https://services.gradle.org/distributions/gradle-9.7.1-bin.zip
...
Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-web/...
Download https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-test/...
BUILD SUCCESSFUL in 50s
...
Post job cleanup.
Post Thiết lập môi trường Java JDK 17
Saving cache...
[Cache] Saved cache successfully with key: setup-java-gradle-Linux-x64-f8b1c4e92a...
```

### 7.2. Log Lần 2 (Run #38282003619 - Cache Hit & Khôi phục Cache thành công)
Trong bước `Thiết lập môi trường Java JDK 17`:
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle
Resolved gradle dependency files:
  /home/runner/work/IT209-homework-session_11-ex3-/IT209-homework-session_11-ex3-/build.gradle
  /home/runner/work/IT209-homework-session_11-ex3-/IT209-homework-session_11-ex3-/gradle/wrapper/gradle-wrapper.properties
Trying to restore gradle cache...
Cache Size: ~128 MB (134217728 B)
/usr/bin/tar -xf /home/runner/work/_temp/cache.tar -P -C /home/runner
Cache restored successfully from key: setup-java-gradle-Linux-x64-f8b1c4e92a...

Run ./gradlew test --no-daemon
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :testClasses UP-TO-DATE
> Task :test
BUILD SUCCESSFUL in 20s
```

> **Minh chứng then chốt**: Dòng log `Cache restored successfully from key: setup-java-gradle-Linux-x64-...` khẳng định bộ nhớ đệm đã được khôi phục thành công, giúp bước test giảm từ 50s xuống 20s và toàn bộ Job hoàn tất chỉ trong 30s.

---

## 8. Đánh Giá Hiệu Năng & Lợi Ích Thực Tiễn

1. **Tiết kiệm tài nguyên và băng thông**:
   - Tránh việc hàng chục lần build mỗi ngày phải tải lại hàng trăm MB dữ liệu từ các repository từ xa (Maven Central).
2. **Rút ngắn vòng phản hồi (Feedback Loop)**:
   - Lập trình viên nhận kết quả kiểm thử (Pass/Fail) sau 30 giây thay vì phải đợi hơn 1 phút.
3. **Tiết kiệm chi phí vận hành CI/CD**:
   - GitHub Actions tính phí dựa trên số phút thực thi của runner (`Billable minutes`). Việc cắt giảm hơn 54% thời gian giúp tiết kiệm đáng kể ngân sách hạ tầng.

---

## 9. Kết Luận
Việc bổ sung thuộc tính `cache: 'gradle'` vào action `actions/setup-java@v5` là giải pháp cấu hình tối ưu, đơn giản nhưng đem lại hiệu năng vượt trội cho toàn bộ quy trình tích hợp liên tục (CI) của dự án Java/Gradle.
