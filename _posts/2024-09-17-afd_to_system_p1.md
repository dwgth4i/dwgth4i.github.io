---
title: "Arbitrary file delete to SYSTEM (Part 1)"
date: 2024-09-17
categories: [research]
tags: [Windows]
---

# Mở đầu
Như title thì hôm nay mình sẽ nói về việc nếu như ta có thể xóa file dưới quyền NT/SYSTEM thì impact cao nhất là gì? Có thể ta sẽ nghĩ đến đơn giản chỉ là DOS thôi nhưng mà với research từ năm 2022 của Abdelhamid Naceri (halov) thì không, anh này đã có thể lợi dụng việc đó và sử dụng để leo lên được quyền SYSTEM của Windows machine, nhưng mà trước khi đi tiếp thì ta sẽ phải hiểu 1 chút về Windows Installer service.
# Windows Installer
Là 1 service ở trong Windows phụ trách việc cài đặt ứng dụng, chắc hẳn các bạn từng thấy file .msi trên máy tính của mình rồi. Tưởng tượng nó sẽ giống như file requirements.txt khi bạn cài 1 thứ gì đó với Python, .msi có thể coi như 1 file database định nghĩa cho những thay đổi, những folder cần tạo ra, những registry key cần sửa, file cần nhét vào, ...

Đây sẽ là bước quan trọng, vì cần bảo đảm hệ thống không cần gặp vấn đề gì trục trặc khi cài đặt không thể được hoàn tất, Windows Installer sẽ có 1 cơ chế để giúp rollback về lúc trước đó. Mỗi lần Windows Installer thay đổi gì liên quan đến hệ thống, nó sẽ ghi lại record chỉnh sửa đó, và mỗi lần nó ghi đè 1 file nào đó đã tồn tại rồi thành phiên bản mới hơn, thì file phiên bản cũ sẽ được lưu lại. Đề phòng nếu cần rollback, Windows Installer sẽ cần dùng đến các record kia để khôi phục lại hệ thống như ban đầu. Và thường thì những file này, trong những trường hợp đơn giản nhất sẽ là ở C:\Config.msi

Trong lúc cài đặt, Windows Installer sẽ tạo ra folder C:\Config.msi và đưa các file chứa thông tin về việc rollback vào đó. Như đã nói ở trên, mỗi khi Windows Installer cài đặt gì đó, nó sẽ ghi lại những thay đổi đó và ghi 1 file .rbs (rollback script) vào bên trong C:\Config.msi. Hơn nữa, nếu Windows Installer ghi đè 1 file cũ với 1 file mới, thì file cũ đó cũng sẽ đi vào C:\Config.msi với extension là .rbf (rollback file). Cuối cùng nếu việc cài đặt gặp trục trặc, nó sẽ đọc các file .rbs và .rbf và dùng chúng để rollback về trạng thái ban đầu trước khi cài đặt.

Đương nhiên là cơ chế này sẽ được bảo vệ bằng cách đặt ra 1 cái DACL rất chặt cho folder C:\Config.msi. Hơn nữa, để nhận biết được giữa C:\Config.msi legit và C:\Config.msi malicious thì Windows Installer có sử dụng 1 registry key được đặt bắt đầu từ HKLM. Full key name là HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Installer, mỗi lần cần tạo ra 1 folder C:\Config.msi, Windows Installer sẽ tạo ra 1 value tương ứng với key trên kia có tên là C:\Config.msi, và việc value này tồn tại sẽ chỉ định rằng C:\Config.msi là legit, đây sẽ là C:\Config.msi với 1 DACL rất chặt và được tin tưởng. Khi xóa folder này, Windows Installer sẽ xóa luôn cả value registry tương ứng, nên nếu attacker có tạo ra 1 folder cùng tên thì cũng sẽ không được trusted bởi Windows Installer bởi vì không có giá trị tương ứng registry tương ứng với key bên trên.

Đến đây thì nếu mà attacker có quyền xóa file bất kì thì sao??? Attacker có thể xóa C:\Config.msi ban đầu và tạo ra 1 C:\Config.msi với DACL yếu hơn. Nhưng mà về việc check registry value thì sao? Đơn giản việc xóa là từ phía của attacker, không phải là từ Install service nên registry value sẽ vẫn còn ở đó và vì thế Windows Installer sẽ trust folder mới được tạo ra này. Vì attacker có full quyền kiểm soát với folder C:\Config.msi nên attacker sẽ có thể đặt file .rbs và .rbf bất kì vào đó. Và từ đây attacker có thể tận dụng installer và làm bất kì những gì attacker muốn với hệ thống dưới quyền SYSTEM khi rollback.

Và nhớ rằng điều kiện cần duy nhất là có thể xóa 1 folder rỗng, hoặc có thể di chuyển chỗ khác hay đổi tên tùy.

# From Arbitrary Folder Delete/Move/Rename to SYSTEM (Privilege Escalation)
Ở thời điểm research này được đăng lên lần đầu vào 2022 thì kĩ thuật để exploit khá là tricky vì phải timing việc xóa rồi tạo nên là exploit vẫn chưa quá khả thi do là việc attacker thường sẽ khó mà có thể trigger được việc xóa file bất kì.

Đến tận tháng 06/2023 thì Abdelhamid Naceri mới có 1 bài update cho research này. Là khi uninstall 1 file, sẽ không đơn thuần là xóa file vì như thế sẽ rất khó để rollback nếu có lỗi xảy ra, vì vậy file đó sẽ được đặt vào C:\Config.msi với 1 cái tên random gì đó với extension .rbf. Và khi rollback, installer service sẽ chuyển file này về vị trí ban đầu và tên ban đầu. Do file này sẽ giữ nguyên DACL từ đầu đến cuối nên attacker có phần nào sẽ làm cho file này mở và ngăn việc xóa file ở cuối quá trình uninstall. Và cũng đồng nghĩa với việc installer service sẽ không xóa được folder C:\Config.msi, bởi vì đây không phải là 1 folder rỗng. Thay vào đó, installer service sẽ bị terminate mà không xóa file C:\Config.msi và cũng đồng nghĩa với việc giữ nguyên registry value của folder. Và vì thế C:\Config.msi vẫn là folder có 1 DACL chặt. Bây giờ, do là installer đã chạy xong nên attacker sẽ thoải mái trigger việc arbitrary folder delete và thay thế bằng folder C:\Config.msi tự tạo ra và có DACL yếu mà attacker có full quyền kiểm soát. Đương nhiên là khi Windows Installer chạy lại 1 lần nữa thì folder C:\Config.msi sẽ được trusted do registry value vẫn ở đấy.

Đến đấy, exploit sẽ có 2 stage:
- Stage 1: Exploit chạy thành công trigger uninstall mà không gặp vấn đề gì, và C:\Config.msi không bị xóa.
- Stage 2: Attacker xóa C:\Config.msi và tạo lại folder C:\Config.msi mới. Sau đó install gì đó và rollback, từ đó attacker có thể làm cho hệ thống làm những gì anh ta muốn từ những file .rbs, .rbf từ folder C:\Config.msi mà anh ta kiểm soát. 

# How to delete as SYSTEM ???
Abdelhamid Naceri có tìm ra được 1 bug khác về User Profile cho phép xóa file dưới quyền SYSTEM còn cụ thể như nào thì đó sẽ lại là 1 bài phân tích dài khác 🙉

# Conclusion
Đây đáng lẽ sẽ là 1 bài viết rất dài nếu như thật sự đầy đủ nên do hơi lười nên sẽ tạm dừng ở đây đã, maybe sẽ có part 2 cho chain exploit thú vị này :D Phía dưới sẽ là các bài reference để mọi người có thể tham khảo.

Ref:

(https://www.zerodayinitiative.com/blog/2022/3/16/abusing-arbitrary-file-deletes-to-escalate-privilege-and-other-great-tricks)
(https://cloud.google.com/blog/topics/threat-intelligence/arbitrary-file-deletion-vulnerabilities/)
(https://mantodeasecurity.de/en/2024/05/cve-2024-27460-plantronics-hub-lpe/) (CVE này đến từ idol xct của mình)
