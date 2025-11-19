# Cấu trúc Folder
Cách in cấu trúc `/root/` 

Hiển thị dạng “cây” dễ nhìn:
```bash
sudo apt install tree  # Cài module hỗ trợ
tree /path/to/folder
```

Một số thư mục quan trọng và ý nghĩa

|Thư mục|Mục đích chính|
|---|---|
|`/bin`|Chứa các nhị phân (binaries) cơ bản cần thiết cho mọi người dùng và cho hệ thống khởi động ở chế độ một người dùng. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/sbin`|Nhị phân dành cho quản trị hệ thống (system binaries) mà người dùng không thường dùng. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/etc`|Cấu hình hệ thống toàn cục (configuration files) cho máy chủ/host này. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/usr`|Thư mục phụ chứa dữ liệu người dùng, ứng dụng đa người dùng, phần lớn phần mềm. ([GeeksforGeeks](https://www.geeksforgeeks.org/linux-unix/linux-file-hierarchy-structure/?utm_source=chatgpt.com "Linux File Hierarchy Structure"))|
|`/usr/local`|Dùng để cài phần mềm riêng cho máy hiện tại, không phải phần mềm hệ thống mặc định. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/var`|Dữ liệu biến đổi trong quá trình hệ thống chạy: log, spool, cache, mail. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/home`|Thư mục chứa thư mục cá nhân (home directories) của người dùng thường. ([GeeksforGeeks](https://www.geeksforgeeks.org/linux-unix/linux-file-hierarchy-structure/?utm_source=chatgpt.com "Linux File Hierarchy Structure"))|
|`/tmp`|File tạm thời (temporary files) có thể bị xóa khi khởi động lại. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/dev`|Thiết bị (devices) như ổ đĩa, cổng serial, thiết bị đặc biệt xuất hiện như file. ([Wikipedia](https://es.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/proc`|Hệ thống file ảo (pseudo-filesystem) cung cấp thông tin về kernel và tiến trình đang chạy. ([Wikipedia](https://en.wikipedia.org/wiki/Procfs?utm_source=chatgpt.com "Procfs"))|
|`/sys`|Thư mục ảo khác, chứa thông tin thiết bị, driver, tính năng kernel. ([Wikipedia](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard?utm_source=chatgpt.com "Filesystem Hierarchy Standard"))|
|`/boot`|Tệp tin khởi động: kernel, initramfs, bootloader. ([GeeksforGeeks](https://www.geeksforgeeks.org/linux-unix/linux-file-hierarchy-structure/?utm_source=chatgpt.com "Linux File Hierarchy Structure"))|

Cách để hiển thị trên VSCode:
Open folder -> `/`

Chuẩn bị các thư viện để cài



