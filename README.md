# UnCrackable Level 1

Link để tải file apk: [UnCrackable-Level1.apk](https://mas.owasp.org/crackmes/Android#android-uncrackable-l1)

Đây là một challenge của OWASP được thiết kế để bypass các cơ chế bảo mật trên ứng dụng Android. Hôm nay mình sẽ

## Chạy thử ứng dụng và phân tích sơ bộ

![ảnh](https://github.com/user-attachments/assets/e78ffd98-7a5e-45d7-8f11-86d05c99964e)

Sau khi install file apk vào máy ảo LDPlayer thì mình sẽ tiến hành mở app và xem hành vi của app trông như thế nào.

![ảnh](https://github.com/user-attachments/assets/a6373bca-c1aa-48bf-bbaf-9c9960571c99)

Popup báo là "Root Detected", khi click "OK" thì chương trình sẽ đóng luôn.

=> Vậy là bước đầu tiên của chúng ta sẽ là bypass được phần check root trước.

Cấu trúc khi decompile bằng bytecode viewer:
```
sg.vantagepoint/
├── a/
│   ├── a.class
│   ├── b.class
│   ├── c.class
└── uncrackable1/
    ├── a.class
    ├── MainActivity$1.class
    ├── MainActivity$2.class
    └── MainActivity.class
```
![ảnh](https://github.com/user-attachments/assets/7c62d436-7967-44ed-8da6-2ddb6da0673c)

Vì source code đã được obfuscation nên tên file và tên hàm có thể gây nhầm lẫn và khó khăn trong quá trình phân tích.

![ảnh](https://github.com/user-attachments/assets/80ad04d5-9796-4a25-bd6e-748bcf73ee95)

```if (c.a() || c.b() || c.c())```
Từ line 25 sẽ kiểm trả thiết bị có root hay không và class c là nơi chứa các method để check root, nếu một trong ba hàm `a()`, `b()`, hoặc `c()` của class c trả về `true` → app hiển thị popup "Root detected!" rồi dừng chương trình.

Mình bắt đầu lần theo dấu vết qua class `sg.vantagepoint.a.c` để xem cụ thể từng hàm đang làm gì.

![ảnh](https://github.com/user-attachments/assets/84592ddf-1ee2-44d8-a1a7-1dba7eee115f)

### Hàm `a()` tìm file `su` trong PATH

```
public static boolean a() {
    for (String str : System.getenv("PATH").split(":")) {
        if (new File(str, "su").exists()) {
            return true;
        }
    }
    return false;
}
```

Nếu ai đã từng root máy hoặc dùng Linux, chắc bạn biết su chính là binary dùng để thực thi lệnh với quyền root. Trên máy đã root, su thường nằm ở các path như /system/xbin, /system/bin,...
Chỉ cần tìm thấy file `su` trong bất kỳ thư mục nào trong PATH → return true. Đây là dấu hiệu đầu tiên mà ứng dụng dựa vào để phát hiện root.

```
public static boolean b() {
    String str = Build.TAGS;
    return str != null && str.contains("test-keys");
}
```
### Hàm `b()` dựa vào `Build.TAGS`
Ở đây, app truy cập vào trường Build.TAGS một thông tin build-in sẵn của Android để xem ROM đang chạy trên thiết bị có dấu hiệu là bản chưa chính thức hay không.
Cụ thể: 
* Nếu bạn dùng ROM chính thức từ nhà sản xuất → thường là "release-keys"
* Còn nếu bạn build thủ công từ AOSP hoặc cài ROM cook → khả năng cao sẽ có "test-keys"

Vì vậy, nếu `Build.TAGS` chứa "test-keys" → app cho rằng đây là một môi trường có khả năng đã bị root.

### Hàm `c()` – Tìm các dấu vết root phổ biến trong filesystem

```
public static boolean c() {
    for (String str : new String[]{
        "/system/app/Superuser.apk",
        "/system/xbin/daemonsu",
        "/system/etc/init.d/99SuperSUDaemon",
        "/system/bin/.ext/.su",
        "/system/etc/.has_su_daemon",
        "/system/etc/.installed_su_daemon",
        "/dev/com.koushikdutta.superuser.daemon/"
    }) {
        if (new File(str).exists()) {
            return true;
        }
    }
    return false;
}
```

Đây là cú chốt hạ. Hàm `c()` duyệt qua một danh sách những đường dẫn cố định, nơi mà các app root, daemon, hoặc tool như SuperSU, Magisk thường để lại dấu vết.
Nếu thiết bị có tồn tại bất kỳ file hoặc thư mục nào trong list trên, app sẽ coi đó là dấu hiệu của root và... boom, bạn bị văng ra.

Sau khi đọc kỹ ba hàm trên, mình có thể tóm gọn như sau:

Hàm `a()` Kiểm tra file su trong các thư mục hệ thống, hàm `b()` dựa vào `Build.TAGS` để phát hiện ROM không chính thức, hàm `c()` tìm dấu vết của các app root trong filesystem. Chỉ cần một trong ba điều kiện trên đúng → chúng ta sẽ thấy pop-up "Root detected" rồi app sẽ exit luôn (thường là gọi System.exit(0) trong đoạn xử lý alert).

![ảnh](https://github.com/user-attachments/assets/0c3e18e4-f279-4ec8-a99f-7e238e054bb5)

## Vậy làm sao để có thể bypass check root??

### Cách 1: patching apk
Đầu tiên mình sẽ decompile file apk bằng apktool: `apktool d "D:\lab mobile\UnCrackable-Level1.apk"`

![ảnh](https://github.com/user-attachments/assets/ce5e760b-afa6-4324-bfbb-1ff1f74d9582)

![ảnh](https://github.com/user-attachments/assets/9ed8715d-fcf5-4b3b-a02a-a830d2fc2482)

Khi có được source code smali, chúng ta sẽ tiến hành sửa lại code của 3 hàm `c.a()`, `c.b()` và `c.c()` trong smali/sg/vantagepoint/a
Chỉ cần 3 hàm này luôn return về false là ngon. Kiểu `const/4 v0`, `0x1` → c`onst/4 v0`, `0x0`

Như mình đã đề cập trước đó, Class c chính là mấu chốt để chúng ta có thể bypass check root, nên sẽ focus vào đây để patching.






