---
title: "Thoughts on CAPE"
date: 2025-10-3
categories: [misc]
tags: [cape]
---

# Opening

Cuối cùng cũng nhặt được cái "áo choàng" hehe, cho anh em nào thắc mắc về độ khó thì nếu như làm nhiều (rất nhiều) lab/box AD thì thật ra đề thi cũng không quá đáng sợ như mình nghĩ, tuy nhiên thì flag đầu vẫn tốn một ngày rưỡi :v 5 flag đầu sẽ khó hơn nhiều so với 5 flag sau, đấy là cảm nhận riêng của mình, còn phía dưới thì anh em mình nói về kĩ thuật một tí nhé.

# CERTIFIED ACTIVE DIRECTORY PENTESTING EXPERT

![alt text](/assets/img/posts/cape.png)

Như syllabus của chứng chỉ này đề ra thì sẽ không giới hạn về mặt kĩ thuật của các concept bao gồm:

- DACL attacks
- Trust attack
- NTLM relay attack
- Kerberos attack
- Windows Evasion
- ADCS attack
- Lateral movement
- Command and Control (C2) frameworks: về cái này thì sẽ có module dạy về sliver, cá nhân mình thấy sliver khá ổn trừ việc binary quá nặng :v

Chắc các bài review khác cũng sẽ có nội dung tương tự khi nói về nội dung học và thi của chứng chỉ này, trong post này mình sẽ đơn giản chỉ nói về trải nghiệm cách học và ôn để thi, mình hết 5 ngày (tính từ thời gian bắt đầu thi) để đủ 9/10 flag, vừa chạm điểm để pass bài thi. Còn câu cuối biết hướng làm nhưng bị hạn chế khá khó chịu nên exploit không thành công nên đành kết thúc bài thi với 90 điểm.

Mình bắt đầu học path của cert này từ cuối năm 2024, thời điểm vừa ra mắt luôn, trong thời gian đó mỗi tháng mình đóng 68$ để được 1000 cube, đồng nghĩa với việc học được 2 module mỗi tháng, vì thế nên cũng mất đủ lâu thì mới học xong được path để có thể thi. Tuy nhiên thì thời gian để ngồi ngẫm lại các module nhiều nên đem áp dụng vào các bài lab làm khá hiệu quả.

Trong thời gian học cho đến lúc thi, mỗi lần xong một module nào đó thì mình sẽ đọc thêm để thật sự hiểu thứ mà mình vừa đọc (yes i'm talking about you COM/DCOM, Kerberos, ADCS certificate mapping, SCCM, ...), background của mình thì cũng chỉ có CPTS, nên path này cho mình cực kì nhiều kiến thức về Windows Internal cũng như là AD. Mặc dù là đọc đến mấy cũng không hiểu hết được cho đến khi thực hành qua các lab của HTB và Vulnlab.

Nói qua một chút về thời điểm Vulnlab trước khi được mua lại bởi HTB, mình có cơ hội được nói chuyện với cũng khá nhiều red team lão làng bên EU và việc họ chia sẻ kiến thức và giải đáp sau mỗi bài lab cực kì tuyệt vời cho những ai không hiểu mình vừa bấm cái gì để exploit được xong box như mình trong thời gian mới bắt đầu :) Giờ thì HTB đã mua lại Vulnlab vài tháng rồi và các lab bên đấy sẽ được merge sang HTB nên mình cực kì recommend cho anh em nào muốn học/thi chứng chỉ này, hãy làm hết lab Windows/AD của Vulnlab (Linux cũng hay đừng bỏ qua nếu có thời gian) và phải thật hiểu câu chuyện đằng sau mỗi bài lab từ cách setup, lỗ hổng, cách exploit, tại sao nó lại hoạt động.

Cuối cùng thì do có kinh nghiệm thi CPTS từ trước, format đề thi sẽ là làm dần dần, tìm flag qua mỗi giai đoạn và submit và quan trọng là 10 ngày để hoàn thành đề thi lẫn report. Lời khuyên tốt nhất cho các anh em thi chứng chỉ của HTB là đừng cảm thấy bồn chồn, lo lắng khi kẹt, đó là lí do mà bài thi cho phép thi trong 10 ngày. Mỗi lần kẹt đừng cố quá để rồi tự bị áp lực, đứng dậy làm ngụm nước, ra ngoài đi dạo chạm cỏ thì sẽ tốt hơn nhiều cho đầu óc, vì chỉ sợ đầu mình không còn đủ tỉnh táo để nghĩ ra thêm kịch bản để thử thôi chứ sẽ luôn có hướng để giải quyết. Miễn là đầu mình luôn được refresh và có kinh nghiệm qua nhiều lần làm lab, ta sẽ luôn nghĩ ra được hướng để đi, vì thế đừng quá stress, keep going :D

# Ending

Mọi thắc mắc anh em có thể được giải đáp qua discord dwgth4i, stay strong và hãy luôn coi bài thi như là 20 ngày cộng thêm một lượt nghỉ giữa hiệp mỗi 10 ngày :D mình không farm chứng chỉ và cũng khuyên anh em không nên vậy, mà hãy chỉ coi nó như một minh chứng rằng mình thật sự đủ kĩ năng cho một việc gì đó thôi, mục tiêu sắp tới của mình khả năng sẽ là một cert gì đấy nặng hơn về evasion do đề thi CAPE cũng khá chill trong khâu evasion, và khả năng cao sẽ là CRTL (bản mới đang được Rastamouse làm lại) hoặc CRTM nếu như trong thời gian tiết kiệm tiền thấy không thích evasion nữa :v.