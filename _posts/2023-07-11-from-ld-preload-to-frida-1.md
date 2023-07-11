---
layout: post
title: From LD_PRELOAD To Frida [0x1|FA]
tags: LD_PRELOAD Frida Hooking
---

<style>
p,ul,h2,h4    {direction: rtl;}
</style>

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/poster.png" width=750 >
</div>

## Frida

خب به بخش مربوط به فریدا خوش آمدید؛
فریدا خلق شده تا کار های مربوط به Hooking ما را آسان‌تر کند.

قبل از اینکه سراغ حل مساله برویم بیایید بیشتر با فریدا آشنا شویم.

فریدا با تزریق یک Javascript engine به Process مورد نظر کار میکند و عموما با کد Javascript کنترل میشود.<br/>
فریدا دارای دو Javascript engine است [JavaScript V8 Engine](https://v8.dev) و [QuickJS](https://bellard.org/quickjs/) (که بجای Ducktape قرار داده شده)، که با استفاده آن ها کد های Javascript را اجرا میکند.

#### Modes of Operation

فریدا چند حالت عملیاتی دارد:
- Injected: وقتی که یک برنامه را میخواهید اجرا کنید یا میخواهید به برنامه درحال اجرا Attach بشوید، این حالت میتواند به شما کمک بکند
- Embedded: بعضی مواقع هم برای ما ممکن نیست که فریدا را در حالت Injected استفاده کنیم، در این صورت میتوانیم frida-gadget که یک Shared library هست را داخل برنامه‌مان Embed بکنیم
- Preloaded: در این روش هم مثل مثال بخش قبل که از `LD_PRELOAD` برای لود کردن Shared objectای استفاده کرده بودیم، استفاده خواهیم کرد، فقط اینبار قرار است frida-gadget را بعنوان مقدار `LD_PRELOAD` بدهیم

ما قرار هست عموما از حالت Injected استفاده کنیم.

ما میتوانیم برنامه ها را توسط فریدا اجرا کنیم یا به برنامه های اجرا شده وصل شویم، تفاوت این دو گزینه به این صورت است:
- اجرای برنامه توسط فریدا (Spawning): برنامه را شروع و سپس آن را متوقف میکند، که این توقفEarly instrumentation را برای ما ممکن میسازد.
- وصل شدن به یک برنامه در حال اجرا (Attaching): فریدا را با استفاده از `PTRACE` به Process در حال اجرا Inject میکند و در نتیجه یک Thread جدید ایجاد میشود که برای Instrumentation هست.


برای اجرای یک برنامه با استفاده از فریدا (Spawning):<br/>
گزینه `pause` برای متوقف کردن Thread اصلی پس از شروع برنامه است.

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/frida-1.png" width=750 >
</div>

برای وصل شدن به یک برنامه در حال اجرا در فریدا (Attaching):

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/frida-2.png" width=750 >
	<figcaption>برنامه در پنجره زیر اجرا میشود و در پنجره بالا ما با استفاده از اسم Process آن، متصل میشویم.</figcaption>
</div>

شما میتوانید با داشتن PID برنامه مورد نظر هم به آن متصل شوید.

که در صورت اجرای هر دو دستور شما در محیط REPL قرار خواهید گرفت.
یکی از مزایای REPL این است که به ما قابلیت Auto complete را میدهد.

گزینه دیگری هم وجود دارد که برای اجرای دستورات درون یک فایل استفاده میشود.
ابتدا شما باید دستورات خود درون یک فایل بنویسید و آن را به عنوان یک Argument به فریدا دهید.

```
frida sleepy --load index.js
```

توصیه میشود که توسعه های طولانی مدت را در اسکریپت ها انجام دهید.


#### Instrumentation

خب رسیدیم به اولین API از فریدا

```
Interceptor.attach(target, callbacks[, data])
```

این API به ما قابلیت بررسی و تغییر مقادیر ورودی و خروجی تابع را میدهد.

برای اینکه بتوانیم Interceptor را به یک تابع Hook کنیم باید به Target و Callback نیازمندیم. 
Target میتواند هم یک Offset باشد و یا میتواند یک Symbol باشد.

زمانی که Symbol ها موجود نیستند باید Offset را بدست بیارید، چون ما Symbol ها را داریم محاسبه Offset ها را بررسی نمیکنیم.<br/>
در هر دو صورت ما به آدرس تابعی که میخواهیم Hook شویم احتیاج داریم.

چون در این حالت ما به Symbol ها دسترسی داریم و می‌دانیم که تابع مورد نظرمان `sleep` است، اکنون میتوانیم با استفاده از API های زیر آدرس `sleep` را داخل Process لود شده پیدا کنیم.
- با استفاده از Export ها `Module.getExportByName("libc.so.6", "sleep")`
- و یا اگر به Debug symbol ها دسترسی داشته باشیم `DebugSymbol.getFunctionByName("sleep")`

بخش Targetمان رو بدست آوردیم حالا نوبت Callback است.

Callback ها دو تابع `onEnter` و `onLeave` هستند:
- تابع `onEnter` زمانی اجرا میشود که وقتی تابع Target صدا زده میشود.
	- تابع `onEnter` به ما دسترسی خواندن و تغییر Argument هایی که تابع Target داده شده را میدهد (در صورت وجود).
- تابع `onLeave` هم زمانی اجرا میشود که تابع Target میخواهد پایان یابد.
	-  تابع `onLeave` هم دسترسی خواندن و تغییر مقداری که Target قرار است برگرداند را میدهد (در صورت وجود).

```js
var sleepPtr= Module.getExportByName(null, "sleep");

Interceptor.attach(sleepPtr, {
	onEnter: function(args) {},
	onLeave: function(retval) {}
});
```

حالا بیایید با استفاده از `Interceptor.attach` برنامه قسمت قبل را تغییر دهیم.

```js
// index.js

var sleepPtr = Module.getExportByName(null, "sleep");

Interceptor.attach(sleepPtr, {
    onEnter: function(args) {
        console.log("[*] Sleep from Frida!");
    },
});

```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/frida-3.png" width=750 >
</div>

جالب شد :)

ولی چطور این اتفاق افتاد؟

بیاید با یک Debugger کد مربوط به `sleep` را در Process بررسی کنیم.<br/>
چون همزمان وصل کردن فریدا و GDB به یک Process ممکن نیست (شایدم من بلد نیستم) من از روش Preloaded برای اتصال فریدا به Process استفاده میکنم تا GDB تنها دیباگر متصل به Process باشد

عکس زیر مربوط به Disassembly کد `sleep` قبل از اتصال فریدا است:

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/disass-1.png" width=750 >
</div>

این عکس هم مربوط به Disassembly کد `sleep` بعد از اتصال فریدا است:

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/disass-2.png" width=750 >
</div>

=! توجه داشته باشید که این رفتار `Interceptor.attach` میباشد، API های دیگر ممکن است به این روش کار نکنند.

خب ما که به Argument و Retval های تابع دسترسی داریم بیایید کمی با آن ها بازی کنیم :)

چون ما به Argument های تابع دسترسی داریم پس میتوانیم آنها را بخوانیم یا تغییرشان دهیم

فقط چند نکته قبل از اینکه با Argument ها بازی کنیم :
- Interceptor از تعداد Argument ها خبری ندارد، این شما هستید که باید تعداد آنها را بدانید
- `args[0]` اولین Argument است.
- نوع داده Argument ها `NativePointer` است.

```
args = [
	NativePointer("0x01"),       # args[0]
	NativePointer("0x0fe4795"),  # args[1]
	...
];
```

برای تغییر Argument تابع `sleep` هم میتوانید از روش زیر استفاده کنید

- تغییر مقادیر عددی (Integer):<br/>
تغییر مقادیر عددی ساده بوده و مانند تصویر زیر انجام میشود.

```js
args[0] = ptr("0x01");
// or
args[0] = new NativePointer("0x01");
```

- تغییر مقادیر رشته‌ای (String):<br/>
تغییر مقادیر String به سادگی مقادیر عددی نیست.
برای اینکار باید یک آرایه از کاراکتر ها با استفاده از ‍`Memory.allocUtf8String` ساخته شود و مقدار Pointer آن ذخیره و با مقدار قبلی جایگزین شود.

برای مثال ما میتوانیم مقدار Argument تابع `printf` را تغییر دهیم.

```js
// index.js

var text = Memory.allocUtf8String("Zzz for %d; %s");

Interceptor.attach(Module.findExportByName(null, "printf"), {
        onEnter: function(args){
	        args[0] = text;
        }
});
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/printf.png" width=750 >
</div>

تغییر مقدار Retval

این هم تقریبا شبیه به مورد قبلی هست ولی اینبار بجای استفاده از تابع `onEnter` از تابع `onLeave` استفاده میکنیم

اینبار از تابع `ctime` استفاده میکنیم تا مقدار Retval آن را تغییر دهیم.
```js
// index.js

var text = Memory.allocUtf8String("500 BCE :)\n");

Interceptor.attach(Module.findExportByName(null, "ctime"), {
    onLeave: function(retval){
	    retval.replace(text);
    }
});
```

که خروجی آن به اینصورت میباشد

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/ctime.png" width=750 >
</div>


## Bonus

در این بخش یک برنامه جدید به‌همراه یک API جدید را بررسی خواهیم کرد.
سعی کنید به کد نگاه نکنید (برای اینکه جواب سوال لو نرود) و آن را کامپایل کنید.

```c
// crypt.c

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

const int open = 1111 ^ 2355;

int test_pin(char *p) {
    int p_int = atoi(p);

    if (p_int == open) {
        return 1;
    }

    return 0;
}

int main(int argc, char **argv) {

    int access = 0;
    char pin[10];

    while (access == 0) {
        printf("Pin: ");
        fgets(pin, sizeof(pin) - 1, stdin);
        if (test_pin(pin) == 1)
            access = 1;
    }

    printf("Pwnd!!\n");

    return 0;
}
```

برنامه بالا را اجرا کنید و چندبار آن را امتحان کنید تا کاربردش به دستتان بیاید.

برنامه بالا یک پین کد از شما میگیرد و آن را با رمز اصلی خود مقایسه میکند، در صورت برابر بودن این دو عدد، برنامه با موفقیت به پایان میرسد.

حالا با این تصور که ما به کد برنامه دسترسی نداریم، سعی کنید با بررسی Disassembly برنامه برای سوالات زیر پاسخی پیدا کنید:
- در صورت صحیح بودن پین کد چه اتفاقی رخ میدهد؟
- تابع `test_pin` در صورت صحیح بودن پین کد چه مقداری را برمیگرداند؟
- تابع `test_pin` مقدار ورودی ما را با چه عددی مقایسه میکند؟

جواب سوالات بالا هم به این صورت است:
- در صورت صحیح بودن مقدار `Pwnd!!` را چاپ میکند.
- `test_pin` در صورت صحیح بودن پین کد مقدار 0x1 را برمیگرداند.
- `test_pin` مقدار ورودی ما را با `0xd64` مقایسه میکند.

این تحلیل Static ما بود، حالا برویم سراغ تحلیل Dynamic.


حالا با استفاده از `interceptor.attach` روی تابع `test_pin` مقدار Argument و Retval تابع را بررسی کنیم.

```js
// index.js

var testPin = DebugSymbol.getFunctionByName("test_pin");

Interceptor.attach(testPin, {
    onEnter: function(args) {
        console.log("test_pin(" + args[0] + ")");
    },
    onLeave: function(retval) {
        console.log(" => ret: " + retval);
    }
});
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/crypt-1.png" width=750 >
</div>

مقدار ورودی تابع، یک آدرس است که از نوع داده `NativePointer` میباشد، برای بررسی آدرس موجود میتوانیم از `hexdump` استفاده کنیم:

```js
//index.js

var testPin = DebugSymbol.getFunctionByName("test_pin");

Interceptor.attach(testPin, {
    onEnter: function(args) {
        console.log("test_pin(" + args[0] + ")");
        console.log(hexdump(args[0]));
    },
    onLeave: function(retval) {
        console.log(" => ret: " + retval);
    }
});
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/crypt-2.png" width=750 >
</div>

یک روش آسان برای برنده شدن، جایگزین کردن مقدار خروجی تابع `test_pin` با `0x1` است.

```js
//index.js

var testPin = DebugSymbol.getFunctionByName("test_pin");

Interceptor.attach(testPin, {
    onLeave: function(retval) {
        retval.replace(ptr(0x1));
    }
});
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/crypt-3.png" width=750 >
</div>

باحال بود :)
ولی مقدار پین کد اصلی چیست؟

اگر میتوانستیم که `test_pin` را خودمان صدا بزنیم و آن را Brute force کنیم، مساله حل میشد.

Frida باز اینجاست که کمک‌مان کند، اما این بار با استفاده از یک API دیگر `NativeFunction`.

```js
new NativeFunction(address, returnType, argTypes[, abi]);
```

`NativeFunction` به ما کمک میکند تا با مشخص کردن آدرس تابع، نوع داده Retval و نوع داده Argument ها، تابع مورد نظر را داخل کد Javascriptمان صدا بزنیم.

```js
// index.js

var testPinPtr = DebugSymbol.getFunctionByName("test_pin");
var test_pin = new NativeFunction(testPinPtr, "int", ["pointer"]);

var my_pin = Memory.allocUtf8String("1235");

var r = test_pin(my_pin);

console.log("Trying: " + my_pin.readCString());
console.log("Retval: " + r);
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/crypt-4.png" width=750 >
</div>

حالا که میتوانیم `test_pin` را صدا بزنیم، بیایید آن را Brute force کنیم:

```js
// index.js

var testPinPtr = DebugSymbol.getFunctionByName("test_pin");
var test_pin = new NativeFunction(testPinPtr, "int", ["pointer"]);

for (var i=0; i<=9999; i++){

    var my_pin = Memory.allocUtf8String(i.toString());
    console.log("Trying: " + my_pin.readCString());

    var r = test_pin(my_pin);

    if (r == 1){
        console.log("Pin: " + my_pin.readCString());
        break;
    }
}
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-1/crypt-5.png" width=750 >
</div>
این هم از بخش Bonus، ولی Frida اینجا به پایان نمیرسد هدف از نگارش این سری از پست ها آشنایی با Frida و بهانه‌ای برای یادگیری مباحث پایه‌ای آن بود.

## Resources
این سری کوتاه از بلاگ‌پست ها برگرفته از ورکشاپ [frida-boot](https://github.com/leonjza/frida-boot) هستش که نوشته [Leon Jacobs](https://twitter.com/leonjza)، نویسنده ابزار [Objection](https://github.com/sensepost/objection)، هست.
