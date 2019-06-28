---
title: "使用requests发送指定Content-Type的multipart请求"
date: 2019-06-25T01:54:55+08:00
draft: false

tags:
- python
- HTTP
---


群里有朋友提问，如何使用`requests`实现如下图postman的`Content-Type`设置？

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190624164458.png)



立刻就有人问了，这是什么？这和HTTP header里的`Content-Type`头是一回事吗？

不是。这是POST请求的一种独特的编码方式：`multipart/form-data`。鉴于这货平时露面机会比较少，我们有必要先介绍一下它。

## multipart/form-data是什么？

HTTP协议规定了HTTP请求的基本格式，包括method、url、header、body等，规定如果请求中需要携带数据，应该放在body体中，但没有具体规定body体内数据的编码规则。携带数据最常见的方式就是form表单了，它不仅可以携带普通表单字段，还可以携带文件。在文件上传的时候，form表单会使用一种可能是HTML圈最复杂的编码方式：`multipart/form-data`。也就是本文的主角。

在文件上传的时候，form表单必须设置`enctype`属性为`multipart/form-data`，然后就可以提交多个文件，混杂多个表单字段了。



来研究一个栗子🌰：

```html
<form method="POST" action="http://www.baidu.com" enctype="multipart/form-data">
    <input type="file" name="file1" /><br />
    <input type="file" name="file2" /><br />
    <input type="text" name="text1" value="value1" /><br />
    <input type="text" name="text2" value="value2" /><br />
    <input type="submit" name="submit" value="submit" />
</form>
```

表单很简单，包含两个文件和两个文本字段。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190624182726.png)

提交的表单可以使用Charles、Wireshark等抓包工具抓到原始报文。

```
POST / HTTP/1.1
Host: www.baidu.com
Connection: keep-alive
Content-Length: 4420
Cache-Control: max-age=0
Origin: null
Upgrade-Insecure-Requests: 1
DNT: 1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBtCkPAw9wNtlcOjj
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.100 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8

------WebKitFormBoundaryBtCkPAw9wNtlcOjj
Content-Disposition: form-data; name="file1"; filename="druid.png"
Content-Type: image/png

�PNG


IHDR�^�KiTXtXML:com.adobe.xmp<?xpacket begin="﻿" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="Adobe XMP Core 5.6-c140 79.160451, 2017/05/06-01:08:21        ">
 <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <rdf:Description rdf:about=""/>
 </rdf:RDF>
</x:xmpmeta>
<?xpacket end="r"?>-CD�gAMA���asRGB���PLTE���mHLyQO}VWR6;���J,3[@DZ=G���Q=AlUV�d_`BGuOTV;A�^^@,+�deʛ����`>@|VQuON�`]p[`[9?���KAOAR12R��������|{�����pjRH_������>��\��e�`�����^HFJ58���uUS= ���B36F;>������KqŻ�jGH/������!&,���aTo�ls���93F,v����ǿ�hLP���c:3���j\B1Ipm~���y��!��~��rr���������n�֏li�����~yo��Đ��qm�ztd�����zvG,'3Y;6L2-O6BS7C}^W�{ub?D��ڎwrkv�z��Ĺ^�}fgG��壙`gw�ia@��oLII'���|m�y��8R���616˥�Q��&m�Cf}�up���Z��T{���B>I~[f���5$%t��������.FMKTR�Ϟkc���;HQ'�� Rm+?Orr�Y�A7O-MU?Y7���WT���|g{_�Β��^���к|q���������P��������s~����֢��qm=34tl��bkgI^���YMY��XAP�����|U�����vqv�W`-~�ݩ����}sy�[Q��Մ�����4X��]]d@]&5D��e��:���VJe{�涱q>8���|��$1v���~vń�6��]���zq	m�QEn	���$+<������\U\�����ڻ������rof��~����nbs���dIQ���<�㘀����ö�Cs����q�Zs�Eg������G��2�����9k�I��n��
�H��IDAT(�c` p�2((�)ڙb�dU�9y��ef�-�G1u��1�o�b--��&���&p*X|�������ʀ�7U�!�*J���f��U�Af�j�<ײ������_(6f�R�[K�	?噴���.���Z'��O J]7J�[h�:L�����BQ�t�u�ù��'�>�t�H�)�hh/��Z�)  ,�������ڒ�$k���R�� ��䎓�#;�㗳�bܝ�����j���H�y�rXZ�K��03�9w��+���n�Q.�r����̗�X���nl۝rvK^�ܰ���B6U�\,���l���\�&� �9�,������{��ڻS���W4�u����Pw�"y���˕��XWmegdb*ܸ�G�)$�P,��q�'��)My�fQ^^Ff{�Rq�IV�t��y���$�u�Y�jR,L����I� �g:&_�ϸ�U����-����Q�A����P$-�2=0r������n�I�bg���HcIMj�Ŷ�K����e�cq��gI����HN���ȹ�?�Q{-'g���&(����a����>�i���,�}��Rɉ	&[M6$���C�U��Ojk�.z9�r���#�3��#��f��_��x�M�h8�0��l�`JZ< ;eF����.�ۦ��f��dqC����5'kN9ˊ�c��&�
�
ό�8������	�Y�d8���IEND�B`�
------WebKitFormBoundaryBtCkPAw9wNtlcOjj
Content-Disposition: form-data; name="file2"; filename="paladin.png"
Content-Type: image/png

�PNG


IHDR�^�KiTXtXML:com.adobe.xmp<?xpacket begin="﻿" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="Adobe XMP Core 5.6-c140 79.160451, 2017/05/06-01:08:21        ">
 <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <rdf:Description rdf:about=""/>
 </rdf:RDF>
</x:xmpmeta>
<?xpacket end="r"?>-CD�gAMA���asRGB���PLTE���|1"���+�ӻ��Y����|CԺ5r<��F���������  !�̲������ן':0HE�J �ѳ������N,$b)��W�ȩ���B>�����������f����{&;<�g&ÿ�>5BB��R�O%#����~&��o�����Ժ������%~K͙(Ґ'�ǆ<$#�_*��ˊSH3�i%ܹ)��\Ҹzzd��ghd+�����y��7��m�����ٱ%USÑ��`�Y!��T�V$à�?�Ϩ��f�P&��{�i�{{]��������������Jʾ�Č�n������9;1@#ux5��鎎]�o
�y��gT]^"=$ҩ-ի��M½a�@%��m]_G��ʐ�U�9hR/��z_4!��'��x��4ƍ]�n&��a��+Ҵ/��.�w��sǬ��lf>��u�s:�b!��ո�,�\ �ۻ��I�������Ͽ��S��Pl-�����\V($�I"��ĲVmI�Ų�Οr`��bjcȧC-	M=t.#�O�UʱH��S}P?��@��2rR����}99-�񚽫t�w%4&+��h��iҮ&�c'��j�L��vT:îD�Ύ�]&�����X�ַon5��@�uT�yc��o�WR0�Q(���//��l�߾�熖����D��;�����Y%��9�ǵ�}����~
�L0W&�`��Aǟf|?^#�6ø3����Ō3��qHb%ų�p9	��C��k�e)�����&��Щ�IDAT(�c` �������
`��mu���L,�%U�s�]JN�����y0�U��b>��5�x�����������Д�
�o.�x75u��yH�j�L炭�z13+J���n�2�&͟Ǒ�'$g7���4ߋҬ��*�Q�W�d��85��r�r��ّ�/�=a�4D:��CP��V�����sfv����Y���&��|Bv���f���Ӳy��I�ϊTI��JjhZ�9U_0c�^3geFvҼ�n�� ��9����ݤ��/����M�߽˼,T~SH��GR�E[��H�:������;~�_��f�%v`�M
�d�4e�:�p�嚡U���_�4�	S*@^_b�fʸ�hOcK=�V�o3+kZ���,Hz�qwx���:�)M5y�β3dE�������%����_c߽�~2q��V%?�;~k�N���q`��������|Btn�7�]�i+����������.��|��k�_��}�[���J����c]t�䝿�:�e�(<Nb�R��qe�FI��q?v��$�o��T�����N2�F�����vm��^�h����/��<�q.Lޚ�����~zs�[����!Rj�P޴e��`d��������(i�UB���q��h���mq�:�h)Y�Sb��A���:��@�(g���D�>u=�H�U�U���i�!u��;IEND�B`�
------WebKitFormBoundaryBtCkPAw9wNtlcOjj
Content-Disposition: form-data; name="text1"

value1
------WebKitFormBoundaryBtCkPAw9wNtlcOjj
Content-Disposition: form-data; name="text2"

value2
------WebKitFormBoundaryBtCkPAw9wNtlcOjj
Content-Disposition: form-data; name="submit"

submit
------WebKitFormBoundaryBtCkPAw9wNtlcOjj--
```



很抱歉使用baidu作为目标地址。各种测试场景大概是baidu能发挥残热的最重要场合了。



我们来解析一下几个关键信息：

```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBtCkPAw9wNtlcOjj
```

这一行表明了整个body体的content-type：`multipart/form-data`，与平时不同的是，这个属性这次还带了后半段：`boundary=----WebKitFormBoundaryBtCkPAw9wNtlcOjj`。由于请求的整个负载包含了多段内容，所以浏览器帮我们生成了一个足够长足够特殊的“分隔符”，用来将整个body体分割成独立的多段。

17行最后一个header之后是一个空行，表示header部分的结束，整个POST请求的body部分开始。

接下来，分隔符就出现了，单独占据一行。每个分隔符之后，就是这个part的范围。它同样由自己的header、空行、body三部分组成。

每个part可以携带两个头：

- `Content-Disposition`表示此部分的处置方式，在这里统一值为`form-data`，后接`name`属性表示字段名称，如果是文件内容的话后接上传时的本地文件名。
- 如果这个part是一个文件，还可以附加`Content-Type`表示这个文件的类型。这也就是群里小哥问的截图红框部分的底层实现。

每个part的body部分非常简单，粗暴的附加了整个字段的内容。无论是文本字段的值，还是二进制文件本体，都不在话下。

然后是下一个分隔符，下一个part，直到最后，由单独一行**分隔符+`--`**结束part列表以及整个请求body体。

这就是HTTP协议层面的`multipart/form-data`。



## `requests`实现

很遗憾的是，以上实现方式是非常“古老”的文件上传方式，曾经流行于HTML+CGI时代，毕竟那时候连JS都很少很少，更不存在AJAX这种可以独立发送和管理请求的功能。不论有多少表单项，不论其中有几个是文件，不论整个请求有多大失败率有多高，都倾向于一把梭提交整个表单。相比之下，现代前端的请求API都不太倾向于这么设计了。资源都是RESTful，文件上传都是独立异步可容错可重试，就连TCP链接都可以复用，还有什么理由一把梭呢？

所以，`requests`以“不够pythonic”为由差不多放弃了这个古老复杂又可怜的`multipart/form-data`，只提供了非常有限的支持。幸运的是，官方文档给出了指引：它介绍了一个`requests`的扩展`requests_toolbelt`。

借由`requests_toolbelt`的帮助，我们可以创建多个part的multipart请求并为每一个part指定Content-Type。

```python
import requests
from requests_toolbelt.multipart.encoder import MultipartEncoder

fields = {
    'file1': ('druid.png', open('druid.png', 'rb'), 'image/png'),
    'file2': ('paladin.png', open('paladin.png', 'rb'), 'image/png'),
    'text1': 'value1',
    'text2': 'value2',
    'submit': 'submit',
}
multipart = MultipartEncoder(fields=fields)
headers = {
    'Content-Type': multipart.content_type,
}
proxies = {
    'http': 'socks5://localhost:8889',  # Charles
}
r = requests.post('http://www.baidu.com', data=multipart, headers=headers, proxies=proxies, timeout=10)
print(r.text)
```



由此，群友的问题已经回答完毕。



## 参考

https://www.ietf.org/rfc/rfc1867.txt，最权威的multipart规范定义。里面还有更多更详细更变态的用法。

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type

[https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Disposition](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Disposition)

http://cn.python-requests.org/zh_CN/latest/



