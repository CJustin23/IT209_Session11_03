# Báo cáo Bài 3 — Tối ưu tốc độ CI bằng Gradle Dependency Caching

## 1. Thông tin chung

| Mục | Giá trị |
| :--- | :--- |
| Học viên | **Nguyễn Trường An** |
| Lớp | HN-KS24-CNTT3 |
| Môn / Session | IT209 — Session 11, Bài 3 |
| Repository | https://github.com/CJustin23/IT209_Session11_03 |
| Đường dẫn nộp bài | `homework/session_11/ex3/` |
| Workflow | `.github/workflows/ci.yml` — `Java CI with Gradle Dependency Caching` |

## 2. Thay đổi so với Bài 2

Chỉ thêm một tham số `cache: 'gradle'` vào bước `actions/setup-java@v5`:

```yaml
      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'   # <- nội dung của Bài 3
```

Khi có `cache: 'gradle'`, action tự tính hash của `*.gradle*` và `gradle-wrapper.properties`, dựng cache key
`setup-java-<runner.os>-gradle-<hash>` rồi restore/save `~/.gradle/caches` (dependency) và `~/.gradle/wrapper` (bản phân phối Gradle).

## 3. Cách đo

Push 2 lần liên tiếp lên nhánh `main` (commit không làm thay đổi file cấu hình Gradle để cache key giữ nguyên):

| Lần | Commit | Run | Trạng thái cache |
| :--- | :--- | :--- | :--- |
| 1 | `3669a2e` — *Enable Gradle dependency caching in setup-java to speed up CI* | [Run #1 / 38290248113](https://github.com/CJustin23/IT209_Session11_03/actions/runs/38290248113) | **Cache miss** (cold) |
| 2 | `9d1618a` — *Trigger second CI run to measure Gradle cache hit* | [Run #2 / 38290483666](https://github.com/CJustin23/IT209_Session11_03/actions/runs/38290483666) | **Cache hit** (warm) |

Số liệu dưới đây lấy trực tiếp từ GitHub Actions Jobs API (`/actions/runs/{id}/jobs`, timestamp `started_at`/`completed_at` của từng step) và file log tải về từ `/actions/runs/{id}/logs`.

## 4. Kết quả so sánh thời gian

| Bước trong Job | Run #1 (chưa cache) | Run #2 (đã cache) | Chênh lệch |
| :--- | :---: | :---: | :---: |
| Set up job | 1s | 2s | +1s |
| Checkout mã nguồn | 1s | 2s | +1s |
| Thiết lập Java JDK 17 (`setup-java@v5`) | 0s — *cache not found* | **3s** — *restore cache* | +3s |
| Cấp quyền `chmod +x gradlew` | 0s | 0s | — |
| `./gradlew test --no-daemon` | **35s** (`BUILD SUCCESSFUL in 33s`) | **12s** (`BUILD SUCCESSFUL in 11s`) | **−23s (−66%)** |
| `./gradlew assemble --no-daemon` | 6s (`in 6s`) | 5s (`in 5s`) | −1s |
| Post Thiết lập Java JDK 17 (save cache) | 5s — *nén & lưu cache* | 0s — *bỏ qua vì đã hit* | −5s |
| Post Checkout + Complete job | 2s | 1s | −1s |
| **Tổng thời gian thực tế của run** | **58s** (18:20:30 → 18:21:28 UTC) | **31s** (18:22:37 → 18:23:08 UTC) | **−27s (−47%)** |

```text
Run #1 (cold):  [██████████████████████████████████████████████████] 58s
Run #2 (warm):  [████████████████████████████] 31s   -> nhanh hơn ~1.9 lần
```

## 5. Bằng chứng trong log

### 5.1. Run #1 — cache miss, phải tải Gradle distribution, cuối job lưu cache

Bước *Thiết lập môi trường Java JDK 17*:

```text
   with:
     java-version: 17
     distribution: temurin
     cache: gradle
 Resolved Java 17.0.20+1 from tool-cache
 gradle cache is not found
```

Bước *Chạy kiểm thử tự động (Test Suite)*:

```text
 Fetching distribution.
 Downloading https://services.gradle.org/distributions/gradle-9.7.1-bin.zip
 ..............10%..............20%...............30%..............40%...............50%
 ..............60%...............70%..............80%..............90%...............100%
 BUILD SUCCESSFUL in 33s
 4 actionable tasks: 4 executed
```

Bước *Post Thiết lập môi trường Java JDK 17*:

```text
 Cache saved with the key: setup-java-Linux-x64-gradle-b4789e9c26fd17595610fbdf9bd4a9126c151e92d31afa44c2820d94e6b173ab
```

### 5.2. Run #2 — cache hit, khôi phục thành công

Bước *Thiết lập môi trường Java JDK 17*:

```text
 Cache hit for: setup-java-Linux-x64-gradle-b4789e9c26fd17595610fbdf9bd4a9126c151e92d31afa44c2820d94e6b173ab
 Cache Size: ~45 MB (47502803 B)
 [command]/usr/bin/tar -xf /home/runner/work/_temp/6cfe53d9-.../cache.tzst -P -C /home/runner/work/IT209_Session11_03/IT209_Session11_03 --use-compress-program unzstd
 Cache restored successfully
 Cache restored from key: setup-java-Linux-x64-gradle-b4789e9c26fd17595610fbdf9bd4a9126c151e92d31afa44c2820d94e6b173ab
 Cache hit for: setup-java-Linux-x64-gradle-wrapper-cb10f63b6e80d04c47ba63d0097b1be1d92513baebbf33fb9b91e65214965f4c
 Cache Size: ~144 MB (150684709 B)
 gradle-wrapper cache restored from key: setup-java-Linux-x64-gradle-wrapper-cb10f63b6e80d04c47ba63d0097b1be1d92513baebbf33fb9b91e65214965f4c
```

Bước *Chạy kiểm thử tự động (Test Suite)* — **không còn dòng `Downloading ... gradle-9.7.1-bin.zip`**, vào thẳng task:

```text
 > Task :compileJava
 > Task :testClasses
 > Task :test
 BUILD SUCCESSFUL in 11s
 4 actionable tasks: 4 executed
```

Bước *Post Thiết lập môi trường Java JDK 17*: 0s — cache đã hit nên action không lưu lại.

## 6. Phân tích

- Hai cache được phục hồi ở Run #2: `~/.gradle/caches` (~45 MB — dependency/module metadata) và `~/.gradle/wrapper` (~144 MB — bản phân phối Gradle 9.7.1). Nhờ đó bước test giảm từ 33s xuống 11s.
- Phần lớn khoản tiết kiệm **nhìn thấy được trong log** của dự án này đến từ việc không phải tải lại Gradle distribution; các lần tải dependency lẻ không in thành dòng riêng trong log Run #1 (Gradle 9.7 gộp output tải artifact), nên không kê khai riêng con số này.
- Bước `setup-java` chạy lâu hơn 3s (0s → 3s) và `Post` giảm 5s → 0s: tổng chi phí quản lý cache gần như hoà vốn, phần lãi nằm ở bước build/test.
- Cache key gắn với hash của `build.gradle`, `settings.gradle`, `gradle-wrapper.properties`. Thêm/nâng cấp dependency hoặc đổi phiên bản Gradle sẽ làm key đổi → lần chạy đầu tiên sau đó lại là cache miss, các lần tiếp theo hit trở lại.
- GitHub Actions cache có giới hạn 10 GB/repo và tự xoá cache không dùng trong 7 ngày.

## 7. Kết luận

Chỉ với một dòng `cache: 'gradle'` trong `actions/setup-java@v5`, tổng thời gian CI của dự án giảm từ **58s xuống 31s (−47%)**, riêng bước `./gradlew test` giảm **33s → 11s (−67%)**. Log Run #2 xuất hiện đầy đủ `Cache restored successfully` / `Cache restored from key: setup-java-Linux-x64-gradle-...` đúng yêu cầu kiểm tra của bài.

## 8. Minh chứng kèm theo

- `.github/workflows/ci.yml` (chứa `cache: 'gradle'`)
- Ảnh chụp trang Summary của Run #2 (thấy `Cache restored successfully` và thời gian job) — xem thư mục `images/`
