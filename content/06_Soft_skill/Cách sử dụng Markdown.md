---
Author: HoàiAn Thái
Date: Invalid date
Status: Done
Tag:
  - Skill
---
Markdown là ngôn ngữ đánh dấu văn bản được tạo ra bởi John Gruber, sử dụng cú pháp khá đơn giản và dễ hiểu, dễ nhớ. Nếu nắm vững các cú pháp của Markdown bạn có thể trình bày bài viết của mình một cách mạch lạc, ấn tượng mà không mất nhiều thời gian. Bài viết dưới đây sẽ hướng dẫn để bạn có thể hiểu và sử dụng được ngay những cú pháp đó.
- [[#Tiêu đề]]
- [[#H1]]
    - [[#H2]]
        - [[#H3]]
- [[#Nhấn mạnh, highlight]]
- [[#Link, image]]
    - [[#Link]]
    - [[#Image]]
    - [[#List]]
        - [[#List dạng gạch đầu dòng]]
        - [[#List dạng số]]
    - [[#Horizonal rules]]
    - [[#Blockquotes]]
    - [[#Escape markdown]]
    - [[#Viblo advanced Markdown Embedment]]
        - [[#Youtube]]
        - [[#Vimeo]]
        - [[#Slideshare]]
        - [[#Codepen]]
        - [[#Gist]]
    - [[#Công thức toán học]]
# Tiêu đề
Bạn có thể viết các lớp tiêu đề h1, h2, h3 cho đến h6 bằng cách thêm số lượng ký tự # tương ứng vào đầu dòng. Một ký tự # tương đương với h1, 2 ký tự # tương đương với h2 ...
Ví dụ khi viết
```Python
# h1
## h2
### h3
```
thì kết quả sẽ là
# H1
## H2
### H3
# **Nhấn mạnh, highlight**
Bạn có thể kẹp một từ ở đầu và cuối bằng 1 ký tự * để _in nghiêng_, 2 ký tự ** để **bôi đậm**, và 3 ký tự *** để _**vừa in nghiêng vừa bôi đậm**_.
```Python
in nghiêng*
**bôi đậm**
***vừa in nghiêng vừa bôi đậm***
```
Để viết `inline code`, bạn dùng cú pháp
```Python
`inlide code`
```
Để highlight block code, bạn viết
````Python
```php
echo ("highlight code");
```
````
sẽ được kết quả là
`echo ("highlight code")`
# **Link, image**
## **Link**
Để chèn link vào bài viết, bạn dùng cú pháp sau
`[title](http://~)`
Ví dụ
`[Markdown](http://https://vi.wikipedia.org/wiki/Markdown)`
=> [Markdown](http://https//vi.wikipedia.org/wiki/Markdown)
## **Image**
Cú pháp chèn hình ảnh như sau
`![alt](http://~)`
Ví dụ nếu viết
`![markdown](https://images.viblo.asia/518eea86-f0bd-45c9-bf38-d5cb119e947d.png)`
thì bạn sẽ chèn được hình ảnh như thế này rồi.
[![](https://images.viblo.asia/518eea86-f0bd-45c9-bf38-d5cb119e947d.png)](https://images.viblo.asia/518eea86-f0bd-45c9-bf38-d5cb119e947d.png)
## **List**
### **List dạng gạch đầu dòng**
- `item`
Ví dụ bạn viết
- `item 1 * item 2 * item 3`
thì sẽ được kết quả như sau
- item 1
- item 2
- item 3
### **List dạng số**
`1. item`
Ví dụ bạn viết
`1. item 1 2. item 2 3. item 3`
thì sẽ được kết quả như sau
1. item 1
2. item 2
3. item 3
## **Horizonal rules**
Để có được horizonal rules bạn chỉ cần viết *** như sau
- `** horizonal rules`
kết quả là
---
horizonal rules
## **Blockquotes**
Cú pháp blockquotes là
`> text`
kết quả:

> text
## **Escape markdown**
Đôi khi trong khi viết bài bạn sẽ sử dụng những kí kiệu trùng với cú pháp của markdown. Để phân biệt, bạn chỉ cần thêm dấu \ trước những kí hiệu đó là được.
Ví dụ nếu bạn viết
`\*text*`
thì kết quả sẽ là *text* chứ không phải _text_(in nghiêng)
## **Viblo advanced Markdown Embedment**
**Cách 1:** Dùng cú pháp dưới đây để chèn link video từ tất cả các nguồn:
`{@embed: URL}`
Ví dụ:
`{@embed: https://www.youtube.com/watch?v=HndN6P9ke6U}`
  
**Cách 2:** Hiện tại Viblo hỗ trợ chèn link từ **Youtube**, **Vimeo**, **Slideshare**, **Codepen**, **Gist**, **JSFiddle** và **Google Slide**. Cú pháp như dưới đây:
### **Youtube**
`{@youtube: Youtube ID or URL}`
Ví dụ
`{@youtube: https://www.youtube.com/watch?v=HndN6P9ke6U}`
### **Vimeo**
`{@vimeo: Vimeo ID or URL}`
Ví dụ
`{@vimeo: https://vimeo.com/62604492}`
### **Slideshare**
`{@slideshare: Slideshare ID or URL}`
Ví dụ
`{@slideshare: http://www.slideshare.net/asanzdiego/markdown-slides-en}`
### **Codepen**
`{@codepen: Codepen ID or URL}`
Ví dụ
`{@codepen: https://codepen.io/nickmoreton/pen/gbyygq}`
### **Gist**
`{@gist: Gist ID or URL}`
Ví dụ
`{@gist: https://gist.github.com/JeffreyWay/207e3bfdb5cafff050a1d75dbf755a5c}`
## **Công thức toán học**
Bên cạnh các cú pháp Markdown phổ biến, Viblo còn hỗ trợ viết các công thức toán học theo cú pháp của TeX_TeX_.