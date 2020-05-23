# CocoaPods

## 创建自己的 Pod 库

### 注册 Session

```shell
$ pod trunk register 0xffdirtmelon@gmail.com 'dirtmelon' --verbose
opening connection to trunk.cocoapods.org:443...
opened
starting SSL for trunk.cocoapods.org:443...
SSL established, protocol: TLSv1.2, cipher: ECDHE-RSA-AES128-GCM-SHA256
<- "POST /api/v1/sessions HTTP/1.1\r\nContent-Type: application/json; charset=utf-8\r\nAccept: application/json; charset=utf-8\r\nUser-Agent: CocoaPods/1.9.1\r\nAccept-Encoding: gzip;q=1.0,deflate;q=0.6,identity;q=0.3\r\nHost: trunk.cocoapods.org\r\nContent-Length: 73\r\n\r\n"
<- "{\"email\":\"0xffdirtmelon@gmail.com\",\"name\":\"dirtmelon\",\"description\":null}"
-> "HTTP/1.1 201 Created\r\n"
-> "Date: Sat, 23 May 2020 09:35:58 GMT\r\n"
-> "Connection: keep-alive\r\n"
-> "Strict-Transport-Security: max-age=31536000\r\n"
-> "Content-Type: application/json\r\n"
-> "Content-Length: 194\r\n"
-> "X-Content-Type-Options: nosniff\r\n"
-> "Server: thin 1.6.2 codename Doc Brown\r\n"
-> "Via: 1.1 vegur\r\n"
-> "\r\n"
reading 194 bytes...
-> "{\"created_at\":\"2020-05-23 09:35:58 UTC\",\"valid_until\":\"2020-09-28 09:35:58 UTC\",\"verified\":false,\"created_from_ip\":\"129.227.148.59\",\"description\":null,\"token\":\"3f29f5199f69303f4378635511c2f53b\"}"
read 194 bytes
Conn keep-alive
```

然后会在收到一封验证的邮件，点击链接进行验证。

### 验证是否成功注册

```shell
$ pod trunk me
  - Name:     dirtmelon
  - Email:    0xffdirtmelon@gmail.com
  - Since:    May 23rd, 03:35
  - Pods:     None
  - Sessions:
    - May 23rd, 03:35 - September 28th, 03:36. IP: 129.227.148.59
```

提示已经注册成功。

### 创建对应的 Git 库
输入 `pod lib create Name` 创建对应的仓库。根据自己的需要对 `.podspec` 文件进行编辑，编辑完成后进行验证：

```shell
$ pod lib lint

 -> FaviconMan (0.1.0)
    - NOTE  | xcodebuild:  note: Using new build system
    - NOTE  | xcodebuild:  note: Building targets in parallel
    - NOTE  | xcodebuild:  note: Planning build
    - NOTE  | xcodebuild:  note: Constructing build description
    - NOTE  | xcodebuild:  warning: Skipping code signing because the target does not have an Info.plist file and one is not being generated automatically. (in target 'App' from project 'App')
    - NOTE  | xcodebuild:  note: Execution policy exception registration failed and was skipped: Error Domain=NSPOSIXErrorDomain Code=1 "Operation not permitted" (in target 'FaviconMan' from project 'Pods')
    - NOTE  | xcodebuild:  note: Execution policy exception registration failed and was skipped: Error Domain=NSPOSIXErrorDomain Code=1 "Operation not permitted" (in target 'Pods-App' from project 'Pods')
    - NOTE  | xcodebuild:  note: Execution policy exception registration failed and was skipped: Error Domain=NSPOSIXErrorDomain Code=1 "Operation not permitted" (in target 'App' from project 'App')
```

`pod lib lint` 通过以后需要添加 `tag` 
```shell
git -tag - a 0.1.0 -m '填写更新信息'
```

然后可以提交 pod ：

```shell
pod trunk push
```

执行成功后会有相关的信息提示，之后别人就可以通过 `CocoaPods` 使用你的库。