# Checklist setup GitHub Actions cho Android Firebase App Distribution

## 1. Cấu trúc thư mục cần xác nhận

Project mẫu:

```text
DemoFirebase/
├── cer/
│   └── certificate.jks
├── app/
│   ├── gradlew
│   ├── settings.gradle.kts
│   └── app/
│       └── build.gradle.kts
└── .github/
    └── workflows/
        └── firebase-distribution.yml
```

Checklist:

- [ ] GitHub Actions mặc định chạy từ repository root.
- [ ] Xác định đúng thư mục Gradle root.
- [ ] Xác định đúng thư mục Android module.
- [ ] Xác định đúng vị trí `gradlew`.
- [ ] Xác định đúng vị trí file `build.gradle.kts`.
- [ ] Xác định đúng vị trí keystore.
- [ ] Không giả định mọi project đều có cấu trúc `app/app/`.

## 2. GitHub Actions workflow

- [ ] File nằm trong `.github/workflows/`.
- [ ] YAML dùng indentation 2 spaces.
- [ ] Không dùng tab.
- [ ] Có `workflow_dispatch` nếu chạy thủ công.
- [ ] Tên input không bị trùng.
- [ ] Input `release_notes` có `description` để làm hướng dẫn.
- [ ] Không thêm `default` cho `release_notes` nếu muốn ô nhập rỗng.
- [ ] Input group có thể dùng chuỗi alias phân cách bằng dấu phẩy.
- [ ] Có `actions/checkout@v5` trước khi dùng git hoặc đọc file.
- [ ] Có `actions/setup-java@v5`.
- [ ] Java version tương thích với Gradle và Android Gradle Plugin.

## 3. Gradle wrapper và working-directory

Nếu `gradlew` nằm trong thư mục `app/`:

```yaml
- name: Grant execute permission for gradlew
  working-directory: app
  run: chmod +x ./gradlew
```

```yaml
- name: Build Debug APK
  working-directory: app
  run: ./gradlew assembleDebug
```

Checklist:

- [ ] `working-directory` trỏ đến thư mục chứa `gradlew`.
- [ ] Không dùng `./app/gradlew` khi đã có `working-directory: app`.
- [ ] Không chạy `./gradlew` từ root nếu root không chứa wrapper.
- [ ] `gradlew` có quyền thực thi.
- [ ] Gradle root có `settings.gradle` hoặc `settings.gradle.kts`.
- [ ] Module được include trong settings Gradle.

## 4. Đường dẫn keystore `cer`

Nếu Gradle chạy trong `app/` và keystore nằm ở root/cer:

```bash
-Pandroid.injected.signing.store.file=$(pwd)/../cer/certificate.jks
```

Nếu Gradle chạy từ repository root:

```bash
-Pandroid.injected.signing.store.file=$(pwd)/cer/certificate.jks
```

Checklist:

- [ ] Kiểm tra file keystore có tồn tại.
- [ ] Đường dẫn được tính theo `working-directory` hiện tại.
- [ ] Không commit file `.jks` lên repository.
- [ ] Secret `KEYSTORE_PASSWORD` tồn tại.
- [ ] Secret `KEY_ALIAS` tồn tại.
- [ ] Secret `KEY_PASSWORD` tồn tại.
- [ ] Alias trong secret đúng với alias trong keystore.
- [ ] Không in password hoặc secret ra log.

Lệnh debug an toàn:

```bash
pwd
ls -la
ls -la ../cer
```

## 5. Build APK và đường dẫn output

Với project có Gradle root là `app/` và module là `app/`, output thường là:

```text
app/app/build/outputs/apk/debug/app-debug.apk
```

Trong step có `working-directory: app`, file nguồn thường là:

```bash
app/build/outputs/apk/debug/app-debug.apk
```

Khi Firebase Action chạy từ repository root, đường dẫn cần là:

```text
app/app/build/outputs/apk/debug/app-debug.apk
```

Checklist:

- [ ] Chạy đúng task, ví dụ `assembleDebug`.
- [ ] Kiểm tra APK thực sự được tạo.
- [ ] Đường dẫn trong shell và đường dẫn trong `with.file` không bị nhầm.
- [ ] Nếu đổi tên APK, cập nhật cả `APK_PATH`.
- [ ] Thêm bước kiểm tra output trước upload.

```yaml
- name: Check APK output
  run: ls -lh app/app/build/outputs/apk/debug/
```

## 6. Tạo tên APK duy nhất

Cách dùng timestamp:

```bash
BUILD_TAG="build-$(date -u +%Y%m%d-%H%M%S)"
```

```bash
mv app/build/outputs/apk/debug/app-debug.apk \
  app/build/outputs/apk/debug/app-debug-${BUILD_TAG}.apk

echo "APK_PATH=app/app/build/outputs/apk/debug/app-debug-${BUILD_TAG}.apk" \
  >> "$GITHUB_ENV"
```

Checklist:

- [ ] `BUILD_TAG` được tạo trước step build hoặc được truyền đúng vào step build.
- [ ] Tên file sau `mv` đúng với tên được upload.
- [ ] `APK_PATH` được ghi vào `$GITHUB_ENV`.
- [ ] `file` dùng `${{ env.APK_PATH }}`.
- [ ] Timestamp có cả ngày và giờ.
- [ ] Nếu nhiều workflow có thể chạy cùng lúc, cân nhắc thêm `${{ github.run_id }}`.

Lưu ý quan trọng:

- Đổi tên file chỉ tránh trùng tên hoặc trùng path trên filesystem.
- Đổi tên file không đảm bảo Firebase luôn tạo release mới.
- Firebase có thể nhận diện release dựa trên package, version và nội dung binary.
- Nếu cần binary khác thật sự nhưng giữ nguyên `versionCode` và `versionName`, phải thay đổi nội dung APK, ví dụ thêm build metadata vào resource hoặc BuildConfig.

## 7. Package name Android và Firebase

Trong `app/app/build.gradle.kts`:

```kotlin
android {
    namespace = "com.example.myapp"

    defaultConfig {
        applicationId = "com.example.myapp"
    }
}
```

Checklist:

- [ ] `namespace` đúng với code Android.
- [ ] `applicationId` đúng với Firebase Android App.
- [ ] Firebase App ID thuộc đúng project.
- [ ] Không có `applicationIdSuffix` làm thay đổi package debug ngoài ý muốn.
- [ ] Không upload package cũ lên Firebase App mới.
- [ ] Có thể kiểm tra package APK bằng `apkanalyzer` hoặc Android Studio.

Lỗi thường gặp:

```text
The APK package name does not match your Firebase app package name
```

## 8. Firebase App Distribution

Mẫu action:

```yaml
- name: Upload to Firebase App Distribution
  uses: wzieba/Firebase-Distribution-Github-Action@v1
  with:
    appId: ${{ secrets.FIREBASE_APP_ID }}
    serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
    groups: ${{ inputs.tester_group }}
    file: ${{ env.APK_PATH }}
    releaseNotesFile: ${{ env.RELEASE_NOTES_FILE }}
```

Checklist:

- [ ] `FIREBASE_APP_ID` là App ID, không phải Project ID.
- [ ] App ID thuộc đúng package Android.
- [ ] `FIREBASE_SERVICE_ACCOUNT` chứa JSON hợp lệ.
- [ ] Service account có quyền Firebase App Distribution.
- [ ] Group đã tồn tại trong Firebase.
- [ ] `groups` dùng group alias, không dùng display name.
- [ ] Nhiều alias phân cách bằng dấu phẩy.
- [ ] Không thêm khoảng trắng thừa trong alias.
- [ ] Nếu lỗi 404 ở bước distribute nhưng upload thành công, kiểm tra group alias trước.
- [ ] Có thể thử `testers` bằng email để tách lỗi group khỏi lỗi upload.

Kiểm tra group alias:

```bash
firebase appdistribution:groups:list --project <project-id>
```

Ví dụ cần phân biệt:

```text
Display name: Group 3
Alias: group-3
```

Giá trị `groups` phải dùng:

```text
group-3
```

không phải:

```text
Group 3
```

## 9. Release notes nhiều dòng

Nên tạo file text thay vì nhúng chuỗi nhiều dòng trực tiếp vào YAML:

```bash
export TZ=Asia/Tokyo
VERSION_NAME=$(grep -E 'versionName *= *"' app/app/build.gradle.kts \
  | head -n 1 \
  | sed -E 's/.*versionName *= *"([^"]+)".*/\1/')
VERSION_CODE=$(grep -E 'versionCode *= *[0-9]+' app/app/build.gradle.kts \
  | head -n 1 \
  | sed -E 's/.*versionCode *= *([0-9]+).*/\1/')
NOW=$(date '+%Y-%m-%d %H:%M')

{
  echo "STG ${VERSION_NAME}(${VERSION_CODE})"
  echo "${RELEASE_NOTE_INPUT}"
  echo "Distributed by ${TRIGGER_USER}"
  echo "At ${NOW} Japan time"
} > release-notes.txt

echo "RELEASE_NOTES_FILE=$GITHUB_WORKSPACE/release-notes.txt" >> "$GITHUB_ENV"
```

Format kết quả:

```text
STG 1.0(1)
Nội dung release note nhập trên web
Distributed by username
At 2026-08-27 18:30 Japan time
```

Checklist:

- [ ] `release_notes` không có default nếu muốn input rỗng.
- [ ] `description` có câu hướng dẫn cho người nhập.
- [ ] Dùng `releaseNotesFile` cho nội dung nhiều dòng.
- [ ] File release note được tạo trước bước upload.
- [ ] `RELEASE_NOTES_FILE` là đường dẫn tuyệt đối hoặc đúng path từ root.
- [ ] `github.actor` là username GitHub, không phải tên thật.
- [ ] Dùng `inputs.trigger_name` nếu cần nhập tên thật thủ công.
- [ ] Múi giờ trong `TZ` khớp với chữ hiển thị.
- [ ] `Asia/Tokyo` là giờ Nhật.
- [ ] `Asia/Ho_Chi_Minh` là giờ Việt Nam, không phải giờ Nhật.

## 10. Biến môi trường và expression

Trong shell:

```bash
${VARIABLE}
```

Trong GitHub Actions YAML:

```yaml
${{ env.VARIABLE }}
${{ inputs.input_name }}
${{ github.actor }}
${{ github.run_number }}
```

Checklist:

- [ ] Không dùng `${SHORT_SHA}` trực tiếp trong `with:`.
- [ ] Dùng `${{ env.SHORT_SHA }}` nếu biến nằm trong `GITHUB_ENV`.
- [ ] Dùng `${{ inputs.release_notes }}` để đọc workflow input.
- [ ] Biến cần dùng giữa các step phải ghi vào `$GITHUB_ENV`.
- [ ] Không nhầm biến shell với expression của GitHub Actions.
- [ ] Khi truyền input vào shell, cân nhắc dữ liệu có ký tự đặc biệt hoặc newline.

## 11. Gradle, plugin và thư viện

Checklist:

- [ ] Gradle wrapper version tương thích với Android Gradle Plugin.
- [ ] Android Gradle Plugin tương thích với Gradle version.
- [ ] Kotlin plugin tương thích với Kotlin compiler.
- [ ] `compileSdk` có sẵn trong runner/toolchain.
- [ ] `targetSdk` tương thích với Android Gradle Plugin.
- [ ] `pluginManagement.repositories` có `google()`.
- [ ] Có `mavenCentral()`.
- [ ] Có `gradlePluginPortal()` nếu plugin cần.
- [ ] Dependency trong `gradle/libs.versions.toml` tồn tại.
- [ ] Không copy nguyên version thư viện từ project khác nếu chưa kiểm tra.
- [ ] Không thêm plugin không cần thiết chỉ để sửa tạm lỗi CI.
- [ ] Kiểm tra `settings.gradle.kts` khi CI báo không resolve được plugin.

Lệnh kiểm tra local:

```bash
cd app
./gradlew tasks
./gradlew assembleDebug
```

## 12. Versioning

Firebase có thể nhận nhiều bản có cùng `versionCode` và `versionName`, nhưng cần nhớ:

- [ ] Google Play bắt buộc `versionCode` tăng.
- [ ] Đổi tên file APK không thay thế cho việc tăng `versionCode` trên Google Play.
- [ ] `versionName` chỉ là tên hiển thị.
- [ ] `versionCode` là số định danh build cho Google Play.
- [ ] Firebase và Google Play có quy tắc nhận diện release khác nhau.

## 13. Chẩn đoán lỗi theo log

### Lỗi không tìm thấy gradlew

```text
./gradlew: No such file or directory
```

Kiểm tra:

- [ ] Vị trí `gradlew`.
- [ ] `working-directory`.
- [ ] Lệnh có dùng đúng path không.

### Lỗi plugin

```text
Plugin ... was not found
```

Kiểm tra:

- [ ] `settings.gradle.kts`.
- [ ] `pluginManagement.repositories`.
- [ ] AGP, Gradle và Kotlin version.
- [ ] Plugin có thực sự cần không.

### Lỗi package

```text
APK package name does not match Firebase app package name
```

Kiểm tra:

- [ ] `applicationId`.
- [ ] `applicationIdSuffix`.
- [ ] `FIREBASE_APP_ID`.

### Lỗi file APK

```text
File does not exist
```

Kiểm tra:

- [ ] Task build thành công.
- [ ] Output path.
- [ ] Tên file sau khi `mv`.
- [ ] Path được tính từ root hay từ `working-directory`.

### Lỗi Firebase 404 khi distribute

```text
failed to distribute to testers/groups: HTTP Error: 404
```

Kiểm tra:

- [ ] Group alias có đúng không.
- [ ] Group thuộc đúng Firebase project không.
- [ ] App ID có đúng không.
- [ ] Service account có quyền không.
- [ ] Thử phân phối bằng email tester để khoanh vùng.

## 14. Bộ lệnh kiểm tra nhanh trước khi push

```bash
cd /path/to/project

# Kiểm tra cấu trúc
pwd
find . -maxdepth 3 -type f \( -name gradlew -o -name settings.gradle.kts -o -name build.gradle.kts -o -name '*.jks' \)

# Kiểm tra wrapper và build local
cd app
chmod +x ./gradlew
./gradlew assembleDebug

# Kiểm tra APK
find . -path '*build/outputs/apk/debug/*.apk' -type f -print

# Quay lại root và kiểm tra YAML
cd ..
ruby -e "require 'yaml'; YAML.load_file('.github/workflows/firebase-distribution.yml'); puts 'YAML parsed'"
git diff --check
```

## 15. Checklist tối thiểu cho người mới

```text
[ ] Đúng cấu trúc project
[ ] Đúng vị trí gradlew
[ ] Đúng working-directory
[ ] chmod +x gradlew
[ ] Đúng đường dẫn cer/certificate.jks
[ ] Đủ secret signing
[ ] Đúng applicationId
[ ] Đúng Firebase App ID
[ ] Đúng service account
[ ] Đúng group alias
[ ] APK được tạo thành công
[ ] APK path trỏ đúng file
[ ] Release notes nhiều dòng dùng releaseNotesFile
[ ] Biến giữa các step dùng GITHUB_ENV
[ ] actions/checkout@v5
[ ] actions/setup-java@v5
[ ] YAML indentation đúng
[ ] Chạy build local trước khi push
[ ] Kiểm tra log gốc khi lỗi
```
