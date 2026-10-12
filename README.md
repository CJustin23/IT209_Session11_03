# Session 11 — Bài 3: Gradle Dependency Caching trong GitHub Actions

Dự án Spring Boot 3.3.4 (Java 17, Gradle wrapper) dùng để đo hiệu quả của `cache: 'gradle'`
trong `actions/setup-java@v5`.

Học viên: **Nguyễn Trường An** — lớp HN-KS24-CNTT3 — môn IT209.
Repo: https://github.com/CJustin23/IT209_Session11_03

## Chạy local

```bash
./gradlew test        # chạy unit test
./gradlew bootJar     # đóng gói jar vào build/libs/
./gradlew assemble
```

## CI

`.github/workflows/ci.yml` chạy trên mỗi push vào `main` / `master`: checkout → setup JDK 17 (temurin,
bật `cache: 'gradle'`) → `chmod +x gradlew` → `./gradlew test` → `./gradlew assemble`.

Báo cáo so sánh thời gian cold run / warm run và trích log: xem [report.md](report.md).
