---
layout: post
title: Leak private title post via xmlrpc in wordpress
categories: [Research]
tags: [WORDPRESS]
description: Tấn công vét cạn tiêu đề bài đăng riêng tư trên wordpress thông qua xmlrpc.
---

Nhân tiện vài hôm nay vẫn còn nóng hổi vụ [SQLi](https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-6676-cqfm-gw84) trên [Wordpress](https://wordpress.com/) thì mình cũng có ngó nghiêng, xem lại những ghi chú cũ về framework mà lúc trước có lưu lại, cốt là để có thêm dữ kiện đọc hiểu được mấy lỗi sau này mà không cần ngồi ngâm lại từ đầu. Dạo trước, trong lúc nghiên cứu thì mình cũng có tìm ra một vài cái thú vị. Nên cứ ném lên đây như khai xuân đầu năm cho cái blog cùi bắp này.

Đối với wordpress, từ khoá xmlrpc cũng không còn xa lạ nữa. Có một số tác động bảo mật phổ biến liên quan đến từ khoá này như [DoS](https://hackerone.com/reports/752073), brute force,... Và phương pháp giảm nhẹ mà nhà cung cấp đưa vào sản phẩm của họ là khuyên người dùng nên [tắt](https://mediatemple.net/community/products/dv/360048950192/how-to-disable-xmlrpc.php-for-wordpress) chức năng này nếu không cần sử dụng đến. Và thứ mình trình bày trong bài viết này cũng liên quan đến nó.

Cụ thể là phương thức [pingback_ping()](https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6828), phương thức này có nhiệm vụ chính là kiểm tra sự tồn tại của một bài đăng(private + public) thông qua các tham số`title` được gửi lên từ người dùng. Cách kiểm tra nó như sau:

> $sql     = $wpdb->prepare( "SELECT ID FROM $wpdb->posts WHERE post_title RLIKE %s", $title );

Không phải bị SQLi nữa đâu =)) Cái cần chú ý là câu truy vấn là sử dụng [RLIKE](https://www.w3resource.com/mysql/string-functions/mysql-rlike-function.php). Trong trường hợp này, cũng không biết nó là tính năng hay lỗi nữa. Nhưng vì cách sử dụng này đôi khi lại cho phép kẻ tấn công có thể thực hiện tấn công vét cạn để tìm ra tiêu đề của các bài đăng riêng tư. Mình sẽ nói sơ lại mình luồng khai thác như sau.

https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6867-L6887

![](https://i.imgur.com/l9ApgGM.png)

Sau khi qua các bước kiểm tra tính hợp lệ của tham số truyền lên thì để thoả mãn các điều kiện thì giá trị của `urltest` sẽ là:

> http://localhost:81/wp/p/a=11#title


https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6888
![](https://i.imgur.com/jRO5Rsh.png)

Đến dòng [#6888](https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6888) thì chính giá trị `fragment` sẽ được lấy làm tham số so sánh với cột `title` bởi RLIKE. Giả sử cứ lần lượt vét cạn 36 lần tương đương với 26 chữ cái + 10 kí tự số cho một kí tự của tiêu đề bài đăng thì việc vét cạn này vẫn có xác xuất thành công rất cao. Chỉ có một điểm khó trong hướng khai thác này là `fragment` đã bị bộ lọc chỉ cho phép các kí tự thuộc phạm vị `[^a-z0-9]`. Điều này dẫn đến không thể chèn các kí tự như `%,_,^,...` để giảm thiểu tối đa các trường hợp vét cạn. Vậy nên trong lần vét cạn đầu tiên, kết quả có được có thể chỉ là một phần của tiêu đề bài đăng thôi. Cần hard-core hơn để có được một kết quả hoàn chỉnh.


Tiếp theo nếu trường hợp `ID` không tồn tại sau khi thực hiện truy vấn thì ứng dụng lập tức phản hồi thông báo mà không có **time delay**.

https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6890-L6893
![](https://i.imgur.com/Q0CgUHj.png)

Nếu vượt qua hết các bộ lọc trên thì ứng dụng sẽ thực hiện `sleep(1)` sau đó phản hồi lại kết quả.

https://github.com/WordPress/WordPress/blob/master/wp-includes/class-wp-xmlrpc-server.php#L6922

![](https://i.imgur.com/U5ld8Oi.png)

Gặp `time-base` thì "U là trời" :>.

Request gửi đến máy chủ sẽ có nội dung đầy đủ như bên dưới:

```
POST /wordpress/xmlrpc.php HTTP/1.1
Host: localhost:81
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:84.0) Gecko/20100101 Firefox/84.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: wp-settings-time-1=1610339729; wordpress_test_cookie=WP+Cookie+check; wordpress_logged_in_8bcaf7c55080a17ee3f073d5ffc5ebe6=khaind2%7C1611549328%7CswqbpHOqzhq5b6pKMiqlGSayxOufcvW1XyjuuUOj9yA%7C6ebc767d888826acf6a7c7b1a241c2f22d15799aeb573786460ca4728e90fb31; XDEBUG_SESSION=10918
Upgrade-Insecure-Requests: 1
Content-Length: 324

<?xml version="1.0" encoding="iso-8859-1"?>
<methodCall>
<methodName>
	pingback.ping
</methodName>
<params>
    <param>
        <value>
            <string>
                http://localhost:81/wordpress/p/a=11#private
            </string>
        </value>
    </param>
    <param>
        <value>
            <string>
                http://localhost:81/wordpress/p/a=11#private
            </string>
        </value>
    </param>
</params>
</methodCall>
```

Mình cũng có báo cáo thông qua H1 thì bên vendor phản hồi rằng không chấp nhận `time-based bruteforce`. Thì thôi z ;)) Mong đầu năm cung cấp được một vài thứ bổ ích cho anh em bạn đọc. 
Chúc mừng năm mới 🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉

References:

[1] https://www.hixie.ch/specs/pingback/pingback

[2] https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-6676-cqfm-gw84

[3] https://www.w3resource.com/mysql/string-functions/mysql-rlike-function.php
