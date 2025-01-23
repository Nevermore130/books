### 术语 ###

WebAuthn：WebAuthn是Web Authentication API的缩写，它主要就是支持Passkey的浏览器API，由W3C和FIDO维护

为了让浏览器支持Passkey，WebAuthn定义了两个JavaScript API：



使用Passkey登录时，**网站存储了用户的公钥，而用户的私钥存放在本地设备（电脑或手机）中，通过给服务器发送私钥签名，服务器验证签名无误，即登录成功**

1. 检测当前设备是否支持 Passkey，是则继续

2. 点击给账号添加 Passkey，浏览器向网站服务器发送请求，**获取创建 Passkey 所需要的 options (网站信息, 用户名，网站地址等)**

3. 浏览器调用 [Web Authentication API](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FWeb_Authentication_API) 也就是 `navigator.credentials.create()` 方法并传入上一步的 options 参数来要求用户进行身份验证以创建密钥，通过设备验证(比如FaceID)后生成一对私钥和公钥，随后会将 **私钥，用户名以及网站地址作为 Passkey 存储在设备中**(可以是手机芯片,U盘)，**并将公钥发送到对应网站**。网站服务端收到公钥和用户名，存入数据库中，Passkey 凭据创建成功。

4. 当下次您使用 Passkey 登录该服务时，浏览器向服务器发起请求，获取凭据认证所需的 options 信息，浏览器调用 `navigator.credentials.get()` 方法，传入上一步获取的 options ，并将**使用与您的帐户绑定的公钥**为您的设备**创建随机质询(challenge)**。(即用服务端保存的公钥来加密challenge)

5. 此时，浏览器调用操作系统接口弹出对话框要求用户选择进行身份验证的密钥并进行身份验证，随后利用存储在您设备上的私钥解决该 challenge，验证身份成功。

   









### 创建Key ###

通过调用`navigator.credentials.create()`创建一个Passkey

```javascript
let credential = await navigator.credentials.create({
    publicKey: publicKeyCredentialCreationOptions
});

```

### 签名 ###

通过调用`navigator.credentials.get()`获取一个带签名的认证信息：

```javascript
let credential = await navigator.credentials.get({
    publicKey: publicKeyCredentialGetOptions
});

//这两个API都是浏览器自带的，可以用以下代码判断浏览器是否支持Passkey：
if (!navigator.credentials) {
    // 浏览器不支持Passkey
}

```

### 注册流程 ###

尽管可以让用户直接通过Passkey注册，但考虑到绝大多数网站已经有了完善的注册体系，更现实的方案是**让用户先以传统方式登录网站，然后向网站注册Passkey，后续登录时，即可使用Passkey登录。**

第一步：浏览器向服务器发送一个请求，要求获取创建Passkey的相关参数。假定服务器路径`https://example.com/create/options`，服务器返回的JSON如下

```json
{
    "challenge": "gVQ2n5FCAcksuEefCEgQRKJB_xfMF4rJMinTXSP72E8",
    "rp": {
        "name": "Passkey Example",
        "id": "example.com"
    },
    "user": {
        "id": "GOVsRuhMQWNoScmh_cK02QyQwTolHSUSlX5ciH242Y4",
        "name": "Michael",
        "displayName": "Michael"
    },
    "pubKeyCredParams": [
        {
            "alg": -7,
            "type": "public-key"
        }
    ],
    "timeout": 60000,
    "attestation": "none",
    "excludeCredentials": [
    ],
    "authenticatorSelection": {
        "authenticatorAttachment": "platform",
        "requireResidentKey": true,
        "residentKey": "required"
    },
    "extensions": {
        "credProps": true
    }
}

```

凡是涉及到bytes数组传输的，编码格式一律为Base64/URLSafe/无Padding模式。几个重要的字段如下：

* `challenge`是服务器发送的一个防重放的安全随机数，建议20～32字节
* `rp`是Relying Party的缩写，代表网站本身，`name`是网站名字，`id`是网站域名（不带端口号）；
* `user`是服务器返回的当前用户信息，`id`是用户在该网站的唯一ID，`name`和`displayName`都是用户名字，仅用于显示
* `pubKeyCredParams`是服务器支持的非对称签名算法，最常用的算法是`-7`，表示用ECDSA/SHA-256签名
* `timeout`是Challenge的有效时间，一般设定为60秒；
* `authenticatorAttachment`指示验证身份时，允许的Passkey来源，指定`platform`表示当前系统本身，还可以指定允许使用USB Key、NFC等外部验证

在浏览器中，**获取到创建参数后，还必须先把Base64编码的部分还原为`Uint8Array`，然后创建一个`Credential`对象**，并将此对象发送给服务器验证：

```javascript
// 请求参数:
const options = await get_json('/create/options');
console.log(options);

// 用Base64 URLSafe解码:
options.challenge = base64_urlsafe_decode(options.challenge);
options.user.id = base64_urlsafe_decode(options.user.id);

// 创建Credential,其中Private Key存储在操作系统的密钥管理器中,JavaScript不能获取Private Key:
const cred = await navigator.credentials.create({
    publicKey: options
});
console.log(cred);

// 用Base64 URLSafe编码:
const credential = {
    id: cred.id,
    rawId: base64_urlsafe_encode(cred.rawId),
    type: cred.type,
    response: {
        clientDataJSON: base64_urlsafe_encode(cred.response.clientDataJSON),
        attestationObject: base64_urlsafe_encode(cred.response.attestationObject),
        transports: cred.response.getTransports ? cred.response.getTransports() : []
    }
};
if (cred.authenticatorAttachment) {
    credential.authenticatorAttachment = cred.authenticatorAttachment;
}
console.log(credential);

// 发送Credential至服务器:
let createResult = await post_json('/register', credential);

```

### 注册Public Key ###

在服务器端，接收到Credential对象后，先把URL编码的字段还原为bytes数组，然后解码`clientDataJSON`和`attestationObject`两个字段

从`clientDataJSON`可以解出一个JSON，需要验证：

* `type`：必须为`webauthn.create`；
* `origin`：必须与域名一致，如`https://example.com`；
* `challenge`：必须与服务器保存的`challenge`一致。

从`attestationObject`可以解出

* `rpId`：域名，如`example.com`；
* `credentialId`：客户端传过来的一个标识符，用于标识Passkey；
* 以bytes数组表示的公钥，具体长度取决于签名算法。

服务器验证了必要的字段后，就可以将`credentialId`和公钥与当前用户关联起来，例如，存在`passkey_auths`表中：

user_id      **credential_id **     **public_key**
123456       jKeuFIcRgx5R1   pQECAyYgAS...YtHkpnD7k4

### 登录 ###

在登录页，此时用户尚未登录，服务器不知道用户身份。用户选择“使用Passkey一键登录”后，浏览器首先向服务器请求参数，服务器返回如下：

```json
{
    "challenge": "x1wRuShyI4k7BqYJi60kVk-clJWsPnBGgh_7z-W9QYk",
    "allowCredentials": [],
    "timeout": 60000,
    "rpId": "example.com"
}

```

* `challenge`：防重放的安全随机数；
* rpId`：域名，如`example.com
* `allowCredentials`：允许使用的Passkey列表，这里为空，表示由浏览器自己选择

在浏览器端，调用`navigator.credentials.get()`获取一个包含签名的对象：

```javascript
// 请求参数:
const options = await get_json('/get/options');
console.log(options);

// 用Base64 URLSafe解码:
options.challenge = base64_urlsafe_decode(options.challenge);

// 创建签名:
const cred = await navigator.credentials.get({
    publicKey: options
});
console.log(cred);

// 用Base64 URLSafe编码:
const credential = {
    id: cred.id,
    rawId: base64_urlsafe_encode(cred.rawId),
    type: cred.type,
    response: {
        clientDataJSON: base64_urlsafe_encode(cred.response.clientDataJSON),
        authenticatorData: base64_urlsafe_encode(cred.response.authenticatorData),
        userHandle: base64_urlsafe_encode(cred.response.userHandle),
        signature: base64_urlsafe_encode(cred.response.signature)
    }
};
console.log(credential);

// 发送Credential至服务器:
let createResult = await post_json('/signin', credential);

```

* authenticatorData`可以解出一个`rpIdHash,用于验证域名
* `userHandle`是注册时服务器传递给浏览器的用户ID，服务器根据这个字段获取到用户ID，再查表获取到对应的Public Key
* 再用Public Key验证`signature`签名字段,如果验证通过，则服务器成功获取到用户ID，完成用户身份认证。

浏览器不关心服务器端的用户ID是如何表示的，因为浏览器拿到的用户ID是Base64/URLSafe编码的bytes数组。服务器使用`int`存储用户ID，就写一个`int2bytes()`和`bytes2int()`来转换，服务器使用`String`存储用户ID时，就写一个`string2bytes()`和`bytes2string()`来转换





