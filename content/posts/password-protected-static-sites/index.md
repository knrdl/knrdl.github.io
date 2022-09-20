---
title: "Password-protected resources on static-site webhosters"
date: "2022-09-20"
tags: [html5, webdev, webcrypto, websec, hashing]
lang: "en"
---

# Scenario

Some web hosters only serve static files and allow no config changes to the webserver. But maybe you want to provide files which are not intended for public view, for example sharing a file with a friend. Therefore, the best you can do is protecting files by giving them names which are hard to guess. Obviously these files should also not be linked somewhere publicly at all.

This concept can be expanded with a clientside-only authentication mechanism, as described next.

# Login process

## 1. The user opens the webpage

A login dialog with password input is shown to the user. The user inputs a password.

<div style="text-align: center; font-size: 20pt">
  <a href="demo" target="_blank">&gt;&gt; click here for a demo &lt;&lt;</a>
</div>

## 2. Clientside password hashing

Now the password must be locally digested on the webpage. A hashing algorithm suitable for passwords must be applied. PBKDF2 as provided by the WebCryptoAPI is acceptable with an [iteration count of 310,000](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) in HMAC-SHA-256 mode. The hash should be salted with [at least 16 bytes of randomness](https://developer.mozilla.org/en-US/docs/Web/API/Pbkdf2Params). The salt can be stored as plaintext alongside the login page. Generating a salt is as easy as `dd if=/dev/urandom bs=1 count=16 | base64`.

```javascript
/** clientside hashing a password
 * @param {string} password - as provided by user
 * @param {string} salt - as base64 encoded
 * @return {Promise<string>} - the hash value
 */
async function hashPassword(password, salt) {
    const passwordKey = await window.crypto.subtle.importKey(
        "raw",
        new TextEncoder().encode(password),
        {name: "PBKDF2"},
        false, // key should not be extractable
        ["deriveBits"]
    )
    const hashBuffer = await window.crypto.subtle.deriveBits(
        {"name": "PBKDF2", salt: base64ToArrayBuffer(salt), "iterations": 310_000, "hash": "SHA-256"},
        passwordKey,
        256
    )
    const hashArray = Array.from(new Uint8Array(hashBuffer))
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('').toUpperCase()
}

/** converts a base64 encoded string into an arraybuffer
 * @param {string} base64text
 * @return {ArrayBuffer}
 */
function base64ToArrayBuffer(base64text) {
    const bytes = new Uint8Array(base64text.length)
    for (let i = 0; i < base64text.length; i++)
        bytes[i] = base64text.charCodeAt(i)
    return bytes.buffer
}
```

## 3. Redirect to the secret path

The created hash-value is taken as a path parameter for the url. As UX improvement, a preflight fetch request checks if the entered password is correct. If that's the case, a redirect is performed. The user is now authenticated.

```javascript
const password = document.querySelector('input[type=password]').value
const salt = 'ChangeTheSaltValueASAP=='
const hashValue = await hashPassword(password, salt)
const url = window.location.origin + window.location.pathname + '/' + hashValue
fetch(url).then(async res => {
    if (res.ok)
        window.location.replace(url)
    else throw Error(await res.text())
}).catch(err => {
    alert('Password wrong')  // todo: evaluate error msg
})
```

> It's possible to create user specific protected paths by concatenating the static salt with a provided additional userID. That way separate accounts with userID and password as credentials would be possible.
> {.info }

# Pros

* As the calculation-heavy hashing is performed exclusively clientside, there is no extra load serverside. For better scalability this approach can even be combined with a CDN.
* A Static Site Generator (SSG), like Hugo, can be used to automatically generate protected resource paths from predefined passwords. The SSG only needs to perform hashing on creation of a new protected resource.
* The hashing works as key stretching operation to generate urls which are long enough to be unsearchable.
  A brute-force attacker who can perform a billion requests per second would need <math>
  <msup>
  <mi>2</mi>
  <mrow>
  <mn>256</mn>
  </mrow>
  </msup>
  <mo>/</mo>
  <msup>
  <mi>10</mi>
  <mrow>
  <mn>9</mn>
  </mrow>
  </msup>
  <mo>&asymp;</mo>
  <msup>
  <mi>10</mi>
  <mrow>
  <mn>60</mn>
  </mrow>
  </msup> <mi> years</mi>
  </math>. That way a bruteforce attack for the passwords still is the most efficient one.
* The user can bookmark protected resource paths, so there is no further login required (ux improvement).

# Cons

* The approach doesn't scale well for many user. As a workaround there might be group-contents defined and each "user protected path" just contains a redirect to the "group protected path". Otherwise, there will be a lot of duplication.
* A dynamic creation of user accounts is not possible. But as it's all about static hosting, this is out of scope.
* Users can give unintentional access to third parties by just copypasting the url. Maybe it's possible to cloak the shown url with a combination of the [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)'s `replaceState` and the [base tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base)? Or just provide the protected content as <abbr title="Single Page Application">SPA</abbr> with a rewritten display url.
* The secret key must be transported as part of the url to the server. That way sensitive information will be written into the access logfiles of webserver and proxies. This violates security goals and is definitely not best practice!
* Accidentally enabling public [directory listings](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html) will also break any security goals apart.
* There is a tradeoff between hashing duration and security. The WebCryptoAPI allows hashing to be performant but only implements PBKDF2. A js/wasm library might provide a better algorithm but perhaps unsatisfying performance. It's a pity that the WebCryptoAPI doesn't support a more elaborated password hashing algorithm, e.g. bcrypt, scrypt, argon2id.
* There is no ratelimiting applicable: A brute force attacker is only limited by the network transmission time and calculation cost for a password hash, which can both be heavily parallelized. As a result the password must be strong enough, e.g. not trivially constructable from password-list permutations.

# Conclusion

Is it possible? Yes, absolutely! And should we implement this? Please don't, if it's avoidable in any way. The explained approach is only useful in a very specific scenario (see above). In almost all cases there would be a more standard-applying way to realize that, for example using good old HTTP Basic Auth. Or initiate a session after login instead, so there is no further exchange of highly privileged key material (user credentials) required.

