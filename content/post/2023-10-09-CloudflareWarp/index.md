---
title: Cloudflare Warp Trace
description: Cloudflare's Warp service seems to be easily scrubbed to unlimited, and this article aims to explore the principles.
date: 2023-10-09 14:29:43+0800
categories:
    - ReverseAnalyze
tags:
    - Cloudflare
    - Warp
image: https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/webpageheadimageCloudflare_54117147.6hbl4terqlj4.webp
---

Source of this article: [MoeomuBlog](/posts/cloudflare-warp-trace/)

> All code and related files mentioned in this article can be downloaded from this link [cloudflare-warp-analyze.zip](./cloudflare-warp-analyze.zip).

## Trigger

While wandering around on the Internet, I found a project like this: [Generate more than 20 million GB of warp+ keys in one click, you know](https://replit.com/@ygkkkkk/WarpKey-Register-PRO). Whether you get it or not, the author didn't get it anyway. Why does it generate so many keys? Maybe there's a list and then it randomly draws one each time it runs?

## Files

Now that I'm here, it's not right not to fork it. There are a couple of files here, the first one is `replit.nix` which states that the entry point for this project is `wpplusreplit.sh`, so let's start with that.

```text
> file *
README.md:       UTF-8 text
replit.nix:      ASCII text
warpplus.sh:     ELF executable, 64-bit LSB x86-64, dynamic (/lib64/ld-linux-x86-64.so.2), BuildID=3dfe013b3027047470d1aac10a3504baf6969725, stripped
wpplusreplit.sh: ASCII text
```

## wpplusreplit.sh

The `wpplusreplit.sh` code is obfuscated, unsurprisingly.

![Fork](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/1-fork-min.1wiu2onj1if4.webp)

It's just a simple variable name obfuscation, and a bit of common sense would tell you to debug the code with `bash -x`, and the result is as follows:

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

Continuing with the base64 decoding, you end up with the following code:

```sh
#!/bin/bash
echo
echo "请稍等，下载更新中……"
rm -rf warpplus.sh
wget -N https://gitlab.com/rwkgyg/CFwarp/-/raw/main/point/warpplus.sh >/dev/null 2>&1
chmod +x warpplus.sh
./warpplus.sh
```

What? Huh? What? What? It's just updating and launching the `warpplus.sh` file, right?

## CFwarp.sh

The above mentions a project [rwkgyg/CFwar](https://gitlab.com/rwkgyg/CFwar) which seems to be where the author of this script stores these scripts and updates. The CFwarp.sh found here was decrypted again using the above method to get the following code:

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

This part of the code has the same download address as shown in `wpplusreplit.sh`, and since the author is using an arm architecture operating system, he downloads the new `warpplusa.sh`, and the above code proves that it functions in the same way as `warpplus.sh`.

## warpplus.sh

> This is an executable file under Linux, as can already be seen in [Files](#files), so this section is centered around analyzing this file.

### Check basic information

Using Detect it easy to check the details of this file unfortunately didn't yield anything of value. The only thing I can tell is that this executable was compiled by GCC and the language is probably C/C++.

![die](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.28.01.ji9fj7ct4nk.webp)

It is also clear that this executable contains some python code, but this entire program seems to be just a zip package that needs to be unzipped.

![py](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.34.35.4jat5v0uops0.webp)

![packed](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.37.42.28nx0l4s89ts.webp)

### pydata

As shown in the figure below, this huge pydata section is the object of this program to decompress, it will dump it out of the file can indeed be identified as zlib compressed file, but I did not try to decompress and analyze the contents of this huge file, which proved to be right, the reasons will be mentioned later.

![pydata](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-15.39.23.3tsqcacobdz4.webp)

### Network Analyse

Before I analyze the network, but also use strace to trace the system API it calls, but did not get any valuable information, so these contents are omitted, but strace logging logs are provided in the attachment together.

Since this program uses a network request to get the WARP Unlimited Traffic Key, we can guess the principle even without looking at the code and just analyzing the network request.

Along these lines, `burpsuite` was first installed. Unfortunately, this software has the required certificates built-in, and `burpsuite`'s certificates cannot be used to decrypt its traffic.

During the run, I found that it releases the python runtime environment libraries in the `/tmp` folder, which total 14 M. This explains the contents of `pydata` above.

At this point, the only thing left to do for network analysis is to export the client key.

1. Set the environment variable to export the SSLKEY in the current tty to the `WarpPlus-TrafficCapture-Key.log` file

   ```sh
   export SSLKEYLOGFILE=WarpPlus-TrafficCapture-Key.log
   ```

2. Open wireshark with root privileges and listen on the NIC.
3. run `warpPlus.sh` in the current tty and wait for it to exit to stop wireshark listening
4. Set `TLS-(Pre)-Master-Secret log filename` in wireshark to the file exported above, as shown in the question.
   ![wireshark-master-secret](https://cdn.statically.io/gh/Misakaou/imagestorage@master/20231009/Screenshot-2023-10-09-at-14.56.45.31guan99fpog.webp)
5. Open http stream analysis in wireshark, dump to file, content please [click here to view](#warpplus-trafficcapture-httpstream)

### Spoilers

> October 10, 2023 Addendum.

As described in [How to get free 12PB to Your Warp Key [BUG / BUFF]](https://telegra.ph/How-to-get-free-12PB-to-Your-Warp-Key-BUG--BUFF-04-09), conforming to the following actions is fine Swipe an empty account to the recommended amount of another account.

1. Backup the license you currently have on your device.
2. Replace the license in the current device with an account that already has a very large number of referrals.
3. Restart the WARP program.
4. Replace the backed up license with the current device.

This also explains the request process below, Cheers!

### Analyzing the process

**For the sake of your brain's health, I'm going to summarize the following directly here, where this article ends. You may not need to read the details afterward.**

1. Client request: `POST /v0a2223/reg`. Explanation: request to create an A account for Cloudflare WARP.
2. Server response: JSON data. Explanation: The A account is currently empty. The device ID and account license representing the client are returned.
3. Client request: `POST /v0a2223/reg`. Explanation: The request is to create a Cloudflare WARP account B.
4. Server response: JSON data. Explanation: The B account is currently empty. The device ID and account license representing the client are returned.
5. client request: `PATCH /v0a2223/reg/v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739`, carrying the parameter `963be82f-4a90-45a8-aaac-be558383fe44` which is the B account in step 4. representing the client's device ID.Explanation: since `0b831bf3-224d-4d45-869b-b59edd27e739` is the device ID of account A representing the client, account A becomes the referrer of account B in this request.
6. server-side response: comparing the data returned for the first time, remove `"usage":0` and add `"referrer": "963be82f-4a90-45a8-aaac-be558383fe44"`.
7. Client request: `DELETE /v0a2223/reg/963be82f-4a90-45a8-aaac-be558383fe44`. Explanation: deleting device `963be82f-4a90-45a8-aaac-be558383fe44` linked by account B.
8. Server response: `204 No Content`. Explanation: The operation was successful.
9. Client request: `PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`, carrying the parameter `{"license": "u7SOF218-6zQ092Od-95qVJ02k"}`. Explanation: Changing the license of the device `0b831bf3-224d-4d45-869b-b59edd27e739` linked to account A to `u7SOF218-6zQ092Od-95qVJ02k`.
10. Server response: `"id": "2b4d3261-ad36-4069-95ea-53520cd42a58"`. Explanation: new device ID granted to this device.
11. Client request: `PUT /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`, carrying the parameter `{"license": "7O95xNh3-81E65Kce-W4ln892I"}`. Explanation: change the license of device `0b831bf3-224d-4d45-869b-b59edd27e739` without WARP account link to `7O95xNh3-81E65Kce-W4ln892I`.
12. server-side response: `"id": "2b4d3261-ad36-4069-95ea-53520cd42a58", "referral_count": 24598563`. Explanation: there seems to be an anomaly on the Cloudflare server side where the number of referrals for the device `0b831bf3-224d-4d45-869b-b59edd27e739` linked without an account becomes `24598563`.
13. Client request: `GET /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739/account`. Explanation: Requesting device details for this no-WARP account link.
14. Server response: `"premium_data": 24598563000000000, "quota": 24598563000000000, "referral_count": 24598563`. Explanation: Cloudflare server seems to have an exception, an unlimited WARP traffic account was created successfully.
15. Client request: `DELETE /v0a2223/reg/0b831bf3-224d-4d45-869b-b59edd27e739`. Explanation: Remove this device with no WARP account link.
16. Server response: `204 No Content`. Explanation: The operation was successful.

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

## Conclusion

Finally, thank you for your patience, may you live a happy life.

## Referrence

1. [yonggekkk/warp-yg - Github](https://github.com/yonggekkk/warp-yg)
2. [WarpKey-Register-PRO - Replit](https://replit.com/@ygkkkk/WarpKey-Register-PRO)
3. [rwkgyg/CFwarp - Gitlab](https://gitlab.com/rwkgyg/CFwarp)
