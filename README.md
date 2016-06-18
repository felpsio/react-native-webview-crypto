# react-native-crypto

This brings `window.Crypto` to your React Native application. It does this
by communicating with a hidden WebView, which performs the actual
computation.

## Why does this exist?

The [Web Cryptography API](http://caniuse.com/#feat=cryptography)
is [implemented in all major browsers](http://caniuse.com/#feat=cryptography)
and provides performant and secure way of doing encyrption in JavaScript. However, it is [not supported in the React Native JavaScript runtime](https://github.com/facebook/react-native/issues/1189)
for some reason. (Go ahead, check if `window.crypto` is defined without the chrome debugger open).

On modern Android and iOS systems, there is a nice implementation of the API already, sitting in your browsers.
So this provides an object that fulfills the `Crypto` interface, by using that implementation,
communicating through a hidden WebView.

### Caveats
Since this uses an asynchronous bridge to execute the crypto logic it
can't quite execute `crypto.getRandomValues` correctly, because that method
returns a value synchronously. It is simply *impossible* (as far as I know, 
please let me know if there any ways to get around this) to wait for the
bridge to respond asynchronously before returning a value.

Instead, we return you the same `TypedArray` you passed in and asynchronously
update it behind the scenes, when we get a response. We also set a `updated` proprety
on it to be a promise that resolves to the same array when we get a response back
and update it. We check for this property on all `cyrpto.subtle` methods that takes in
`TypedArray`s and will automatically wait for them to update before asking the
webview to execute them.

*TLDR*: If you need to use the result from `crypto.getRandomValues` for something
other than passing into a `crypto.subtle` method, you have to wait for the
Promise on the `updated` property of the returned `TypedArray` to resolve
before the values in that `TypedArray` will be updated.



## Install

1. Get started with React Native
2. Install [React Native WebView Javascript Bridge](https://github.com/alinz/react-native-webview-bridge)
   and verify that it is working for your platform.
3. `npm install --save react-native-crypto`


## Quickstart

Render the `CryptoWorker` component so that the worker starts up.
Then get the `crypto` attribute from it and use that as `window.Crypto`

```javascript
import React, { Component } from 'react';
import { View } from 'react-native';

import App from './app';

import CryptoWorker from 'react-native-crypto';

class TopLevelComponent extends Component {
  render() {
    return (
      <View>
        <CryptoWorker ref={(cw) => window.Crypto = cw.crypto}/>
        <App>
      </View>
    );
  }
}

AppRegistry.registerComponent('WhateverName', () => TopLevelComponent);
```

Now, in any of your code, you can access `window.Crypto`, just like
if it was native.
Using [this example for symmetric encryption](https://blog.engelke.com/2014/06/22/symmetric-cryptography-in-the-browser-part-1/)
your application should log `This is very sensitive stuff.` if it was
succesful.


```javascript
var keyPromise = window.crypto.subtle.generateKey(
    {name: "AES-CBC", length: 128}, // Algorithm the key will be used with
    true,                           // Can extract key value to binary string
    ["encrypt", "decrypt"]          // Use for these operations
);

var aesKey;   // Global variable for saving
keyPromise.then(function(key) {aesKey = key;});
keyPromise.catch(function(err) {alert("Something went wrong: " + err.message);});

var iv = new Uint8Array(16);
window.crypto.getRandomValues(iv);

var iv = window.crypto.getRandomValues(new Uint8Array(16));

var plainTextString = "This is very sensitive stuff.";

var plainTextBytes = new Uint8Array(plainTextString.length);
for (var i=0; i<plainTextString.length; i++) {
    plainTextBytes[i] = plainTextString.charCodeAt(i);
}

var cipherTextBytes;
var encryptPromise = window.crypto.subtle.encrypt(
    {name: "AES-CBC", iv: iv}, // Random data for security
    aesKey,                    // The key to use
    plainTextBytes             // Data to encrypt
);
encryptPromise.then(function(result) {cipherTextBytes = new Uint8Array(result);});
encryptPromise.catch(function(err) {alert("Problem encrypting: " + err.message);});

var decryptPromise = window.crypto.subtle.decrypt(
    {name: "AES-CBC", iv: iv}, // Same IV as for encryption
    aesKey,                    // The key to use
    cipherTextBytes            // Data to decrypt
);
var decryptedBytes;
decryptPromise.then(function(result) {decryptedBytes = new Uint8Array(result);});
decryptPromise.catch(function(err) {alert("Problem decrypting: " + err.message); });

var decryptedString = "";
for (var i=0; i<decryptedBytes.byteLength; i++) {
    decryptedString += String.fromCharCode(decryptedBytes[i]);
}

console.log(decryptedString)
```
