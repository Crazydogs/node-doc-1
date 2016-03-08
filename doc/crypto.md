## 加密

<div class="s s2"></div>

`crypto` 模块封装了诸多的加密功能，包括 OpenSSL 的哈希、HMAC、加密、解密、签名和验证函数。

需要使用 `require('crypto')` 导入该模块：

```js
const crypto = require('crypto');

const secret = 'abcdefg';
const hash = crypto.createHmac('sha256', secret)
                   .update('I love cupcakes')
                   .digest('hex');
console.log(hash);
// 输出结果: c0fa1bc00531bd78ef38c628449c5102aeabd49b5dc3a2a516ea6ea959d6658e
```

## Class: Certificate

SPKAC 是由 Netscape 提出的一个证书签发请求机制（Certificate Signing Request mechanism），现在已经正式成为 HTML5 keygen 元素的一部分。

`crypto` 模块提供的 `Certificate` 类就是用于处理 SPKAC 数据，最常见的用处就是处理 HTML5 `<keygen>` 元素生成的数据。Node.js 在内部使用 [OpenSSL 所实现的 SPKAC](https://www.openssl.org/docs/apps/spkac.html)。

#### new crypto.Certificate()

通过 `new` 操作符或直接调用 `crypto.Certificate()` 方法都可以创建 `Certificate` 类的实例：

```js
const crypto = require('crypto');

const cert1 = new crypto.Certificate();
const cert2 = crypto.Certificate();
```

#### certificate.exportChallenge(spkac)

在 `spkac` 的数据结构中包含一个公钥和一个口令。`certificate.exportChallenge()` 方法会以 Node.js Buffer 的形式返回口令模块。这里的 `spkac` 参数可以字符串，也可以是 Buffer：

```js
const cert = require('crypto').Certificate();
const spkac = getSpkacSomehow();
const challenge = cert.exportChallenge(spkac);
console.log(challenge.toString('utf8'));
// Prints the challenge as a UTF8 string
```

#### certificate.exportPublicKey(spkac)

在 `spkac` 的数据结构中包含一个公钥和一个口令。`certificate.exportPublicKey()` 方法会以 Node.js Buffer 的形式返回公钥模块。这里的 `spkac` 参数可以字符串，也可以是 Buffer：

```js
const cert = require('crypto').Certificate();
const spkac = getSpkacSomehow();
const publicKey = cert.exportPublicKey(spkac);
console.log(publicKey);
// Prints the public key as <Buffer ...>
```

#### certificate.verifySpkac(spkac)

如果 `spkac` 的数据结构是有效地，则该方法返回 `true`，否则返回 `false`。这里的 `spkac` 参数必须是一个 Node.js 的 Buffer 实例：

```js
const cert = require('crypto').Certificate();
const spkac = getSpkacSomehow();
console.log(cert.verifySpkac(new Buffer(spkac)));
// Prints true or false
```

## Class: Cipher

`Cipher` 类的实例常用于对数据进行加密。使用该类主要有两种方式：

- 包装成可读写的 stream 实例，写入纯数据，读出加密数据
- 使用 `cipher.update()` 和 `cipher.final()` 方法生成加密数据

`crypto.createCipher()` 或 `crypto.createCipheriv()` 两个方法常用于创建 `Cipher` 类的实例。`Cipher` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Cipher` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const cipher = crypto.createCipher('aes192', 'a password');

var encrypted = '';
cipher.on('readable', () => {
  var data = cipher.read();
  if (data)
    encrypted += data.toString('hex');
});
cipher.on('end', () => {
  console.log(encrypted);
  // Prints: ca981be48e90867604588e75d04feabb63cc007a8f8ad89b10616ed84d815504
});

cipher.write('some clear text data');
cipher.end();
```

下面代码演示了如何将 `Cipher` 类的实例封装成管道流（piped stream)：

```js
const crypto = require('crypto');
const fs = require('fs');
const cipher = crypto.createCipher('aes192', 'a password');

const input = fs.createReadStream('test.js');
const output = fs.createWriteStream('test.enc');

input.pipe(cipher).pipe(output);
```

下面代码演示了如何使用 `cipher.update()` 和 `cipher.final()` 方法：

```js
const crypto = require('crypto');
const cipher = crypto.createCipher('aes192', 'a password');

var encrypted = cipher.update('some clear text data', 'utf8', 'hex');
encrypted += cipher.final('hex');
console.log(encrypted);
// Prints: ca981be48e90867604588e75d04feabb63cc007a8f8ad89b10616ed84d815504
```

#### cipher.final([output_encoding])

返回剩余的加密内容。如果 `output_encoding` 参数是 `binary` / `base64` 或 `hex` 中的一个，则返回字符串，否则返回 Buffer 实例。

一旦调用 `cipher.final()` 方法之后，`Cipher` 类的实例就不能再用于加密数据。多次调用 `cipher.final()` 方法将会抛出错误。

#### cipher.setAAD(buffer)

当使用加密认证模式（目前只支持 GCM）时，`cipher.setAAD()` 方法用于设置附加认证数据（AAD）的参数。

#### cipher.getAuthTag()

当使用加密认证模式（目前只支持 GCM）时，`cipher.getAuthTag()` 方法会返回一个 Buffer 实例，该实例包含由原始数据计算得来的验证标记（authentication tag）。

`cipher.getAuthTag()` 方法必须在 `cipher.final()` 方法完成加密操作之后再调用。

#### cipher.setAutoPadding(auto_padding=true)

当使用块加密算法时，`Cipher` 类将会自动为原始数据填充额外的内容，便于切割成指定大小的块。如果开发者希望禁用默认的数据填充行为，可以调用 `cipher.setAutoPadding(false)`。

如果 `auto_padding` 的值为 `false`，那么原始数据的大小必须是加密块的整数倍，否则 `cipher.final()` 方法将会抛出错误。禁用自动填充数据有助于实现非标准的填充，比如使用 `0x0` 而不是 PKCS 进行填充。

`cipher.setAutoPadding()` 方法必须在 `cipher.final()` 方法之后调用。

#### cipher.update(data[, input_encoding][, output_encoding])

该方法用于通过 `data` 更新加密数据。如果传入了 `input_encoding` 参数，则其值必须为 `utf8`、`ascii` 和 `binary` 中的一个，且 `data` 参数必须为指定编码格式的字符串数据。如果未指定 `input_encoding` 参数，则 `data` 参数必须是 Buffer 实例。如果已知 `data` 参数是 Buffer 实例，则 `input_encoding` 无论是何值都会被忽略。

`out_encoding` 参数制定了加密数据输出的编码格式，值为 `utf8`、`ascii` 和 `binary` 之一。如果指定了 `output_encoding` 参数，则返回与该编码格式相符的字符串；如果没有传入该参数，则返回一个 Buffer 实例。

在 `cipher.final()` 方法调用之前，可以多次调用 `cipher.update()` 方法。如果在 `cipher.final()` 之后调用 `cipher.update()`，则会抛出一个错误。

## Class: Decipher

`Decipher` 类的实例常用于对数据进行解密。使用该类主要有两种方式：

- 包装成可读写的 stream 实例，写入纯数据，读出加密数据
- 使用 `decipher.update()` 和 `decipher.final()` 方法生成解密数据

`crypto.createDecipher()` 或 `crypto.createDecipheriv()` 两个方法常用于创建 `Decipher` 类的实例。`Decipher` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Decipher` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const decipher = crypto.createDecipher('aes192', 'a password');

var decrypted = '';
decipher.on('readable', () => {
  var data = decipher.read();
  if (data)
  decrypted += data.toString('utf8');
});
decipher.on('end', () => {
  console.log(decrypted);
  // Prints: some clear text data
});

var encrypted = 'ca981be48e90867604588e75d04feabb63cc007a8f8ad89b10616ed84d815504';
decipher.write(encrypted, 'hex');
decipher.end();
```

下面代码演示了如何将 `Decipher` 类的实例封装成管道流（piped stream)：

```js
const crypto = require('crypto');
const fs = require('fs');
const decipher = crypto.createDecipher('aes192', 'a password');

const input = fs.createReadStream('test.enc');
const output = fs.createWriteStream('test.js');

input.pipe(decipher).pipe(output);
```

下面代码演示了如何使用 `decipher.update()` 和 `decipher.final()` 方法：

```js
const crypto = require('crypto');
const decipher = crypto.createDecipher('aes192', 'a password');

var encrypted = 'ca981be48e90867604588e75d04feabb63cc007a8f8ad89b10616ed84d815504';
var decrypted = decipher.update(encrypted, 'hex', 'utf8');
decrypted += decipher.final('utf8');
console.log(decrypted);
// Prints: some clear text data
```

#### decipher.final([output_encoding])

返回剩余的解密内容。如果 `output_encoding` 参数是 `binary` / `base64` 或 `hex` 中的一个，则返回字符串，否则返回 Buffer 实例。

一旦调用 `decipher.final()` 方法之后，`Decipher` 类的实例就不能再用于加密数据。多次调用 `decipher.final()` 方法将会抛出错误。

#### decipher.setAAD(buffer)

当使用加密认证模式（目前只支持 GCM）时，`decipher.setAAD()` 方法用于设置附加认证数据（AAD）的参数。

#### decipher.setAuthTag()

当使用加密认证模式（目前只支持 GCM）时，`decipher.setAuthTag()` 方法常用于传递接收到的验证标记（authentication tag）。如果没有提供任何标记，或者加密文本已被篡改，`decipher.final()` 方法将会抛出错误，用于提示开发者当前加密文本未通过验证必须被抛弃。

#### decipher.setAutoPadding(auto_padding=true)

如果原始数据加密时没有使用标准的块填充，可以调用 `decipher.setAuthPadding(false)` 方法禁用自动填充，从而避免 `decipher.final()` 校验和移除填充数据。

只有原始数据的长度是加密数据块的整数倍，才可以禁用数据的自动填充行为。

`decipher.setAutoPadding()` 方法必须在 `decipher.update()` 方法之前调用。

#### decipher.update(data[, input_encoding][, output_encoding])

该方法用于通过 `data` 更新解密数据。如果传入了 `input_encoding` 参数，则其值必须为 `utf8`、`ascii` 和 `binary` 中的一个，且 `data` 参数必须为指定编码格式的字符串数据。如果未指定 `input_encoding` 参数，则 `data` 参数必须是 Buffer 实例。如果已知 `data` 参数是 Buffer 实例，则 `input_encoding` 无论是何值都会被忽略。

`out_encoding` 参数制定了加密数据输出的编码格式，值为 `utf8`、`ascii` 和 `binary` 之一。如果指定了 `output_encoding` 参数，则返回与该编码格式相符的字符串；如果没有传入该参数，则返回一个 Buffer 实例。

在 `decipher.final()` 方法调用之前，可以多次调用 `decipher.update()` 方法。如果在 `decipher.final()` 之后调用 `decipher.update()`，则会抛出一个错误。

## Class: DiffieHellman

`DiffieHellman` 类封装了一系列函数，用于创建 Diffie-Hellman 秘钥交换协议。

通过调用 `crypto.createDiffieHellman()` 函数可以创建 `DiffieHellman` 类的实例：

```js
const crypto = require('crypto');
const assert = require('assert');

// Generate Alice's keys...
const alice = crypto.createDiffieHellman(11);
const alice_key = alice.generateKeys();

// Generate Bob's keys...
const bob = crypto.createDiffieHellman(11);
const bob_key = bob.generateKeys();

// Exchange and generate the secret...
const alice_secret = alice.computeSecret(bob_key);
const bob_secret = bob.computeSecret(alice_key);

assert(alice_secret, bob_secret);
// OK
```

#### diffieHellman.computeSecret(other_public_key[, input_encoding][, output_encoding])

该方法使用 `other_public_key` 作为第三方的弓腰计算共享密钥，最后返回计算后得到的共享密钥。`input_encoding` 参数指定公钥的编码格式，`output_encoding` 参数指定共享密钥的编码格式。编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果未指定 `input_encoding` 参数，则传入的 `other_public_key` 应为 Buffer 实例。

如果指定了 `output_encoding` 参数，则返回字符串，否则返回 Buffer 实例。

#### diffieHellman.generateKeys([encoding])

该方法用于生成 Diffie-Hellman 公钥和私钥的值，并根据指定的 `encoding` 格式返回公钥。该值必须传给第三方。编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### diffieHellman.getGenerator([encoding])

根据 `encoding` 参数指定的编码格式返回一个 Diffie-Hellman 的生成器，编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### diffieHellman.getPrime([encoding])

根据 `encoding` 参数指定的编码格式返回一个 Diffie-Hellman 的质数，编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### diffieHellman.getPrivateKey([encoding])

根据 `encoding` 参数指定的编码格式返回一个 Diffie-Hellman 的私钥，编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### diffieHellman.getPublicKey([encoding])

根据 `encoding` 参数指定的编码格式返回一个 Diffie-Hellman 的公钥，编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### diffieHellman.setPrivateKey(private_key[, encoding])

该方法用于设置 Diffie-Hellman 的私钥。如果指定了 `encoding` 的值为 `bianry`、`hex` 或 `base64` 中的一个，则 `private_key` 需要是字符串数据，否则 `private_key` 需要时 Buffer 实例。

#### diffieHellman.setPublicKey(public_key[, encoding])

该方法用于设置 Diffie-Hellman 的公钥。如果指定了 `encoding` 的值为 `bianry`、`hex` 或 `base64` 中的一个，则 `public_key` 需要是字符串数据，否则 `public_key` 需要时 Buffer 实例。

#### diffieHellman.verifyError

该属性包含了 `DiffieHellman` 对象初始化时所产生的所有警告和错误。

该属性的有效值（在 `constants` 模块中定义）有以下几种：

- DH_CHECK_P_NOT_SAFE_PRIME
- DH_CHECK_P_NOT_PRIME
- DH_UNABLE_TO_CHECK_GENERATOR
- DH_NOT_SUITABLE_GENERATOR

## Class: ECDH

`ECDH` 类封装了一系列函数，用于创建 Elliptic Curve Diffie-Hellman（ECDN）密钥交换协议。

通过调用 `crypto.createECDH()` 方法可以创建 `ECDH` 类的实例：

```js
const crypto = require('crypto');
const assert = require('assert');

// Generate Alice's keys...
const alice = crypto.createECDH('secp521r1');
const alice_key = alice.generateKeys();

// Generate Bob's keys...
const bob = crypto.createECDH('secp521r1');
const bob_key = bob.generateKeys();

// Exchange and generate the secret...
const alice_secret = alice.computeSecret(bob_key);
const bob_secret = bob.computeSecret(alice_key);

assert(alice_secret, bob_secret);
// OK
```

#### ecdh.computeSecret(other_public_key[, input_encoding][, output_encoding])

该方法使用 `other_public_key` 作为第三方的弓腰计算共享密钥，最后返回计算后得到的共享密钥。`input_encoding` 参数指定公钥的编码格式，`output_encoding` 参数指定共享密钥的编码格式。编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果未指定 `input_encoding` 参数，则传入的 `other_public_key` 应为 Buffer 实例。

如果指定了 `output_encoding` 参数，则返回字符串，否则返回 Buffer 实例。

#### ecdh.generateKeys([encoding[, format]])

该方法用于生成 Diffie-Hellman 公钥和私钥的值，并根据指定的 `encoding` 和 `format` 返回公钥。该值必须传给第三方。

`format` 参数指定了编码格式，值为 `compressed`、`uncompressed` 或 `hybrid` 中的一个。如果没有指定 `format` 参数，则默认使用 `compressed` 格式。

`encoding` 参数的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。 

#### ecdh.getPrivateKey([encoding])

根据 `encoding` 参数指定的编码格式返回一个 EC Diffie-Hellman 的私钥，编码格式的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。

#### ecdh.getPublicKey([encoding[, format]])

该方法根据 `encoding` 和 `format` 参数返回一个 EC Diffie-Hellman 的公钥。

`format` 参数指定了编码格式，值为 `compressed`、`uncompressed` 或 `hybrid` 中的一个。如果没有指定 `format` 参数，则默认使用 `compressed` 格式。

`encoding` 参数的值为 `bianry`、`hex` 或 `base64` 中的一个。如果指定了 `encoding` 参数，则返回一个字符串，否则返回一个 Buffer 实例。 

#### ecdh.setPrivateKey(private_key[, encoding])

该方法用于设置 Diffie-Hellman 的私钥。如果指定了 `encoding` 的值为 `bianry`、`hex` 或 `base64` 中的一个，则 `private_key` 需要是字符串数据，否则 `private_key` 需要时 Buffer 实例。如果 `ECDH` 创建时指定的 carve 对 `private_key` 无效，将会抛出错误。在配置私钥的同时，ECDH 对象中相应的公钥也会被创建和配置。

#### echd.setPublicKey(public_key[, encoding])

<div class="s s0"></div>

## Class: Hash

`Hash` 类的实例常用于对数据进行哈希化。使用该类主要有两种方式：

- 包装成可读写的 stream 实例，写入纯数据，读出 Hash 数据
- 使用 `hash.update()` 和 `hash.digest()` 方法生成 Hash 数据

`crypto.createHash()` 方法常用于创建 `Hash` 类的实例。`Hash` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Hash` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const hash = crypto.createHash('sha256');

hash.on('readable', () => {
  var data = hash.read();
  if (data)
    console.log(data.toString('hex'));
    // Prints:
    //   6a2da20943931e9834fc12cfe5bb47bbd9ae43489a30726962b576f4e3993e50
});

hash.write('some data to hash');
hash.end();
```

下面代码演示了如何将 `Hash` 类的实例封装成管道流（piped stream)：

```js
const crypto = require('crypto');
const fs = require('fs');
const hash = crypto.createHash('sha256');

const input = fs.createReadStream('test.js');
input.pipe(hash).pipe(process.stdout);
```

下面代码演示了如何使用 `hash.update()` 和 `hash.digest()` 方法：

```js
const crypto = require('crypto');
const hash = crypto.createHash('sha256');

hash.update('some data to hash');
console.log(hash.digest('hex'));
// Prints:
//   6a2da20943931e9834fc12cfe5bb47bbd9ae43489a30726962b576f4e3993e50
```

#### hash.digest([encoding])

该方法用于计算原始数据的哈希摘要。`encoding` 参数的值必须为 `hex`、`binary` 或 `base64` 其中的一个。如果指定了有效的 `encoding` 参数，则该方法返回一个字符串，否则返回一个 Buffer 实例。

在 `hash.digest()` 方法执行之后，不能再重复调用 `Hash` 对象，多次调用该方法将会抛出错误。

#### hash.update(data[, input_encoding])

该方法根据 `data` 参数更新哈希后的数据，可选参数 `input_encoding` 指定的编码格式必须为 `utf8`、`ascii` 或 `binary` 之一。如果没有指定 `encoding` 参数，且 `data` 为字符串数据，则强制使用 `binary` 编码格式。如果 `data` 是一个 Buffer 实例，则会自动忽略 `input_encoding` 参数的值。

当数据被包装成 stream 实例之后，该方法可以调用多次。

## Class: Hmac

`Hmac` 类封装了一系列方法，用于创建加密的 HMAC 摘要。使用该类主要有两种方式：

- 包装成可读写的 stream 实例，写入纯数据，读出 HMAC 摘要
- 使用 `hmac.update()` 和 `hmac.digest()` 方法生成 HMAC 摘要

`crypto.createHmac()` 方法常用于创建 `Hmac` 类的实例。`Hmac` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Hmac` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const hmac = crypto.createHmac('sha256', 'a secret');

hmac.on('readable', () => {
  var data = hmac.read();
  if (data)
    console.log(data.toString('hex'));
    // Prints:
    //   7fd04df92f636fd450bc841c9418e5825c17f33ad9c87c518115a45971f7f77e
});

hmac.write('some data to hash');
hmac.end();
```

下面代码演示了如何将 `Hmac` 类的实例封装成管道流（piped stream)：

```js
const crypto = require('crypto');
const fs = require('fs');
const hmac = crypto.createHmac('sha256', 'a secret');

const input = fs.createReadStream('test.js');
input.pipe(hmac).pipe(process.stdout);
```

下面代码演示了如何使用 `hmac.update()` 和 `hmac.digest()` 方法：

```js
const crypto = require('crypto');
const hmac = crypto.createHmac('sha256', 'a secret');

hmac.update('some data to hash');
console.log(hmac.digest('hex'));
// Prints:
//   7fd04df92f636fd450bc841c9418e5825c17f33ad9c87c518115a45971f7f77e
```

#### hmac.digest([encoding])

该方法用于计算原始数据的 HMAC 摘要。`encoding` 参数的值必须为 `hex`、`binary` 或 `base64` 其中的一个。如果指定了有效的 `encoding` 参数，则该方法返回一个字符串，否则返回一个 Buffer 实例。

在 `hmac.digest()` 方法执行之后，不能再重复调用 `Hmac` 对象，多次调用该方法将会抛出错误。

#### hmac.update(data)

该方法根据 `data` 参数更新加密的 HMAC 数据。当数据被包装成 stream 实例之后，该方法可以调用多次。

## Class: Sign

`Sign` 类封装了一系列方法，用于生成签名。使用该类主要有两种方式：

- 包装成可写的 stream 实例并对数据进行签名，然后使用 `sign.sign()` 方法生成签名
- 使用 `sign.update()` 和 `sign.sign()` 方法生成签名

`crypto.createHmac()` 方法常用于创建 `Sign` 类的实例。`Sign` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Sign` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const sign = crypto.createSign('RSA-SHA256');

sign.write('some data to sign');
sign.end();

const private_key = getPrivateKeySomehow();
console.log(sign.sign(private_key, 'hex'));
// Prints the calculated signature
```

下面代码演示了如何使用 `sign.update()` 和 `sign.sign()` 方法：

```js
const crypto = require('crypto');
const sign = crypto.createSign('RSA-SHA256');

sign.update('some data to sign');

const private_key = getPrivateKeySomehow();
console.log(sign.sign(private_key, 'hex'));
// Prints the calculated signature
```

#### sign.sign(private_key[, output_format])

计算通过 `sign.update()` 或 `sign.write()` 方法传入的所有数据的签名信息。

`private_key` 参数可以是一个对象或是一个字符串。如果 `private_key` 是一个字符串，将会被当成没有 passphrase 的 key；如果 `private_key` 是一个对象，那么它将会被解释为一个哈希数据，该数据包含两个属性：

- `key`，字符串，使用 PEM 格式加密的私钥
- `passphrase`，字符串，私钥的密码

`output_format` 参数的值必须为 `hex`、`binary` 或 `base64` 其中的一个。如果指定了有效的 `output_format` 参数，则该方法返回一个字符串，否则返回一个 Buffer 实例。

在 `sign.sign()` 方法执行之后，不能再重复调用 `Sign` 对象，多次调用该方法将会抛出错误。

#### sign.update(data)

根据传入的 `data` 更新签名对象。当数据被包装成 stream 实例之后，该方法可以调用多次。

## Class: Verify

`Verify` 类封装了一系列的函数，用于验证签名。使用该类主要有两种方式：

- 包装成可写的 stream 实例并对签名进行验证
- 使用 `verify.update()` 和 `verify.verify()` 方法验证签名

`crypto.createVerify()` 方法常用于创建 `Verify` 类的实例。`Verify` 类的实例不能直接由 `new` 关键字创建。

下面代码演示了如果将一个 `Verify` 类的实例封装成 stream 实例：

```js
const crypto = require('crypto');
const verify = crypto.createVerify('RSA-SHA256');

verify.write('some data to sign');
verify.end();

const public_key = getPublicKeySomehow();
const signature = getSignatureToVerify();
console.log(sign.verify(public_key, signature));
// Prints true or false
```

下面代码演示了如何使用 `verify.update()` 和 `verify.verify()` 方法：

```js
const crypto = require('crypto');
const verify = crypto.createVerify('RSA-SHA256');

verify.update('some data to sign');

const public_key = getPublicKeySomehow();
const signature = getSignatureToVerify();
console.log(verify.verify(public_key, signature));
// Prints true or false
```

#### verifier.update(data)

该方法根据传入的 `data` 更新验证对象。当数据被包装成 stream 实例之后，该方法可以调用多次。

#### verifier.verify(object, signature[, signature_format])

该方法根据传入的 `object` 和 `signature` 参数验证数据。`object` 参数是一个字符串，该字符串包含一个使用 PEM 格式加密的对象，该对象可以是一个 RSA 公钥，可以是一个 DSA 公钥，或者是一个 X.509 证书。`signature` 参数是此前根据数据计算出来的签名。`signature_format` 参数的值可以是 `binary`、`hex` 或 `base64` 中的一个。如果指定了 `signature_format` 参数，那么 `signature` 必须是一个字符串，否则，`signature` 必须是一个 Buffer 实例。

该方法根据对数据签名和公钥的验证结果返回 true 或 false。

在 `verify.verify()` 方法执行之后，不能再重复调用 `verifier` 对象，多次调用该方法将会抛出错误。

## crypto 模块的成员方法和成员属性

#### crypto.DEFAULT_ENCODING

该属性用于声明函数的默认编码格式，可以是一个字符串或 Buffer 实例，默认值为 `buffer`，这表示函数默认接收的 Buffer 对象。

`crypto.DEFAULT_ENCODING` 存在的价值就是为了向后兼容老程序，在老程序中默认的编码格式是 `binary`。

新的应用程序应该使用默认值 `buffer`。该属性在未来的 Node.js 版本中可能会被抛弃。

#### crypto.createCipher(algorithm, password)

该方法根据传入的 `algorithm` 和 `password` 参数创建和返回一个 `Cipher` 对象。

`algorithm` 参数取决于 OpenSSL，比如 `aes192` 等。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的加密算法。

`password` 参数用于派生密钥和初始化向量（IV，initialization vector）。该参数的值必须是一个 `binary` 类型的加密字符串或者 Buffer 数组。

`crypto.createCipher()` 方法使用了 OpenSSL 的 `EVP_BytesToKey` 函数来派生密钥，该函数是 MD5 摘要算法，特点是一次迭代，无需加盐。因为没有加盐，所以容易被字典攻击暴力破解（使用相同的密码产生相同的密钥）。低迭代两和非加密的哈希算法则会让密码被快速的试错。

OpenSSL 建议使用 pbkdf2 替代 `EVP_BytesToKey`，也建议开发者使用 `crypto.pbkdf2()` 派生密钥和 IV，使用 `crypto.createCipheriv()` 创建 `Cipher` 对象。

#### crypto.createCipheriv(algorithm, key, iv)

该方法根据传入的 `algorithm`、`key` 和 `iv` 创建和返回一个 `Cipher` 对象。

`algorithm` 参数取决于 OpenSSL，比如 `aes192` 等。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的加密算法。

这里的 `key` 是用于 `algorithm` 的原始密钥，`iv` 是一个初始化向量。这两个参数都必须是 `binary` 格式的加密字符串或者 buffer 实例。

#### crypto.createCredentials(details)

<div class="s s0"></div>

#### crtpto.createDecipher(algorithm, password)

该方法根据传入的 `algorithm` 和 `password` 参数创建和返回一个 `Decipher` 对象。

`crypto.createDecipher()` 方法使用了 OpenSSL 的 `EVP_BytesToKey` 函数来派生密钥，该函数是 MD5 摘要算法，特点是一次迭代，无需加盐。因为没有加盐，所以容易被字典攻击暴力破解（使用相同的密码产生相同的密钥）。低迭代两和非加密的哈希算法则会让密码被快速的试错。

OpenSSL 建议使用 pbkdf2 替代 `EVP_BytesToKey`，也建议开发者使用 `crypto.pbkdf2()` 派生密钥和 IV，使用 `crypto.createCipheriv()` 创建 `Decipher` 对象。

#### crypto.createDecipheriv(algorithm, key, iv)

该方法根据传入的 `algorithm`、`key` 和 `iv` 创建和返回一个 `Decipher` 对象。

`algorithm` 参数取决于 OpenSSL，比如 `aes192` 等。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的加密算法。

这里的 `key` 是用于 `algorithm` 的原始密钥，`iv` 是一个初始化向量。这两个参数都必须是 `binary` 格式的加密字符串或者 buffer 实例。

#### crypto.createDiffieHellman(prime[, prime_encoding][, generator][, generator_encoding])

该方法根据传入的 `prime` 参数和可选的 `generator` 参数创建一个 `DiffieHellman` 密钥交换对象。

`generator` 参数可以是数值、字符串和 Buffer 实例。如果未指定 `generator` 函数，则为其添加默认值 `2`。

`prime_encoding` 和 `generator_encoding` 参数可以是 `binary`、`hex` 或 `base64`。

如果指定了 `prime_encoding`，那么 `prime` 需要是一个字符串，否则，`prime` 需要是一个 Buffer 实例。

如果指定了 `generator_encoding`，那么 `generator` 需要是一个字符串，否则，`generator` 需要是一个数值或 Buffer 实例。

#### crypto.createDiffieHellman(prime_length[, generator])

该方法用于创建 `DiffieHellman` 密钥交换对象，并根据 `prime_length` 的大小使用可选的数值类型的 `generator` 生成指定大小的质数。如果未指定 `generator`，则使用默认值 `2`。

#### crypto.createECDH(curve_name)

该方法根据字符串参数 `curve_name` 创建一个 Elliptic Curve Diffie-Hellman 密钥交换对象。使用 `crypto.getCurves()` 可以得到一组可用的 curve 名字。在最新的 OpenSSL 版本中，使用 `openssl ecparam -list_curves` 命令可以每一个可用的椭圆曲线（elliptic curve）的名字和描述。

#### crypto.createHash(algorithm)

该方法用于创建和返回一个 Hash 对象，该对象可以使用指定 `algorithm` 生成哈希摘要。

`algorithm` 参数的值受限于当前平台 OpenSLL 所支持的算法，比如 `sha256`、`sha512` 等。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的摘要算法。

下面代码演示了如何生成一个文件的 sha236 摘要：

```js
const filename = process.argv[2];
const crypto = require('crypto');
const fs = require('fs');

const hash = crypto.createHash('sha256');

const input = fs.createReadStream(filename);
input.on('readable', () => {
  var data = input.read();
  if (data)
    hash.update(data);
  else {
    console.log(`${hash.digest('hex')} ${filename}`);
  }
});
```

#### crypto.createHmac(algorithm, key)

该方法根据传入的 `algorithm` 和 `key` 创建和返回一个 Hmac 对象。

`algorithm` 参数的值受限于当前平台 OpenSLL 所支持的算法，比如 `sha256`、`sha512` 等。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的摘要算法。

这里的参数 `key` 是 HMAC 用于生成加密 HMAC 哈希的密钥。

下面代码演示了如何生成一个文件的 sha256 HMAC:

```js
const filename = process.argv[2];
const crypto = require('crypto');
const fs = require('fs');

const hmac = crypto.createHmac('sha256', 'a secret');

const input = fs.createReadStream(filename);
input.on('readable', () => {
  var data = input.read();
  if (data)
    hmac.update(data);
  else {
    console.log(`${hmac.digest('hex')} ${filename}`);
  }
});
``` 

#### crypto.createSign(algorithm)

该方法根据传入的 `algorithm` 创建和返回一个 `Sign` 对象。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的签名算法，比如 `RSA-SHA256`。

#### crypto.createVerify(algorithm)

该方法根据传入的 `algorithm` 创建和返回一个 `Verify` 对象。在最新的 OpneSSL 版本中，使用 `openssl list-cipher-algorithms` 命令可以显示所有可用的签名算法，比如 `RSA-SHA256`。

#### crypto.getCiphers()

该方法返回一个数组，该数组描述了当前所支持的所有加密算法：

```js
const ciphers = crypto.getCiphers();
console.log(ciphers); 
// ['aes-128-cbc', 'aes-128-ccm', ...]
```

#### crypto.getCurves()

该方法返回一个数组，该数组描述了当前所支持的所有椭圆曲线（elliptic curves):

```js
const curves = crypto.getCurves();
console.log(curves); // ['secp256k1', 'secp384r1', ...]
```

#### crypto.getDiffieHellman(group_name)

该方法用于创建一个约定义的 `DiffieHellman` 密钥交换对象。支持的组包括：`modp1`、`modp2`、`modp5`（以上定义在 RFC 2412 中）和 `modp14`、`modp15`、`modp16`、`modp17`、`modp18`（以上定义在 RFC 3526 中）。返回的对象类似 `crypto.createDiffieHellman()` 方法所返回的对象，但不允许交换密钥。使用该方法的优势在于，无需提前生成或交换组余数，节省了计算和通信时间。

```js
const crypto = require('crypto');
const alice = crypto.getDiffieHellman('modp14');
const bob = crypto.getDiffieHellman('modp14');

alice.generateKeys();
bob.generateKeys();

const alice_secret = alice.computeSecret(bob.getPublicKey(), null, 'hex');
const bob_secret = bob.computeSecret(alice.getPublicKey(), null, 'hex');

/* alice_secret and bob_secret should be the same */
console.log(alice_secret == bob_secret);
```

#### crypto.getHashes()

该方法返回一个数组，数组内容是当前所支持的 Hash 算法的名字：

```js
const hashes = crypto.getHashes();
console.log(hashes); // ['sha', 'sha1', 'sha1WithRSAEncryption', ...]
```

#### crypto.pbkdf2(password, salt, iterations, kenlen[, digest], callback)

该方法是 PBKDF2(Password-Based Key Derivation Function 2) 的一个异步版本。参数 `digest` 指定了 HMAC 摘要算法，该算法根据 `password`、`salt` 和 `iterations` 参数创建一个 `keylen` 长度的密钥。如果未指定 `digest` 算法，则使用默认值 `sha1`。

`callback` 回调函数接收两个参数：`err` 和 `derivedKey`。如果存在错误，则 `err` 会被赋值，否则 err 的值为 null。成功生成的 `derivedKey` 将会被被传递为一个 Buffer 实例。

`iterations` 参数必须是一个尽可能大的数值，数值越大，密钥的安全性越高，则计算时间也会越长。

`salt` 参数应该尽可能独一无二。建议 `salt` 为随机值且长度大于 16 个字节。

```js
const crypto = require('crypto');
crypto.pbkdf2('secret', 'salt', 100000, 512, 'sha512', (err, key) => {
  if (err) throw err;
  console.log(key.toString('hex'));  // 'c5e478d...1469e50'
});
```

调用 `crypto.getHashes()` 方法可以获取当前支持的摘要函数。

#### crypto.pbkdf2Sync(password, salt, iterations, keylen[, digest])

该方法是 PBKDF2(Password-Based Key Derivation Function 2) 的一个同步版本。参数 `digest` 指定了 HMAC 摘要算法，该算法根据 `password`、`salt` 和 `iterations` 参数创建一个 `keylen` 长度的密钥。如果未指定 `digest` 算法，则使用默认值 `sha1`。

如果存在错误，则 `err` 会被赋值，否则 err 的值为 null。成功生成的 `derivedKey` 将会被被传递为一个 Buffer 实例。

`iterations` 参数必须是一个尽可能大的数值，数值越大，密钥的安全性越高，则计算时间也会越长。

`salt` 参数应该尽可能独一无二。建议 `salt` 为随机值且长度大于 16 个字节。

```js
const crypto = require('crypto');
const key = crypto.pbkdf2Sync('secret', 'salt', 100000, 512, 'sha512');
console.log(key.toString('hex'));  // 'c5e478d...1469e50'
```

调用 `crypto.getHashes()` 方法可以获取当前支持的摘要函数。

#### crypto.privateDecrypt(private_key, buffer)

该方法跟根据传入的 `private_key` 解密 `buffer`。

`private_key` 参数可以是一个对象或是一个字符串。如果 `private_key` 是一个字符串，将会被当成没有 passphrase 的 key，且会使用 `RSA_PKCS1_OAEP_PADDING`；如果 `private_key` 是一个对象，那么它将会被解释为一个哈希对象，该对象包含以下属性：

- `key`，字符串，使用 PEM 格式加密的私钥
- `passphrase`，字符串，私钥的密码
- `padding`: 一个可选的填充值，有效值包括：
    - constants.RSA_NO_PADDING
    - constants.RSA_PKCS1_PADDING
    - constants.RSA_PKCS1_OAEP_PADDING

所有的填充行为都被定义在了 `constants` 模块中。

#### crypto.privateEncrypt(private_key, buffer)

该方法跟根据传入的 `private_key` 加密 `buffer`。

`private_key` 参数可以是一个对象或是一个字符串。如果 `private_key` 是一个字符串，将会被当成没有 passphrase 的 key，且会使用 `RSA_PKCS1_PADDING`；如果 `private_key` 是一个对象，那么它将会被解释为一个哈希对象，该对象包含以下属性：

- `key`，字符串，使用 PEM 格式加密的私钥
- `passphrase`，字符串，私钥的密码
- `padding`: 一个可选的填充值，有效值包括：
    - constants.RSA_NO_PADDING
    - constants.RSA_PKCS1_PADDING
    - constants.RSA_PKCS1_OAEP_PADDING

所有的填充行为都被定义在了 `constants` 模块中。

#### crypto.publicDecrypt(public_key, buffer)

该方法跟根据传入的 `public_key` 解密 `buffer`。

`public_key` 参数可以是一个对象或是一个字符串。如果 `public_key` 是一个字符串，将会被当成没有 passphrase 的 key，且会使用 `RSA_PKCS1_PADDING`；如果 `public_key` 是一个对象，那么它将会被解释为一个哈希对象，该对象包含以下属性：

- `key`，字符串，使用 PEM 格式加密的私钥
- `passphrase`，字符串，私钥的密码
- `padding`: 一个可选的填充值，有效值包括：
    - constants.RSA_NO_PADDING
    - constants.RSA_PKCS1_PADDING
    - constants.RSA_PKCS1_OAEP_PADDING

因为 RSA 公钥可以由私钥产生，所以可以通过传递私钥代替传递公钥。

所有的填充行为都被定义在了 `constants` 模块中。

#### crypto.publicEncrypt(public_key, buffer)

该方法跟根据传入的 `public_key` 加密 `buffer`。

`public_key` 参数可以是一个对象或是一个字符串。如果 `public_key` 是一个字符串，将会被当成没有 passphrase 的 key，且会使用 `RSA_PKCS1_OAEP_PADDING`；如果 `public_key` 是一个对象，那么它将会被解释为一个哈希对象，该对象包含以下属性：

- `key`，字符串，使用 PEM 格式加密的私钥
- `passphrase`，字符串，私钥的密码
- `padding`: 一个可选的填充值，有效值包括：
    - constants.RSA_NO_PADDING
    - constants.RSA_PKCS1_PADDING
    - constants.RSA_PKCS1_OAEP_PADDING

因为 RSA 公钥可以由私钥产生，所以可以通过传递私钥代替传递公钥。

所有的填充行为都被定义在了 `constants` 模块中。

#### crypto.randomBytes(size[, callback])

该方法用于生成一个高安全等级的伪随机数据，这里的 `size` 参数指定了生成的数据大小，单位是字节。

如果指定了 `callback`，则使用异步方式生成数据，且 `callback` 函数接收两个参数: `err` 和 `buf`。如果执行过程中抛出错误，`err` 会被赋值为一个 Error 实例，否则为 null。`buf` 参数是一个 Buffer 实例，该实例引用了生成后的数据：

```js
// Asynchronous
const crypto = require('crypto');
crypto.randomBytes(256, (err, buf) => {
  if (err) throw err;
  console.log(`${buf.length} bytes of random data: ${buf.toString('hex')}`);
});
```

如果未指定 `callback` 函数，则使用同步的方式生成随机数据并返回一个 Buffer 实例。如果生成过程中有任何错误，则会抛出错误：

```js
// Synchronous
const buf = crypto.randomBytes(256);
console.log(`${buf.length}` bytes of random data: ${buf.toString('hex')});
```

如果没有足够的熵，则 `crypto.randomBytes()` 方法会停止。通常来说，这只会占用几毫秒的时间。唯一可能会阻塞较长时间的可能是在系统启动后生成随机数据，因为此时整个系统的熵比较低。

#### crypto.setEngine(engine[, flags])

该方法对某些或全部的 OpenSSL 函数加载和设置 `engine`。

`engine` 可以是 engine 共享库的 id 或路径。

可选参数 `flags` 的默认值为 `ENGINE_METHOD_ALL`，该值可以是以下列出的一个或多个值：

- `ENGINE_METHOD_RSA`
- `ENGINE_METHOD_DSA`
- `ENGINE_METHOD_DH`
- `ENGINE_METHOD_RAND`
- `ENGINE_METHOD_ECDH`
- `ENGINE_METHOD_ECDSA`
- `ENGINE_METHOD_CIPHERS`
- `ENGINE_METHOD_DIGESTS`
- `ENGINE_METHOD_STORE`
- `ENGINE_METHOD_PKEY_METH`
- `ENGINE_METHOD_PKEY_ASN1_METH`
- `ENGINE_METHOD_ALL`
- `ENGINE_METHOD_NONE`

## 后记

#### Legacy Streams API (pre Node.js v0.10)

The Crypto module was added to Node.js before there was the concept of a unified Stream API, and before there were Buffer objects for handling binary data. As such, the many of the crypto defined classes have methods not typically found on other Node.js classes that implement the streams API (e.g. update(), final(), or digest()). Also, many methods accepted and returned 'binary' encoded strings by default rather than Buffers. This default was changed after Node.js v0.8 to use Buffer objects by default instead.

#### Recent ECDH Changes

Usage of ECDH with non-dynamically generated key pairs has been simplified. Now, ecdh.setPrivateKey() can be called with a preselected private key and the associated public point (key) will be computed and stored in the object. This allows code to only store and provide the private part of the EC key pair. ecdh.setPrivateKey() now also validates that the private key is valid for the selected curve.

The ecdh.setPublicKey() method is now deprecated as its inclusion in the API is not useful. Either a previously stored private key should be set, which automatically generates the associated public key, or ecdh.generateKeys() should be called. The main drawback of using ecdh.setPublicKey() is that it can be used to put the ECDH key pair into an inconsistent state.

#### Support for weak or compromised algorithms

The crypto module still supports some algorithms which are already compromised and are not currently recommended for use. The API also allows the use of ciphers and hashes with a small key size that are considered to be too weak for safe use.

Users should take full responsibility for selecting the crypto algorithm and key size according to their security requirements.

Based on the recommendations of NIST SP 800-131A:

- MD5 and SHA-1 are no longer acceptable where collision resistance is required such as digital signatures.
The key used with RSA, DSA and DH algorithms is recommended to have at least 2048 - bits and that of the curve of ECDSA and ECDH at least 224 bits, to be safe to use for several years.
- The DH groups of modp1, modp2 and modp5 have a key size smaller than 2048 bits and are not recommended.

See the reference for other recommendations and details.

<style>
.s {
    margin: 1.5rem 0;
    padding: 10px 20px;
    color: white;
    border-radius: 5px;
}
.s:before {
    display: block;
    font-size: 2rem;
    font-weight: 900;
}
.s0 {
    background-color: #C04848;
}
.s0:before {
    content: "接口稳定性: 0 - 已过时";
}
.s1 {
    background-color: #F07241;
}
.s1:before {
    content: "接口稳定性: 1 - 实验中";
}
.s2 {
    background-color: #457D97;
}
.s2:before {
    content: "接口稳定性: 2 - 稳定";
}
.s3 {
    background-color: #14C3A2;
}
.s3:before {
    content: "接口稳定性: 3 - 已锁定";
}
</style>