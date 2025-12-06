Làm cho UI code bởi AI trông không bị AI quá và cách mình sử dụng AI để code UI hiệu quả, đồng bộ mà vẫn có điểm riêng.

  

Xin chào anh em, dạo này mình làm UI cho mấy trang kiểu lehuythai[.]com hay vobabolt[.]fun thì có một số bạn hay hỏi mình làm sao code được UI đẹp, đồng bộ và để trông không bị AI quá, thì mình có nghĩ lại cách mình làm UI cả khi dùng AI và cả khi không dùng AI để lên bài này, mong rằng sẽ có gì đó hữu ích tới anh em.

  

Bài này sẽ có 2 phần:

- Làm UI code bởi AI trông không bị AI quá

- Cách mình sử dụng AI để code UI hiệu quả, đồng bộ mà vẫn có điểm riêng.

  

Oke bắt đầu

  

I. Làm UI code bởi AI trông không bị AI quá

  

Có nhiều UI anh em nhìn qua một cái là biết khả năng cao code với AI rồi đúng không ạ? Nó có gì đó rất là chung chung và cảm giác không thoải mái, kiểu nó bị không có điểm riêng ấy. Thì anh em có thể hạn chế 1 số thứ sau để nó mới lạ và đẹp hơn nhé:

  

1. Box-shadow

Gần như cấm tuyệt đối => Thay bằng border nhạt. AI nó dùng rất nhiều box shadow, mà thật sự trong UI mình thấy box shadow là trend cũ rồi giờ người ta cũng không hay dùng nữa.

  

2. Gradient → giảm tối đa

Chỉ dùng Gradient khi thật cần (kiểu như button hover thôi), cứ nhìn đâu cũng gradient thì quá chán, rối và kiểu hấp diêm con mắt vậy.

  

3. Rounded/Border radius quá đà

Cái này mình cũng thích dùng cơ mà nếu UI đang trông bị AI quá thì nên hạn chế. Và nên tuân thủ Rounded cha lớn hơn Rounded con để trông hài hoà, thay vì cứ bằng nhau hết

  

4. Font default

AI thường đa số dụng Font Family mặc định trông cứ na ná nhau như vậy (biết là nó sẽ phù hợp với đa số mọi người nhưng quá nhàm) => Sử dụng 1 vài cái font khác, mới lạ hơn và vẫn đẹp. Ví dụ Montserrat huyền thoại hay Be Vietnam Pro (mình đang dùng)

  

5. Glassmorphism / blur background

Nên hạn chế mấy cái này (mình cũng thích nhưng nếu UI bị trông AI quá thì nên hạn chế)

  

6. Màu xanh-tím

AI toàn gen ra màu này (chuẩn chưa ạ?) => nên Hạn chế. Mấy màu này mình cực thích, toàn chủ động dùng, mà không hiểu sao AI nó toàn gen ra màu này, đụng hàng quãi :((

  

7. Icon Emoji

AI nó hay dùng icon emojy để làm UI trông rất không chuyên nghiệp, không thể đổi style khi hover, animation => Sử dụng thư viện Icon hoặc svg thay cho các Emoji (ví dụ Font Awesome, Lucide Icon, ...)

  

8. Animation lố, loạn lạc

Nhiều lúc chúng ta cần nó trông chuyên nghiệp hơn nên cần đẩy Animation vào, thì có thể thằng AI nó sẽ làm Animation rất lố và không đồng bộ => Yêu cầu làm Animation thật chuyên nghiệp, đồng bộ và mượt mà.

  

Oke cơ bản là thế, anh em còn gì để lại comment cho mọi người né nhé ạ.

  

II. Cách mình sử dụng AI để code UI hiệu quả, đồng bộ mà vẫn có điểm riêng.

  

Cái này có vẻ anh em dev quan tâm, làm sao làm việc với AI mà hiệu quả (nhanh, đẹp), mà vẫn đồng bộ (trông nó hoà hợp, thống nhất, làm được lâu dài) mà vẫn có điểm riêng (không bị hao hao các web AI khác). Thì sau đây là quy trình của mình.

  

Note: Đây là UI, sẽ bàn về UX ở bài khác nhé ạ.

  

1. Đi tìm tài liệu "tham khảo"

  

Hãy dạo quanh những trang web về UI/UX kiểu: Dribbble / Behance / Pinterest / ... (mình sẽ để link ở comment nhé ạ) tìm kiếm chủ để web app mình sắp làm, hoặc chỉ đơn giản là lướt lướt ở đó, note lại những UI mà mình thích.

  

Hoặc đơn giản là mỗi khi lướt web thấy site nào đẹp thì note hoặc bookmark lại để sau làm tài liệu "tham khảo"

  

2. Đóng gói style rule của web app

  

Đây là 1 bước CỰC KỲ quan trọng để đạt được những điểm trên. Sau khi đã có 1 lượt tài liệu "tham khảo" đủ rồi, thì hãy ngồi lại review hết 1 lượt, xem mình thích UI này ở điểm nào, điểm nào khiến mình wow. Hoặc lười hơn thì chụp lại 1 mớ ảnh màn hình gửi lên mấy con AI cho nó phân tích style, rồi điểm nào thích thì pick về.

  

Sau đó đưa ra được 1 bộ quy tắc kiểu như sau:

- Các màu: Primary color, secondary color, tertiary color, text color, ... vân vân là gì

- Spacing phân cấp ra sao (ví dụ section là 48px, các box là 24px, các items là 16px, ...)

- Border radius như thế nào (ví dụ box to thì rounded lớn, box nhỏ thì 8, button, các element thì 4, ...)

- Font gì? Font của title, header có cần khác với body, caption không?

- Các loại section, component yêu thích, ...

- vân vân và mây mây

  

Chỗ này nếu bạn lo là không thể làm đủ chi tiết thì không cần phải lo quá, vì đây chỉ là bản phác thảo mà thôi, đừng lo, nghĩ được gì, thấy được gì, ghi đó.

  

Nếu dùng thư viện kiểu tailwindcss thì note kiểu tailwind nhé anh em, rất tiện để theo dõi về sau. Nói chung là note hết ra những cái Quy tắc UI mà mình thấy đẹp, mình thích. Tránh tính trạng khi code mỗi khu vực 1 style khác nhau, mỗi màn 1 style khác nhau, cái nút ở chỗ này khác cái nút ở chỗ kia, ...

  

3. Triển khai thử nghiệm

  

Sau khi đã có các quy tắc mà bạn nghĩ ra được bên trên, tống hết và 1 cái file md hoặc đưa hết vào 1 promt và cho AI bắt đầu code những cái màn hình/section/component đầu tiên (mình hay cho vào file cho dễ viết, dễ thể hiện những cái phức tạp, có gì sau promt lại cũng dễ)

  

Ví dụ:

Bạn là Senior UI/UX Designer. Code UI phải sạch, tối giản. Luôn tuân thủ 100% các Rules sau, sử dụng Tailwind CSS:

- Palette: BG/Surface (neutral-900/neutral-800), Text (neutral-200/neutral-400), Accent (yellow-500), Border (neutral-700).

- Cấm: shadow-*, gradient, glass effect, emoji.

- Border: Luôn dùng border border-neutral-700 (1px).

- Spacing: Các cấp 8 12 16

- Font: Heading: Montserrat Alternates (600). Body: Be Vietnam Pro (400/500).

- Radius: Button: rounded-lg (12px). Card: rounded-xl (16px). Input: rounded-md (8px).

- Icons: Lucide Icons, outline, size w-5/w-6.

- vân vân và mây mây

  

Code cho tôi page/section/component sau:

[mô tả về thứ cần code]

  

Lúc này AI sẽ code cho bạn UI cơ bản là sẽ phù hợp với yêu cầu bên trên của bạn.

  

4. Review và refactor UI

  

Khả năng rất cao là UI bên trên cũng tuân thủ kha khá yêu cầu của bạn, nhưng sẽ có rất nhiều điểm bạn thấy chưa ưng: có thể là do AI code ngu, có thể có những cái bạn quên định nghĩa nên AI nó làm đại, có những cái bạn chọn nhưng nó không quá đẹp, ...

Lúc này hãy sửa UI hoặc bảo con AI sửa, sửa đến khi nào bạn ưng thì thôi. Và khi bạn thấy ưng rồi thì có thể cho AI làm thêm 1 số page/section/component khác dựa vào style của những cái đã code

  

5. Đóng gọi lại Rules mới cực chỉ tiết

  

Đến đoạn này chúng ta đã có những UI đẹp, ưng ý, lúc này hãy bảo con AI rằng, đọc tất cả những UI trong dự án đóng gói thành quy tắc và đưa vào file UI_RULES md cho tao

  

AI nó làm cức chi tiết từ việc padding của cái button hay là khoảng cách giữa các section, hay là font size ở mỗi chỗ, ... thành 1 file siêu dài (ảnh minh hoạ)

  

Bạn có thể đọc lại xem cấn chỗ nào, không thích chỗ nào hoặc nhờ con AI sử dụng kiến thức design để review xem có vấn đề gì không

  

6. Áp dụng với các UI mới

  

Mỗi khi bạn làm UI mới, hãy đưa file UI_RULES . md lên và bảo con AI áp dụng vào, cẩn thận thì áp dụng xong bắt nó review lại xem đã làm đúng chưa.

  

7. Refactor, update, đổi style trong tương lai

  

Trong tương có thể các bạn sẽ đổi ý, đổi style, làm những thứ không giống trong UI_RULES thì rất đơn giản có 2 kiểu ở đây.

  

- Sửa UI không theo UI_RULES => Bảo AI update lại UI_RULES theo UI mới của bạn

- Sửa UI style toàn hệ thống => Sửa UI_RULES và bảo Ai review/update các UI cũ

  

Thì đây chính xác là cách mình làm việc với AI để cho ra những cái UI mà các bạn có thể thấy được ở lehuythai[.]com hay vocabolt[.]fun đấy ạ, mình đâu tư thời gian và suy nghĩ rất nhiều cho bước 1-2. 

  

Oke còn gì nữa không nhỉ, mình nghĩ tạm thời thế đã. Anh em cố ý tưởng gì hay thì chia sẻ với mọi người để cùng làm nhé ạ. Cảm ơn anh em.