# Bot API

This readme describes how to create signature for Membrana REST API.

## Making request

Before creating request you need to receive KeyID and Secret to perform requests.

To create request you need to make next steps:

1. Combine message.
2. Create HMAC-SHA256.
3. Create HTTP request.

### Combine message

Message consists of HTTP method, URL (without schema), nonce and raw body separated with new line `\n` and prefixed with 64-bit length in Big Endian bytes order.

```
MSG = LENGTH_PREFIX DATA
DATA = HTTP_METHOD NL URL NL NONCE NL BODY
```

* `LENGTH_PREFIX` is fixed value (64bit number, 8 bytes long). This value is length of `DATA`.
* `HTTP_METHOD` is an uppercased method name. Example: `GET`.
* `URL` is a schemaless address consisted of hostname and pathname. Example: `membrana.io/api/v1/extern/orders`.
* `NONCE` is stringified number less then 64-bit integer maximum value. It could be timestamp. Example: `1536320723113`.
* `BODY` is stringified JSON or empty string for read requests.

### Creating HMAC signature

Combined message should be signed with the secret using HMAC-SHA256 algorithm. This is well known process.

### Creating HTTP request

To create HTTP request you should to:
1. Add Authorization HTTP header:
	`Authorization: membrana-token KEY_ID:SIGNATURE:NONCE`
	Where:
	* `KEY_ID`  is KeyID received from Membrana.
	* `SIGNATURE` is hex encoded signature.
	* `NONCE` is request unique counter which should be greater that previously sent value. Could be current time (`Date.now`).
3. If request has a body, set Content-Type header:
	`Content-Type: application/json`.

## Example

This is complete example based on fetch library.

```javascript
const crypto = require('crypto');
const fetch = require('node-fetch');

function request({method = 'GET', url, body, keyId, secret}) {
	// Prepare params
	const bodyValue = body && JSON.stringify(body);
	const nonce = Date.now();
	
	// Create signature
	const signature = signRequestParams({
		method,
		url,
		body: bodyValue,
		nonce,
		secret,
	})
	.toString('hex');

	// Prepare HTTP config headers.
	const headers = {
		'Authorization': `membrana-token ${keyId}:${signature}:${nonce}`,
	};
	
	if (body) {
		headers['content-type'] = 'application/json';
	}

	// Send request
	return fetch(`https://${url}`, {
		method,
		headers,
		body: bodyValue,
	});
}

function signRequestParams({method, url, nonce, body = '', secret}) {
	const params = `${method}\n${url}\n${nonce}\n${body}`;
	return hmac(toLengthPrefixedBuffer(params), secret);
}

function toLengthPrefixedBuffer(msg) {
    const msgBuf = Buffer.from(msg, 'utf8');
    const sizeBuf = Buffer.allocUnsafe(8);
    
    // Write 64 bit length. Took from SO: https://stackoverflow.com/a/51098122/884491
    const size = msgBuf.length;
    const big = ~~(size / 0x0100000000);
    const low = (size % 0x0100000000);
    sizeBuf.writeUInt32BE(big, 0);
    sizeBuf.writeUInt32BE(low, 4);
    
    return Buffer.concat([sizeBuf, msgBuf]);
}

function hmac(value, secret) {
	return crypto.createHmac('sha256', secret)
	.update(value)
	.digest();
}
```
