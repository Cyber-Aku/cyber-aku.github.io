---
layout: post
title: From LD_PRELOAD To Frida [0x0|FA]
tags: LD_PRELOAD Frida Hooking
---

<style>
p,ul,h2    {direction: rtl;}
</style>

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/poster.png" width=750 >
</div>

## مقدمه

خب مساله از جایی شروع میشود که دیگر انسان ها نمیخواهند توابع سابق در برنامه ها را اجرا کنند، یا از سر کنجکاوی میخواهند مقادیر ورودی و خروجی توابع را بررسی کنند، البته روش ها و ابزار هایی که ما در این چند پست قرار است بررسی کنیم قادر به کار هایی فراتر از اینهاست.

اول از همه، برای اینکه برای مشکلی راه حلی پیدا کنیم، باید به‌خوبی آن مشکل یا مساله را شناسایی کنیم، در بخش قبل تقریبا با چیزی که میخواهیم حلش کنیم آشنا شدیم، آنور آبی ها به آن Hooking میگویند.

خب بیایید برای شما یک مساله موقت طرح کنم تا با هم برای حلش تلاش کنیم.

```c
// sleepy.c

#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <time.h>

int rand_range(int min, int max) {
    return min + rand() % (max+1 - min);
}

int main() {

    printf("[+] Starting up!\n");

    int d;
    time_t tm;
    srand(time(NULL));

    for (int i = 0; i < 100; i++){
        d = rand_range(1, 5);

        time(&tm);

        printf("[*] Sleeping for %d seconds; %s", d, ctime(&tm));
        sleep(d);
    }

    printf("[+] Done!\n");
}
```

این برنامه یک عدد تصادفی بین ۱ تا ۵ را انتخاب میکند و به‌اندازه همان مقدار صبر میکند و سپس پیغامی را 
چاپ میکند.<br/>

برای کامپایل کردن کد فوق نیز باید از دستور زیر استفاده کنید:

```
gcc sleepy.c -o sleepy
```

هدف ما این است که هر چه سریع از این حلقه خارج شویم و مدت زیادی صبر نکنیم، راه‌حل چیست؟

یک روش میتواند پیاده‌سازی مجدد تابع `sleep` در libc و کامپایل کردن مجدد آن باشد یا میتوانیم با ماشین زمانی که در حیاط پشتی خانه‌مان داریم به آینده سفر کنیم.<br/>
ولی روش های بهتری هم هستند.

## LD_PRELOAD

اصلا `LD_PRELOAD` چیست؟<br/>
`LD_PRELOAD` یک Environment variable هست که Dynamic linker/loader مقدار آن را قبل اجرای برنامه میخواند و در صورت اینکه قرار باشد تابع‌ای به صورت Dynamic آدرس دهی شود، آدرس توابع موجود در Shared object ما را در اولویت اول قرار میدهد.

پس علت وجود `LD_PRELAOD` تغییر توابع برای برنامه های Dynamic است؛ فرض کنید ما میخواهیم تابع `write` در libc را به تابعی که خودمان که نوشته‌ایم تغییر دهیم؛ تصور کنید که اگر پس کامپایل کتابخانه، تابع‌ای که ما نوشته‌ایم به‌درستی کار نکند چه فاجعه‌ای رخ میدهد، تمام برنامه هایی که از `write`  استفاده میکنند دیگر به‌درستی کار نخواهند کرد.<br/>
و قهرمان ما `LD_PRELOAD` اینجاست که این مشکل را حل کند؛ با کمک `LD_PRELOAD` میتوانیم تغییراتمان را فقط روی یک برنامه اعمال کنیم.

اگه نمیدونین Dynamic linker/loader چی هستش توصیه میکنم درباره Linux Process Loading یه مطالعه داشته باشین.

بیاید برای مساله مطرح شده یک راه حل با `LD_PRELOAD` بسازیم.<br/>
ما میخواهیم تابع `sleep` هر چه زودتر تمام شود، پس تابع هدفمان `sleep` است.

برای اینکه بتوانیم رفتار `sleep` را تغییر بدهیم باید پیاده‌سازی اصلی تابع هدفمان را بشناسیم، تا بتوانیم آن را تقلید کنیم، برای اینکار به کل بدنه تابع نیاز نداریم، چون قرار است بدنه را خودمان پیاده‌سازی کنیم، ما فقط به Function signature تابع نیازمندیم.

Function signature به ما نوع داده هایی که قرار است تابع به عنوان ورودی و نوع داده‌ای که قرار است برگرداند را مشخص میکند.

برای اینکه بتوانیم Function signature مربوط به توابع داخل libc را دریافت کنیم کافیست از دستور ‍`man` کمک بگیریم.<br/>
Function signature مربوط به تابع `sleep` به این صورت است:

<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/sleep_man.png" width=750 >
</div>

این تابع به عنوان ورودی داده‌ای با نوع Unsigned int دریافت میکند، که مقدار زمانی هست که باید صبر کند.<br/>
و به عنوان خروجی یک Unsigned int تحویل میدهد، که اگر  مقدارش صفر باشد به این معنی هست که تابع با موفقیت اجرا شده، در غیر اینصورت، مقدار زمانی که هنوز باقی‌مانده را برمیگرداند (این حالت وقتی اتفاق میوفتد که یک Signal handler در اجرای `sleep` اخلالی ایجاد کند).


خب حالا که Function signature مربوط به `sleep` را بدست آوردیم میتوانیم کد خودمان را پیاده‌سازی کنیم.

```c
// fake_sleep.c

#include <stdio.h>

unsigned int sleep (unsigned int sec){
    printf("too much caffeine, I can't sleep");
}
```


حالا باید کد را بصورت یک Shared object کامپایل کنیم، که با دستور زیر انجام میشود:


```
gcc -fPIC -shared fake_sleep.c -o fake_sleep.so
```

و برای اجرای برنامه‌مان کافیست که Shared objectای که ساختیم را به‌عنوان `LD_PRELOAD` به برنامه‌مان بدهیم.

```
LD_PRELOAD=./fake_sleep.so ./sleepy
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/fake_sleep.png" width=550 >
</div>

خیلی سریع بود مگه نه :) ؟

چون ما الان جای تابع `sleep` را گرفته‌ایم طبیعتا میتوانیم Argument های آن را بخوانیم یا Retval آن را تغییر دهیم.<br/>
مثال زیر Argument داده شده به تابع را چاپ میکند.

```c
// fake_sleep2.c

#include <stdio.h>

unsigned int sleep (unsigned int sec){
    printf("sleep(%d)\n", sec);
}
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/fake_sleep2.png" width=550 >
</div>

اینبار بیایید Retval تابع `ctime` را تغییر دهیم تا بجای تاریخ، رشته موردنظر ما را برگرداند.

```c
// fake_ctime.c

#include <time.h>

char* ctime(const time_t *timep) {
    char *buffer = "where is my ctime\n";

    return buffer;
}

```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/fake_ctime.png" width=550 >
</div>

حالا اگر نخواهیم در این حد هم سریع تمامش کنیم چه؟ مثلا بدون توجه به اعداد تصادفی برای همه یک ثانیه ثابت اعمال کنیم.<br/>
کنترل `sleep` دست ماست، هر چه که پادشاه امر کند :)

برای انجام اینکار ما باید تابع `sleep` اصلی (متعلق به libc) را از داخل `sleep`ای که خودمان ساختیم صدا بزنیم.<br/>
برای همین هم باید آدرس `sleep` اصلی را پیدا کنیم (در غیر اینصورت یک Recursion رخ میدهد).

برای اینکار هم باید از `dlsym` کمک بگیریم.<br/>
`dlsym` به ما کمک میکند تا آدرس یک Symbol را از داخل یک Shared object یا برنامه پیدا کنیم.

```c
// fake_sleep3.c

#include <stdio.h>
#include <dlfcn.h>

unsigned int sleep (unsigned int sec){

    unsigned int (*libc_sleep) (unsigned int);

    libc_sleep = dlsym(RTLD_NEXT, "sleep");

    libc_sleep(1);
}
```

اینبار برای کامپایل کردن کدمان باید از دستور زیر استفاده کنیم:

```
gcc -fPIC -shared fake_sleep3.c -o fake_sleep3.so -ldl
```
<br/>
<div style="text-align: center">
    <img src="/assets/images/from-ld-preload-to-frida-0/fake_sleep3.png" width=550 >
</div>

حالا بدون توجه به مقداری که به تابع داده میشود ما یک ثانیه ثابت را برای همه اعمال میکنیم.

---

در قسمت بعد سراغ Frida خواهیم رفت تا ببینیم که چقدر کار ما را راحت‌تر کرده.
