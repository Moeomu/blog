---
title: Cloudflare Warp 迷踪
description: Cloudflare的Warp服务似乎很容易就能被刷成无限量，本文旨在探讨原理。
date: 2023-10-09 14:29:43+0800
categories:
    - 逆向分析
tags:
    - Cloudflare
    - Warp
image: https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/webpageheadimageCloudflare_54117147.6hbl4terqlj4.webp
---

本文来源: [MoeomuBlog](/zh-cn/posts/cloudflare-warp-迷踪/)

> 本文所有提及的代码和相关文件可从此链接[cloudflare-warp-analyze.zip](./cloudflare-warp-analyze.zip)下载。
>
> 如果您身处中国，此博客的图片可能会加载不出来，请勇敢跨越中国国家防火墙。

## 起因

在网上闲逛时，发现一个这样的项目：[一键生成2000多万GB的warp+密钥，你懂的](https://replit.com/@ygkkkk/WarpKey-Register-PRO)。不管你懂了没有，反正笔者是没看懂。它为什么能产生这么多的密钥？也许是有一个列表，然后每次运行的时候随机抽一个？

## 文件

既然来都来了，不fork一下说不过去啊。看到这里有几个文件，首先是`replit.nix`里写明了这个项目入口是`wpplusreplit.sh`，那就先从它看起。

```text
> file *
README.md:       UTF-8 text
replit.nix:      ASCII text
warpplus.sh:     ELF executable, 64-bit LSB x86-64, dynamic (/lib64/ld-linux-x86-64.so.2), BuildID=3dfe013b3027047470d1aac10a3504baf6969725, stripped
wpplusreplit.sh: ASCII text
```

## wpplusreplit.sh

`wpplusreplit.sh`代码被混淆了，不出所料。

![Fork](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/1-fork-min.1wiu2onj1if4.webp)

只是区区简单的变量名混淆，稍微有点常识就知道用`bash -x`调试一下代码就行，结果如下：

```bash
bash -c 'bash -c "$(base64 -d <<< "\
YmFzaCAtYyAiJChiYXNlNjQgLWQgPDw8ICJcClltRnphQ0F0WXlBaUpDaGlZWE5sTmpRZ0xXUWdQ
RHc4SUNKY0NsbHRSbnBoUTBGMFdYbEJhVXBEYUdsWldFNXNUbXBSWjB4WFVXZFEKUkhjNFNVTktZ
ME5zYkhSU2JuQm9VVEJHTUZkWWJFSmhWWEJFWVVkc1dsZEZOWE5VYlhCU1dqQjRXRlZYWkZFS1Vr
aGpORk5WVGt0WgpNRTV6WWtoU1UySnVRbTlWVkVKSFRVWmtXV0pGU21oV1dFSkZXVlZrYzFkc1pF
Wk9XRTVWWWxoQ1UxZHFRalJYUmxaWVdrWkZTMVZyCmFHcE9SazVXVkd0MFdncE5SVFY2V1d0b1Ux
VXlTblZSYlRsV1ZrVktTRlJWV210WFYwcEdVMjFvVjFkRlNrWlhWbFpyWXpGa2MxcEYKV2s5WFJU
VldXV3hvUTFVeFpIRlJhbEpZVW14YVdWZHJXa1pUTVZaeUNtRkhjRTlTYXpWWFZrZDBNRmRuY0U1
U1ZGWTJWMWQwYjFVeApWWGxUYmxaU1lsUnNWMVpyVmt0VFJsSldWMjEwV0ZZd2NFZFZNakZ2VmpG
a1JsTnJXbGhXYkZweVdYcEdhMk14Y0VZS1YyczVXRkpVClZsZFhWM2h2VVRGVmVGcElSbEpoYkVw
WlZXMTRZVmRXWkhKWGExcFVUVlphZVVOdFJraGpSVGxUWVhwV1dGWnJaREJOUm1SdVkwVTEKVTFa
R1dUSldNV1F3WWpGVmVBcFdXR3hVWW14YVUxbHNVbk5XTVZweVZtdDBWRkpzU2xkV01qRXdWMFpa
ZDJORlpGWk5ha1oyVm1wRwphMUpzVG5KWGJHaFhZa1p3ZVZkWWNFZGhNazE0WTBWWlMxWXljelZY
UmtwVkNsWnNaRmhXTTJoMlZWUkdWbVZHY0VsU2JFcG9Za1Z3CldsWlhNVFJaVm1SWFdraEtXR0V4
Y0ZWVVZscGhaVlZPZEZKcmFHcFNWR3hVV1Zod1YxZEdXbkphUkVKT1VtMVNkVmt3VlRFS1ZURmEK
UjFkVVJsZE5WMUYzVmxSR1dtVkJjRmRYUjJob1ZXeGtVMk5XVm5GUmJVWmFWbTE0ZVZkcldrdFVi
RXB6VTJ4b1YwMXVhRkJXVkVaaApZMnMxV1dKSFJsTldNVW95Vm14U1FncGxSa3BYVTJ4V1UySkhV
azlaYlhSTFVsWmFSMVp0Um1wTlZtdzBWMnRhY2xNeFpISldWRlpZClVtdHdNVU5zUm5SaFJtaFha
V3RKTUZac1VrSmtNbFpJVTJ0c1ZHSkhhSEJaVkU1RENtVnNXblJqUlU1YVZtczFlbFl4YUhOaE1V
NUgKWTBaV1ZWWnNjR2hXYlhSUFl6RktkVkpzV21sWFJrcDNWbTE0VTFZd05VZFhXR3hyVWpOQ2Mx
VnFRbUZWTVd0M1draE5TMWxWVlhnSwpWMVpHY2sxV1pHaE5XRUpWVmxSS2VtVkdaRUpqUm1ScFVq
RktiMVpYTUhoaU1WWkhWMjVLVjJKdFVuRlphMlEwVm14VmVXTkZPVlZpClZYQkpXbFZhWVZZeFNY
cGhTRXBYWVRKU1RBcFdha1pQWTFaR2MxWnJOVmRoTTBKT1ZtMXdTbVZCY0ZOTmF6VjVWR3hhVjFW
dFNrbFIKYmtKV1lsaFNNMXBXV21Ga1IxWklaRVpTVGxZeFNrcFdiVEV3WTJ4TmVHSklTbGhoZW14
WENsUlZVa05PUlU1elUyNUdWbUpIVWxoVgpiRkpXWld4YWNsVnJkRlZOVld3MVZURm9kMWxXU1hw
VmJGSlZWbnBXZGtOdFVuTlhibEpxVWxoU1YxUldXa3RYUmxsNFlVYzVXbFpyCk5VY0tXVEJhVjFa
V1duTlhiR2hWWWtaYVVGa3ljekZXTVdSMFpFWk9UazFWY0ZaV2JURjNWREpKZUZOdVRsaGhNbEpa
V1d4U2MyTlcKVWxkYVJGSldUVmQwTTFkcll6UlRNVnB4VW0xRlN3cFdNR1JUVG14T2MxcEhhR2hO
V0VKMlZqRmtkMUl4VW5SV2JFcHFVbXh3V1ZWcQpUbTlXTVdSWFZXdDBhVTFyTVRSV2JUVkxWakpL
VmxkdVJsZGlWRlpFVmpCYVlWZEhWa2hrUmxaT0NtRXpRa3BYYkZaaFlURmFkRk5zClZsZGlhM0JZ
VldwT1QwNUJjRmROUmxVeFZteGFZV014Y0VoaVJtaFRWbGhDUjFadGVGTlRNRFZDWTBaT1RsSkdX
alpXVkVreFZERlcKZEZOclpGUUtZa2RvV0ZsWGRHRlVSbEpZWlVkR1UwMVdjREJhUlZwaFZHeGFW
VlpzYkZkV2VrRjRWbTE0VG1WSFJYcGFSbWhvVFVSVwpka05zVm5SbFNHUlVWbFUxZWxscVRuZGhW
a3AwVldzNVdncGlXRkpNVmtaYWExZFhUa1pUYlhoVFlYcFdTVlpyWXpGVE1rWkhVMjVLClQxZEZT
bGhaVjNNeFpHdE9jMVZZYUdGU2JXaHpWVzE0ZDFReFduTlZhMlJzWWtkNGVWbFZWVFZXTVZwekNt
TkZaMHRXYWtreFZERloKZVZKdVNsaGhNbWhXV1d0YVlWVkdjRVpYYkdScVlsVmFSMVF4V210VWF6
RjBZVVp3VjJFeGNISmFWM040VTBaYWMxcEdhR2xTTVVwWQpWMVpTUWsxV2JGY0tWMjVPVm1FeGNI
TlphMlF3VFRGWmVVNVhjRlJOVm13elZqSjBlbE4zY0ZkTlZuQklWakZrVDFJeGNFZFViR1JPClls
ZGplRlpxU2pCVk1VMTRWbGhzVm1Fd2NIRlZiWGhoWTBac2NncFdibVJYVm0xU1dGZHJhSGRVYkZw
elUyNXdWMVl6YUhaWlYzaEwKVjBaV2RWRnNWbGRpVmtWM1ZtcENZV0V4WkZoVWExcFZZbGRvVDBO
dFJYcFJiR2hYVWpOb1dGbDZSbUZXYXpGWENtRkhhRk5XYTNCbwpWbTB3ZUZVeFVrSmpSbkJzWVRG
d1RWZFVSbUZVTWxKSFUyNU9WV0pGTlZsVmJGWjNVekZhY1ZOcVVscFdNRmw2V1RCYVYxUnNXbFZX
CmJHeFhWbnBCZUZacVJtRUtWMFpPYzJKR1dVdFphMlEwVmpGc2NsZHJkRk5OV0VKWFZqSXhNRmRH
V1hkT1ZXUmhVbGRTZWxsV1drdE8KYlVZMlVXeGtWMkpWTVRSV1ZtUTBWRzFXUjFac2JHaFNNRnBV
Vld4V2R3cGhSVTV6VjI1U1RsWnJOVlZWYkZVeFRVWlZlV1JHWkZkUwpNSEJLVlZjMVExWjNjR2hO
V0VKdlZtcEdZV0V4WkZoVWExcHJVbXhLVDFac2FFTlRWbHBZVFZSU2FrMXJXbGhWTWpWTENsZEhT
bFZpClJtaGFZa1pLUjFwWGRFOWphekZXV2taa2FWSnNjRlpXYWtKcllqRmFjMVZzYUd0VFJUVlFW
bTE0VjA1V2NGWlplbFpYWWtWd2VrTnQKU2tWWFZYUlhZa2RSZDFSVldtRUtZekZrY2xkck9WZGhl
bFpYVm0xNFlWbFdWa2RoTTJ4T1ZsaFNWRmxzVm5kVFZsWjBaVVU1VldKVgpjRmxaVlZKVFZqQXhX
RlJxVWxWV1ZuQlBXa1JCZUZOWFJraGlSbEpUVjBWS01ncFdiR04zWlVaVmVWUllaMHRaYTFwWFZs
ZEtWV0pGCk9WZGlXR2d6VlRGYVUxWnNWbk5YYkZKT1ZteHdOVll5ZEZkaGJFNHpZMFprYVZKdVFs
bFhWRVpoVkRKU1IxTnVUbFZpUlRWWkNsVnMKVm5kVE1WcHhVMnBTV2xZd1ZqUldWbWh2VmxkS1Jt
TklSbFppV0ZJeldUQmFjMWRSY0dwU2JWSnpWbTE0ZDJWR1ZsaGxSMFpwVW10dwpWbFZ0ZUc5WGJV
VjRVMjFvVjJFeVVrd0tWbXhhWVdNeFduUlNiR1JwVW01Qk1sWXlkRk5TTVZGNFdrVm9WR0V4Y0Za
WmJHUnZWMFZPCmRGTnNiR2hTTUZwWVdWUktUMDB4VW5OWGF6bHFUVlZ3V2tOc2NFaGlSbEpUVjBW
S1dRcFdiVEUwVm1zeFYxUnFUbXBTYkhCUFZGYzEKYjFSR1pGVlJiR1JxVFdzMVNGVnROVk5oVmtw
MVVXeHNWbUpHU2xoVVYzaFdaVVphY2s5V1VtbFdWbXcyVjFSQ1lWTXhWbkpOVldocwpDbEpVUmxW
V2FrbzBaVlpzVjFadVRVdFZNRnBQWkVkR1NHSXdkRlZXZWtaeVdXMTRUMWRIU2tkVWJFcFhWak5v
TVZkWE5YWmtNa1pXClpFWlNWRll5VW1GWmJGWmhUbXhzVmxSclNtZ0tWbGhDUjFWV1pITlNSbkEy
VFVSc1NtRlhkSEJUVldSTFlVZE5lVm95WkVwaFZrcEMKVTFka2RsQlRTWEJKYVVKcFdWaE9iMGxE
U1d0UlEwbExJaWtpSUdKaGMyZ2dJaVJBSWdvPSIpIiBiYXNoICIkQCIK")" bash "$@"' bash
```

继续用base64解码，最终得到如下的代码：

```sh
#!/bin/bash
echo
echo "请稍等，下载更新中……"
rm -rf warpplus.sh
wget -N https://gitlab.com/rwkgyg/CFwarp/-/raw/main/point/warpplus.sh >/dev/null 2>&1
chmod +x warpplus.sh
./warpplus.sh
```

什么？啊？啊？啊？就是更新一下然后启动`warpplus.sh`文件是吧？

## CFwarp.sh

上文提到了一个项目[rwkgyg/CFwar](https://gitlab.com/rwkgyg/CFwar)，似乎是这个脚本的作者存储这些脚本和更新的地方。在这里找到的CFwarp.sh再次使用上述方法解密后，得到如下代码：

```sh
...
if [[ $cpu = amd64 ]]; then
curl -sSL -o warpplus.sh --insecure https://gitlab.com/rwkgyg/CFwarp/-/raw/main/point/warpplus.sh >/dev/null 2>&1
elif [[ $cpu = arm64 ]]; then
curl -sSL -o warpplus.sh --insecure https://gitlab.com/rwkgyg/CFwarp/-/raw/main/point/warpplusa.sh >/dev/null 2>&1
fi
chmod +x warpplus.sh
timeout 60s ./warpplus.sh
...
```

这一部分代码和`wpplusreplit.sh`所展示的下载地址相同，由于笔者使用的是arm架构操作系统，因此下载新的`warpplusa.sh`，上述代码证明它的功能和`warpplus.sh`相同。

## warpplus.sh

> 这是一个Linux系统下的可执行文件，在[文件](#文件)中已经可以看出来，因此本节围绕此文件展开分析。

### 查看基本信息

使用Detect it easy检查一下此文件的详细信息，可惜并没有能得到什么有价值的内容。只能看出这个可执行文件是由GCC编译来的，语言可能是C/C++。

![die](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.28.01.ji9fj7ct4nk.webp)

另外可以清晰看出，这个可执行程序内包含了python的一些代码，但这整个程序似乎都只是一个压缩包，需要解压缩。

![py](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.34.35.4jat5v0uops0.webp)

![packed](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.37.42.28nx0l4s89ts.webp)

### pydata

如下图所示，这个巨大的pydata节就是此程序要解压的对象，将它转储出来之后也确实可以识别出它是zlib压缩后的文件，但笔者并没有尝试去解压缩分析这个巨大的文件内容，事实证明这是对的，后文将会提及原因。

![pydata](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.39.23.3tsqcacobdz4.webp)

### 网络分析

在笔者分析网络之前，也使用strace追踪了它调用的系统API，但是并没有得到什么有价值的信息，因此这些内容省略，但是strace的记录日志在附件中一并提供。

既然这个程序是使用网络请求获得WARP无限流量Key的，那么即使不看代码，只分析网络请求我们也能将原理猜到。

按照这个思路，首先安装了`burpsuite`。不幸的是，此软件将需要用到的证书内置其中，`burpsuite`的证书无法用于解密它的流量。

在运行过程中，笔者发现它会在`/tmp`文件夹下释放python运行环境库，这些文件共计14M。这也解释了上文的`pydata`中的内容。

至此，网络分析只剩下导出客户端密钥这一条路可以走。

1. 设置环境变量，在当前tty中导出SSLKEY到`WarpPlus-TrafficCapture-Key.log`文件中

   ```sh
   export SSLKEYLOGFILE=WarpPlus-TrafficCapture-Key.log
   ```

2. 使用root权限打开wireshark，监听网卡
3. 在当前tty中运行`warpplus.sh`，等待它退出后停止wireshark监听
4. 在wireshark中设置`TLS-(Pre)-Master-Secret log filename`为上文导出的文件，如题所示。
   ![wireshark-master-secret](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-14.56.45.31guan99fpog.webp)
5. 在wireshark中打开http流分析，转储到文件中，内容请[点击这里查看](#warpplus-trafficcapture-httpstream)

### 剧透

> 2023年10月10日增补内容

根据[How to get free 12PB to Your Warp Key [BUG / BUFF]](https://telegra.ph/How-to-get-free-12PB-to-Your-Warp-Key-BUG--BUFF-04-09)所描述，符合以下操作就可以将空账户刷入另一个账户的推荐数量。

1. 备份当前设备所拥有的license。
2. 使用一个账户中已有极多推荐数量的license替换到当前设备中。
3. 重新启动WARP程序。
4. 重新将曾经备份到license替换到当前设备中。

这也解释了下文的请求过程，Cheers！

### 分析过程

**为了您的脑子健康，我直接将下文的总结写在这里，本文到此结束。您可以不用看后文的详情。**

1. 客户端请求：`POST /v0a2223/reg`。解释：请求创建一个Cloudflare WARP的A账户。
2. 服务端响应：JSON数据。解释：A账户目前是空账户。返回了代表客户端的设备ID和账户license。
3. 客户端请求：`POST /v0a2223/reg`。解释：请求创建一个Cloudflare WARP的B账户。
4. 服务端响应：JSON数据。解释：B账户目前是空账户。返回了代表客户端的设备ID和账户license。
5. 客户端请求：`PATCH /v0a2223/reg/v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739`，携带的参数`963be82f-4a90-45a8-aaac-be558383fe44`是第4步中，B账户代表客户端的设备ID。解释：由于`0b831bf3-224d-4d45-869b-b59edd27e739`是A账户代表客户端的设备ID，因此这次请求中A账户成为了B账户的引荐人。
6. 服务端响应：对比首次返回数据，删除`"usage":0`，增加`"referrer":"963be82f-4a90-45a8-aaac-be558383fe44"`。
7. 客户端请求：`DELETE /v0a2223/reg/963be82f-4a90-45a8-aaac-be558383fe44`。解释：删除账户B链接的设备`963be82f-4a90-45a8-aaac-be558383fe44`。
8. 服务端响应：`204 No Content`。解释：操作成功。
9. 客户端请求：`PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`，携带参数为`{"license": "u7SOF218-6zQ092Od-95qVJ02k"}`。解释：更改A账户所链接的设备`0b831bf3-224d-4d45-869b-b59edd27e739`的license为`u7SOF218-6zQ092Od-95qVJ02k`。
10. 服务端响应：`"id": "2b4d3261-ad36-4069-95ea-53520cd42a58"`。解释：授予此设备新的设备ID。
11. 客户端请求：`PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`，携带参数为`{"license": "7O95xNh3-81E65Kce-W4ln892I"}`。解释：更改无WARP账户链接的设备`0b831bf3-224d-4d45-869b-b59edd27e739`的license为`7O95xNh3-81E65Kce-W4ln892I`。
12. 服务端响应：`"id": "2b4d3261-ad36-4069-95ea-53520cd42a58","referral_count": 24598563`。解释：Cloudflare服务端似乎出现异常，无账户链接的设备`0b831bf3-224d-4d45-869b-b59edd27e739`引荐数量变成`24598563`。
13. 客户端请求：`GET /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`。解释：请求此无WARP账户链接的设备详细资料。
14. 服务端响应：`"premium_data": 24598563000000000,"quota": 24598563000000000,"referral_count": 24598563`。解释：Cloudflare服务端似乎出现异常，一个无限WARP流量账户创建成功。
15. 客户端请求：`DELETE /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739`。解释：删除此无WARP账户链接的设备。
16. 服务端响应：`204 No Content`。解释：操作成功。

### WarpPlus TrafficCapture HTTPStream

```log
POST /v0a2223/reg HTTP/1.1
Host: api.cloudflareclient.com
Content-Length: 0
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:13 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 813492d808d996c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-cache-set: true
x-envoy-upstream-service-time: 625
Set-Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=; path=/; expires=Mon, 09-Oct-23 07:00:13 GMT; domain=.cloudflareclient.com; HttpOnly; Secure; SameSite=None
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=%2F8Gdnjhpm99lkfYYkpf9I%2B4wMlEDZtiQuhUK1tBiAk7TwjKtvEnrjr9fBFDSEja3O4YSdnT%2FHR2JFKbsW1zm7ITkDMUZDI8kwyTLScq8fvuHdpTSXyIN9aME%2BZAVjVxWRr5gdOXuMMfCcQ%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"0b831bf3-224d-4d45-869b-b59edd27e739","type":"a","name":"","account":{"id":"b55baa96-a450-4332-9f39-f3046b6ea86a","account_type":"free","created":"2023-10-09T06:30:12.823920679Z","updated":"2023-10-09T06:30:12.823920679Z","premium_data":0,"quota":0,"usage":0,"warp_plus":true,"referral_count":0,"referral_renewal_countdown":0,"role":"child","license":"7O95xNh3-81E65Kce-W4ln892I"},"token":"40ca084f-315e-45e2-9191-b59199c2331a","warp_enabled":false,"waitlist_enabled":false,"created":"2023-10-09T06:30:12.509754043Z","updated":"2023-10-09T06:30:12.509754043Z","place":0,"locale":"zh-CN","enabled":true,"install_id":""}



POST /v0a2223/reg HTTP/1.1
Host: api.cloudflareclient.com
Content-Length: 0
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:14 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 813492de4daa96c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-cache-set: true
x-envoy-upstream-service-time: 803
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=rQULnzZb4N3TSVT6j8frTuQE01%2FJ4o58lJs4h9ROd%2F5EUN%2BhmxsjB%2BgU3%2B60wkEvCqNngwsddsIej6cIqS1nnZlTgM4mSFAvbSYwY4UurL3GOdMqNdqHCmpHkysd5UaV5l0uz3BCCNivKQ%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"963be82f-4a90-45a8-aaac-be558383fe44","type":"a","name":"","account":{"id":"ec26bf68-baa5-445a-b6b3-c8fb628509a2","account_type":"free","created":"2023-10-09T06:30:13.899453227Z","updated":"2023-10-09T06:30:13.899453227Z","premium_data":0,"quota":0,"usage":0,"warp_plus":true,"referral_count":0,"referral_renewal_countdown":0,"role":"child","license":"2W07lo5d-90c2f7TE-sK7Tq803"},"token":"379fc773-f8ab-45b2-a7cd-672dbf72e748","warp_enabled":false,"waitlist_enabled":false,"created":"2023-10-09T06:30:13.501458226Z","updated":"2023-10-09T06:30:13.501458226Z","place":0,"locale":"zh-CN","enabled":true,"install_id":""}



PATCH /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739 HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Content-Type: application/json; charset=UTF-8
Authorization: Bearer 40ca084f-315e-45e2-9191-b59199c2331a
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=
Content-Length: 52

{"referrer": "963be82f-4a90-45a8-aaac-be558383fe44"}



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:16 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 813492e5ebdc96c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-cache-set: true
x-envoy-upstream-service-time: 1608
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=ftYufwYVFe9blsUVXEH%2Bx7GGcyvLZDeToO7yhv1zKvkVerVw4By89zAfkP58xBnwWinHJLlybAoipaUM7yej7lmQhlUHLV8KzAXfORubkpvtrxizutxWESQu8m9DfE0AcTJU%2F9%2FnNt7AOw%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"0b831bf3-224d-4d45-869b-b59edd27e739","type":"a","name":"","account":{"id":"b55baa96-a450-4332-9f39-f3046b6ea86a","account_type":"free","created":"2023-10-09T06:30:12.82392Z","updated":"2023-10-09T06:30:12.82392Z","premium_data":0,"quota":0,"warp_plus":true,"referral_count":0,"referral_renewal_countdown":0,"role":"child","license":"7O95xNh3-81E65Kce-W4ln892I"},"warp_enabled":false,"waitlist_enabled":false,"created":"2023-10-09T06:30:12.509754Z","updated":"2023-10-09T06:30:14.909577261Z","place":0,"locale":"zh-CN","enabled":true,"install_id":"","referrer":"963be82f-4a90-45a8-aaac-be558383fe44"}



DELETE /v0a2223/reg/963be82f-4a90-45a8-aaac-be558383fe44 HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Authorization: Bearer 379fc773-f8ab-45b2-a7cd-672dbf72e748
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=



HTTP/1.1 204 No Content
Date: Mon, 09 Oct 2023 06:30:17 GMT
Connection: keep-alive
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=Cp6WP7S5U1wpURnPmkVX9o5wdGZj0DpZF6i7XdLAoVmC%2BTMjdFRuU4r1uRXpPJgD9qtRI9g2FEuV4c9vO%2F%2BPDaCb6GQ%2F0gfQBLOC2gCoLXFXYSiE3C%2Fm82%2Bcli8PCFsavtchA6D3VumK4g%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Vary: Accept-Encoding
Server: cloudflare
CF-RAY: 813492f20d3796c0-SJC



PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Content-Type: application/json; charset=UTF-8
Authorization: Bearer 40ca084f-315e-45e2-9191-b59199c2331a
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=
Content-Length: 41

{"license": "u7SOF218-6zQ092Od-95qVJ02k"}



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:21 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 813492f7c9a696c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-envoy-upstream-service-time: 4098
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=Qy8emNDUPCJGcbD5y2yy04auosZBfIsuUKdFzv13Qu6YVCHItE%2BWjBtf6G2NBL9UcW2uaLGM6C0RJzmLg80ITaN0yKA%2BmiGPRBxU70VOfJIr1aZo7jurHbycP%2FiaKrc5tD2VBeOaesI90A%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"2b4d3261-ad36-4069-95ea-53520cd42a58","created":"0001-01-01T00:00:00Z","updated":"2023-10-09T06:30:20.041792568Z","premium_data":0,"quota":0,"warp_plus":true,"referral_count":0,"referral_renewal_countdown":0,"role":"child"}



PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Content-Type: application/json; charset=UTF-8
Authorization: Bearer 40ca084f-315e-45e2-9191-b59199c2331a
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=
Content-Length: 41

{"license": "7O95xNh3-81E65Kce-W4ln892I"}



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:23 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 81349313896196c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-envoy-upstream-service-time: 1211
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=0drwc3hjXQSCR2MN19A9gx%2BF3frLTlL76tvd1viYTqf3tTU0Js%2FdM3eoyqU%2FL0py%2FoPJYAJsZTemfGo%2BYsEuVLddoeEJ%2BzaE9RECqldUdGpuskVjquX3RMqoD5UtMAWdXsjtiR5mB7Wagw%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"b55baa96-a450-4332-9f39-f3046b6ea86a","created":"0001-01-01T00:00:00Z","updated":"2023-10-09T06:30:23.027025975Z","premium_data":24598563000000000,"quota":24598563000000000,"warp_plus":true,"referral_count":24598563,"referral_renewal_countdown":0,"role":"child"}



GET /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Authorization: Bearer 40ca084f-315e-45e2-9191-b59199c2331a
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=



HTTP/1.1 200 OK
Date: Mon, 09 Oct 2023 06:30:23 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
CF-Ray: 8134931d99d396c0-SJC
CF-Cache-Status: DYNAMIC
Vary: Accept-Encoding
x-envoy-upstream-service-time: 199
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=UUOYPQkNQWPKUfe6SGnsX8qgZ4yYrd%2FvoiX83NpDNYALCUunDdWYN9NL6SQHLhxzSCv%2FRyPWDEIL7q16wtOGVWrR39hbCZsKH4IA%2FFObYgmb7C6m0HHC3eCQxvn6eqxV9L77U7T8rumrBQ%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Server: cloudflare
Content-Encoding: gzip

{"id":"b55baa96-a450-4332-9f39-f3046b6ea86a","account_type":"limited","created":"2023-10-09T06:30:12.82392Z","updated":"2023-10-09T06:30:23.027025Z","premium_data":24598563000000000,"quota":24598563000000000,"warp_plus":true,"referral_count":24598563,"referral_renewal_countdown":0,"role":"child","license":"7O95xNh3-81E65Kce-W4ln892I"}



DELETE /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739 HTTP/1.1
Host: api.cloudflareclient.com
Accept: */*
CF-Client-Version: a-6.11-2223
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.12.1
Authorization: Bearer 40ca084f-315e-45e2-9191-b59199c2331a
Cookie: __cf_bm=Fy2G.5O4ravUG7amYkVN3pFXyXviJwPUOkSrgy47_so-1696833013-0-AcoPy8M7MrOFuI+c74qOYNs875Cp/we0MEq6M/wdlySHgOigyzY5f3+52sIv12h9pB+WV55CalcoCwYkuN3VQSo=



HTTP/1.1 204 No Content
Date: Mon, 09 Oct 2023 06:30:25 GMT
Connection: keep-alive
Report-To: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v3?s=0xOQ2ahaxjRl6A1krBHDAP33evulwaakRrFst09ItQ%2BIsWyQ7rkeTFyNhYd39BMFXc2wR6ULCRtVuRhbY8BhXZGv1XegNl%2Bm6d2%2Fz7%2FkMX6LhBtAJYb9Bz1GGdooOhWiKUy3GbIlK3bDjw%3D%3D"}],"group":"cf-nel","max_age":604800}
NEL: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
Vary: Accept-Encoding
Server: cloudflare
CF-RAY: 813493218d1396c0-SJC
```

## 总结

最后，感谢您的耐心观看，愿您生活愉悦。

## 参考文献

1. [yonggekkk/warp-yg - Github](https://github.com/yonggekkk/warp-yg)
2. [WarpKey-Register-PRO - Replit](https://replit.com/@ygkkkk/WarpKey-Register-PRO)
3. [rwkgyg/CFwarp - Gitlab](https://gitlab.com/rwkgyg/CFwarp)
4. [How to get free 12PB to Your Warp Key [BUG / BUFF]](https://telegra.ph/How-to-get-free-12PB-to-Your-Warp-Key-BUG--BUFF-04-09)
5. [WarpKeyGen - Replit](https://replit.com/@SerdarAD/WarpKeyGen)
