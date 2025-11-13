---
title: Jasper Injection
tags: [Java]
slug: jasper-injection
date: 2025-03-11 14:59:21+0000

---

Khi được tiếp xúc với các hệ thống code bằng Java, mình thấy có khá nhiều website sử dụng thư viện JasperReports trong việc render báo cáo. Tuy là một công cụ mạnh mẽ nhưng Japser sở hữu function mà có thể lợi dụng để RCE. Nên mình quyết định mày mò nó một chút, cũng như chia sẻ những kiến thức mà mình biết và tìm hiểu được được về Jasper.
## Code Injection In Jasper
### Một chút khái niệm
Jasper là một thư viện có mục đích chính dùng để render nội dung. Được sử dụng chủ yếu cho việc chuyển đổi từ file có cấu trúc sang văn bản báo cáo, có thể xuất ra với nhiều định dạng khác nhau như: PDF, HTML, Excel,...
Jasper hỗ trợ nhiều file đầu vào:
- JRXML: file XML với các attribute của jasper
- JASPER: file được compile từ jrxml
- CSV, JSON, XML,...

Trong phạm vi bài viết, mình sẽ đề cập đến chức năng phổ biến khi sử dụng thư viện JasperReports mà mình hay gặp nhất, đó là chuyển đổi file JRXML sang định dạng PDF.
### Cấu trúc file JRXML
Như mình đã nói ở trên, file JRXML cơ bản là một file XML nhưng các attribute sẽ tuân theo tiêu chuẩn của một Jasper Template, nên ta cũng sẽ có tag `<?xml>` như bao file XML khác.
Thẻ root của một file JRXML sẽ là `<jasperReport>`, trong thẻ đó ta sẽ khai báo những thuộc tính cơ bản của report như in dọc hay ngang, ngôn ngữ sử dụng, tên báo cáo, đồng thời có thêm một vài XSD references. Nhưng về cơ bản thì ta chỉ cần thuộc tính name là đủ, các thuộc tính còn lại là tùy chọn, có thể có cũng có thể không.
Bên trong thẻ root, ta có thể chèn vào các thẻ khác nhau để biểu đạt nội dung của báo cáo, từ các label hoặc chữ cố định đến các biến có giá trị thay đổi, cũng như có thể chèn vào hình ảnh, format bảng, chức năng tính toán,...
Một file JRXML in ra chữ Hello World đại khái sẽ như này:
```xml!
<?xml version="1.0" encoding="UTF-8"?>
<jasperReport name="hello"> 
    <detail>
        <band height="50">
            <staticText>
                <reportElement x="200" y="10" width="200" height="30"/>
                <textElement textAlignment="Center">
                </textElement>
                <text><![CDATA[Hello World]]></text>
            </staticText>
        </band>
    </detail>
</jasperReport>
```
Chi tiết về các attribute sử dụng trong JasperReport các bạn có thể tham khảo tại [Jasper document](https://jasperreports.sourceforge.net/JasperReports-Ultimate-Guide-3.pdf) . Thay vào đó, mình sẽ đi vào tính năng khiến cho Jasper trở thành công cụ render report mạnh mẽ, đó là Expressions.
### Expressions
#### Kiến thức nền tảng
Là một tính năng mạnh mẽ Jasper cung cấp cho người dùng, các expression (có thể dịch ra là các biểu thức, các mô tả) được sử dụng để khai báo biến trong báo cáo, cũng như hỗ trợ xử lý các biến đó một cách linh hoạt từ hiển thị cho đến tính toán. Để làm được điều đó, các expression sẽ được viết bằng ngôn ngữ lập trình, mà mặc định trong Jasper sẽ là Java code.
Vừa là tính năng có thể giúp việc xử lý dữ liệu in ra trên báo cáo được trực quan và dễ dàng, nhưng cũng vừa là nơi để chèn code tùy ý. Tuy nhiên có một lưu ý rằng code truyền vào bên trong các expression cần phải trả về một kiểu dữ liệu nhất định để hoạt động, nếu không sẽ gây lỗi khi render.
Các expression này sẽ được trigger trong quá trình render file JRXML sang PDF. Nếu không có cơ chế phòng thủ phù hợp, mình hoàn toàn có thể khai thác lỗi code injection để RCE, đọc tải file,...
#### Ví dụ
Để ví dụ thêm trực quan, mình có một đoạn code spring đơn giản, phục vụ việc upload 1 file JRXML và trả về file report.pdf
Mình sử dụng Jasper phiên bản 6.21.4 là ver mới nhất của dòng 6., vì mình thấy dòng 6. hiện đang được sử dụng nhiều nhất.
![image](https://hackmd.io/_uploads/BykfHMEnJg.png)
Sử dụng luôn ví dụ bên trên, thay vì sử dụng thẻ `<text>` để hiển thị nội dung tĩnh trong field `<staticText>`, mình sẽ sử dụng thẻ `<textFieldExpression>` dùng để diễn đạt nội dung bên trong thẻ `<textField>`, đại khái lúc này file JRXML sẽ như sau:
```xml!
<?xml version="1.0" encoding="UTF-8"?>
<jasperReport name="1337"> 
    <detail>
        <band height="50">
            <textField>
                <reportElement x="200" y="10" width="200" height="30"/>
                <textFieldExpression><![CDATA[new BufferedReader(new InputStreamReader(Runtime.getRuntime().exec("whoami").getInputStream())).readLine()]]></textFieldExpression>
            </textField>
        </band>
    </detail>
</jasperReport>
```
![image](https://hackmd.io/_uploads/Hkf4HfEnyl.png)
Tải file PDF về và output câu lệnh sẽ hiển thị ra ngoài:
![image](https://hackmd.io/_uploads/rkv90a7h1l.png)
Ngoài cách tấn công này ra thì một số bài viết trên mạng thì đa số họ sẽ tấn công vào thẻ `<variableExpression>` trong quá trình khai báo biến `<variable>`, nên mình sẽ không nhắc đến nó nữa, mình sẽ để chúng ở trong phần ref để các bạn có thể tham khảo thêm.
## Bypasses
Giả sử như dev giờ đã biết được tầm nghiêm trọng của vấn đề và có một số biện pháp phòng thủ, mình sẽ đưa ra một số phương pháp có thể bypass cơ chế phòng thủ như sau:
### Expression Attribute
Hướng đầu tiên mà mình nghĩ đến trong các trick bypass đó là sử dụng các thẻ khác ngoài những thẻ đã có sẵn payload trên mạng. JasperReports có kha khá các expression khác mà mình có thể lợi dụng, vì bản thân trường dữ liệu nào cũng cần có nhu cầu hiển thị và xử lý dữ liệu linh hoạt.
Ta có thể sử dụng attribute defaultValueExpression bên trong thẻ `<parameter>`, rồi call tại thẻ textField quen thuộc. Để call đến param, trong textFieldExpression mình sẽ sử dụng cú pháp `$P{Tên param}`:
![image](https://hackmd.io/_uploads/SyuawfNh1x.png)
Tại thẻ variable thì còn một expression nữa ngoài variableExpression là initialValueExpression cũng có thể chèn code, call đến variable thông qua syntax `$V{Tên variable}`:
![image](https://hackmd.io/_uploads/SkZ-FfNhyl.png)
Một lưu ý nhỏ là để in ra giá trị variable thì textfield sẽ được khai báo bên trong thẻ `<title>`, nếu sử dụng `<detail>` sẽ luôn hiển thị giá trị null
![image](https://hackmd.io/_uploads/ByX33fN2Je.png)
![image](https://hackmd.io/_uploads/BkQkTzEhJx.png)
Mò mẫm document, mình thấy nếu như không nhất thiết phải lấy output của câu lệnh thì tồn tại một số Expression đặc thù có thể lợi dụng để RCE, nhưng nó sẽ gây ra lỗi. Attribute imageExpression thuộc thẻ `<image>` vốn được sử dụng để Jasper có thể lấy ảnh từ một URL xác định dạng String, khi không đáp ứng đúng dữ liệu yêu cầu, câu lệnh sẽ trả về lỗi nên mình vẫn có thể lợi dụng để thực hiện các câu lệnh blind để confirm.
Tại đây vì mình truyền vào Runtime.getRuntime().exec(), output trả về sẽ là một object thuộc ProcessImpl nên chương trình sẽ văng ra lỗi như sau:
![image](https://hackmd.io/_uploads/BJXTSDNnkx.png)
Nếu như set up để câu lệnh trả về dữ liệu, Jasper khi cố gắng truy cập URL để lấy byte ảnh về thất bại cũng sẽ văng ra lỗi, tuy nhiên mình thấy extract thông tin 1 dòng thì có vẻ không giòn lắm:
![image](https://hackmd.io/_uploads/rJRUIP4h1l.png)
Tiếp đến là attribute printWhenExpression có thể được khai báo trong thẻ `<band>` hoặc `<reportElement>`, sử dụng khi ta chỉ muốn dữ liệu được in ra theo một điều kiện nhất định, nên nó yêu cầu cần trả về giá trị kiểu Boolean hoặc giá trị null. Việc truyền java code vào attribute này cũng sẽ gây lỗi lúc render PDF nên ta ưu tiên bằng một số câu lệnh blind như curl, ping,...
![image](https://hackmd.io/_uploads/HJHsvwVnyx.png)
![image](https://hackmd.io/_uploads/S1ZTvPNnkx.png)
Attribute conditionExpression thuộc thẻ `<style> -> <conditionalStyle>` có tác dụng mô tả điều kiện để áp dụng một style cho dữ liệu cụ thể, cũng yêu cầu dữ liệu trả về Boolean. Để trigger được điều kiện này, ta cần một element trong file JRXML áp dụng style đã inject, tại đây mình PoC với thẻ textField cho tiện:
![image](https://hackmd.io/_uploads/H1WDiwE3yx.png)
Nếu như ta muốn thay đổi cách hiển thị của một vùng dữ liệu, ta có thể sử dụng propertyExpression để mô tả điều kiện cũng như quy định cách biến đổi. Đồng thời cũng là một nơi để chèn code. Tuy trả về kiểu String, expression này không phải là một giá trị hiển thị mà chỉ có thể thiết lập thuộc tính cho giá trị đó. Khi sử dụng mình vẫn nên ưu tiên các câu lệnh out bound để confirm bug sẽ tốt hơn:
![image](https://hackmd.io/_uploads/Hkt-guVh1g.png)
Ngoài những expression mình đã liệt kê cách khai thác ở bên trên, vẫn còn nhiều các expression khác như: dataSourceExpression, filterExpression, anchorNameExpression,... Mỗi loại sẽ được trigger trong một thẻ và dưới điều kiện cụ thể, các bạn có thể tùy vào luồng nghiệp vụ của báo cáo để đa dạng hóa nơi chèn code, bypass các blacklist nếu có.
### Jasper compiler
Trong một file JRXML, có một thuộc tính của thẻ root mà mình chưa đề cập đến ở bên trên, nó là `language` - tức ngôn ngữ xử lý các expression xuất hiện trong JasperReports. Nếu không được khai báo thì ngôn ngữ mặc định sẽ là Java (cũng là ngôn ngữ mà mình sử dụng demo ở những phần trên). Tuy nhiên, Jasper hỗ trợ xử lý thêm 3 loại ngôn ngữ khác, khi được khai báo trong `language` thì chúng sẽ được xử lý với compiler tương ứng, đó là:
- Groovy: JRGroovyCompiler
- Javascript: JavaScriptCompiler
- BeanShell: JRBshCompiler (Đã không còn hỗ trợ từ Jasper 6.6.0 nên trong phạm vi bài viết mình sẽ không đề cập đến nữa)

Điều kiện để Jasper compile được các ngôn ngữ này là hệ thống cần có lib để xử lý ngôn ngữ đó tương ứng. Với groovy thì sẽ cần lib: org.apache.groovy -> groovy-all, còn với javascript ta sẽ import thêm lib: org.mozilla -> rhino.
Cụ thể thì mình sử dụng version sau:
```xml!
<!-- https://mvnrepository.com/artifact/org.codehaus.groovy/groovy-all -->
<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy-all</artifactId>
    <version>3.0.20</version>
    <type>pom</type>
</dependency>

<!-- https://mvnrepository.com/artifact/org.mozilla/rhino -->
<dependency>
    <groupId>org.mozilla</groupId>
    <artifactId>rhino</artifactId>
    <version>1.7.14</version>
</dependency>
```
#### Groovy
Để Jasper sử dụng groovy làm ngôn ngữ biên dịch code trong các expression, ta thêm property `language="groovy"`, cũng như hệ thống có chứa lib hỗ trợ xử lý Groovy.
Mình ví dụ một payload chạy lệnh notepad thông qua method `.execute()` trong groovy:
![image](https://hackmd.io/_uploads/HkfI_SLnJl.png)
Lưu ý là mình không sử dụng được lệnh println để in ra kết quả câu lệnh, chẳng hạn như: `println("${"cmd /c ver".execute().text}"` sẽ hiển thị output câu lệnh tại server console, còn tại PDF nội dung sẽ là null:
![image](https://hackmd.io/_uploads/H1tXtS8n1x.png)
![image](https://hackmd.io/_uploads/rk9HtBI3yg.png)
Ngoài việc thực hiện lệnh, mình cũng có thể ghi file nếu như biết đường dẫn bằng `newFile("path/to/shell").write('1337')`, hoặc nhiều các tác vụ file khác. Mình nghĩ nên dùng các native syntax code của từng ngôn ngữ thay vì thực thi câu lệnh để tránh việc bị bên blue phát hiện sớm (nếu có).
#### Javascript
Bản thân javascript không có native syntax để execute command, nên mình sẽ lợi dụng khả năng gọi Java trong javascript để call các package thông qua syntax `Packages`, cái này khá hữu ích khi bên họ chặn theo kiểu blacklist 1 chuỗi method `Runtime.getRuntime().exec()` hoặc tương tự.
Lưu ý là một textField sẽ chỉ có thể in ra nội dung trên 1 dòng nên hầu hết những câu lệnh trên mình hay demo bằng whoami, để lấy được nhiều input hơn, mình sẽ sử dụng thêm attribute `isStretchWithOverflow="true"` để Jasper tự động kéo dài textField cho mình, nếu không có attribute đó Jasper chỉ in ra 1 dòng đầu tiên của output command.
Nếu sử dụng javascript ta có thể thoải mái sử dụng ; để chèn nhiều câu lệnh, mình ví dụ một đoạn code ngắn để lấy hết output câu lệnh `dir`:
```javascript!
var r = Packages.java.lang.Runtime;
var p = r.getRuntime().exec("cmd /c dir");
var i = Packages.java.io.InputStreamReader(p.getInputStream());
var b = Packages.java.io.BufferedReader(i);
var line;
var output = "";
while ((line = b.readLine()) !== null) {
    output += line + "\n";
}
b.close();
p.waitFor();
output;
```
Tại PDF mình đã có thể lấy hết đc output câu lệnh:
![image](https://hackmd.io/_uploads/ryswnUIhyg.png)
### Multi-line Code Injection
Nếu như các bạn để ý thì khi chèn Java và Groovy code thì hầu như mình luôn ưu tiên dùng one-line code, bởi vì đôi khi sử dụng dấu `;` thì sẽ xảy ra lỗi thực thi, lúc trước mình chưa hiểu tại sao nên cũng tạm thời bỏ qua mà sử dụng one-line code.
Tuy nhiên, việc chèn nhiều hơn 1 câu lệnh sẽ cần thiết nếu như ta muốn sử dụng native code để thực hiện reverse shell, download file, upload file,... thay vì chạy lệnh hệ thống. Nên mình cũng ngồi mày mò debug để nắm rõ vấn đề.
#### Java
Mình sẽ truyền vào `1337;;;` tại textFieldExpression để thử đã.
Với cả 2 ngôn ngữ, việc expression được biến thành source code như thế nào nào sẽ nằm tại method `compileReport`, mình đặt breakpoint từ nó để debug:
![image](https://hackmd.io/_uploads/SkcsJvU31e.png)
Từ JasperCompileManager#compileReport, Jasper kiểm tra ngôn ngữ để chọn đúng Compiler cho report. Vì đây là Java nên Jasper sẽ sử dụng JRJdtCompiler#generateSourceCode
![image](https://hackmd.io/_uploads/HJFklPIhJg.png)
Tiếp tục call đến JRClassGenerator#generateClass, các method và param sẽ được thiết lập tại đây bằng một loạt các method generate:
![image](https://hackmd.io/_uploads/ry6bgPLn1g.png)
Nhảy vào JRClassGenerator#generateMethod, mình đã thấy nơi xử lý code có trong expression, đó là thông qua method JRClassGenerator#writeExpression
![image](https://hackmd.io/_uploads/S1SBeDInkg.png)
Câu lệnh của mình được đưa vào lệnh case, với số case được lấy từ expression Id, còn giá trị thì nối chuỗi với đoạn code `value = ...`, sau đó break ra ngoài
![image](https://hackmd.io/_uploads/SyRFgPU2yx.png)
Nếu vậy thì với Java để chèn thêm code mình nghĩ khá đơn giản, chỉ cần đưa cho value một chuỗi ban đầu rồi xuống dòng viết tiếp lệnh là được:
![image](https://hackmd.io/_uploads/S1ylEvU31x.png)
#### Groovy
Tuy nhiên khi sử dụng cách đó với groovy thì không thành công:
![image](https://hackmd.io/_uploads/BkT8qvLnJg.png)
Điểm khác biệt nằm tại JRGroovyGenerator#writeExpression, lúc này code được đưa vào `value = (` và đồng thời có một dấu `);` kết thúc trong câu lệnh `if`, câu lệnh bị lỗi do mình chưa đóng ngoặc value lại:
![image](https://hackmd.io/_uploads/BJ0cqPL3kx.png)
Để chạy được multi-line ở groovy, mình cần thêm `);` sau chuỗi, đồng thời comment lại dấu `);` đằng sau để không trigger lỗi syntax:
![image](https://hackmd.io/_uploads/ryDkhvUnJg.png)
![image](https://hackmd.io/_uploads/ry3e3vInkg.png)

=> Khi sử dụng multi-line code tại Java và Groovy, ta đã ngắt câu lệnh value tức giá trị trả về của textFeild là chuỗi tại dòng đầu tiên, nên để lấy được output, mình sẽ cần phải set lại giá trị cho biến value bên trên.
Ví dụ tại Java, mình sẽ gán lại giá trị cho value bằng output câu lệnh `cmd /c ver`:
![image](https://hackmd.io/_uploads/H13NuO8nJe.png)
![image](https://hackmd.io/_uploads/Byde__Inyx.png)
Tương tự với Groovy, mình gán lại output của def cmd vào là có thể lấy hết output:
![image](https://hackmd.io/_uploads/Bya-tOIhkx.png)
![image](https://hackmd.io/_uploads/r1M4tdI3yx.png)

## Kết luận
Tùy vào tình hình cụ thể của hệ thống mà mình sẽ đưa ra lựa chọn nên tấn công theo cách bypass nào. Cũng như có thể kết hợp việc sử dụng ngôn ngữ khác và thẻ lạ để da dạng hóa cách thức tấn công, khiến cho việc truy vết trở nên khó khăn hơn. Mình mong bài viết của mình sẽ cung cấp cho mọi người những kiến thức tổng quan về Jasper, từ đó có thể tìm được thêm nhiều kiểu bypass khác nữa.
Hiện tại mình chưa thấy có cách nào để phòng tránh triệt để lỗi này, nên các developer khi sử dụng JasperReports cần chú ý hạn chế tối đa cho người dùng được chỉnh sửa các thành phần trong template, chỉ sử dụng khung template có sẵn.
## References
- https://jasperreports.sourceforge.net/JasperReports-Ultimate-Guide-3.pdf
- https://www.cnblogs.com/snad/p/17566075.html
- https://gist.github.com/v-p-b/dd95c72c6924dc1338e78e9d380bd388
- https://insinuator.net/2023/06/jasper-reports-library-code-injection/