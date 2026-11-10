---
chapter: 6
season: 1
title: "CSRF"
status: draft
---

# CSRF (Cross-Site Request Forgery)

অধ্যায় ৫ একটা প্রশ্নে শেষ হয়েছিল, আর আরিফ প্রশ্ন ঝুলিয়ে রাখার মানুষ নয়।

*যে আক্রমণকারী আমাদের session ID কখনো দেখেনি, সে কি তবু শুভর browser দিয়ে শুভর নামে request পাঠাতে পারে?*

পরদিন সকালে সে staging-এ রাফির (`user-1`, Owner) নামে login করল। নিজের laptop-এ একটা test পাতা বানাল, যার ভেতরে লুকোনো একটা form, `action` সোজা staging-এর `POST /manage/orders/1051/cancel`, আর একটা এক লাইনের JavaScript যা পাতা খুললেই form-টা submit করে দেয়। তারপর অন্য একটা ট্যাবে সেই test পাতা খুলল।

Server-এর log-এ request এল। কিন্তু cookie ছাড়া।

```text
POST /manage/orders/1051/cancel   cookie নেই   401
```

এই opening ঘটনাটি KenaKata-র fictional staging scene, বাস্তব browser test-এর report নয়। আধুনিক browser-এ cookie কীভাবে পাঠানো হয়, সেটি নির্দিষ্ট supported-browser matrix-এ আলাদা করে যাচাই করতে হবে.

আরিফ একটু অবাক হয়ে বসে রইল। তারপর হাসল। অধ্যায় ৪-এ সে `SameSite=Lax` লিখেছিল "যেহেতু সবাই লেখে" গোছের মনোভাবে। আজ ওই একটা শব্দ শুভকে, আর রাফিকে, বাঁচাল। Lax-এর নিয়মে অন্য site থেকে আসা `POST`-এ browser cookie পাঠায় না।

"তাহলে CSRF-এর ভয় কেটে গেল," আরিফ নিজেকে বলল। অধ্যায় ৪-এর সতর্কবাণীটা সে মনে করতে চাইল না।

কয়েকদিন পরের এক সকাল সাড়ে নয়টা। আরিফের ফোনে নীরার নাম।

"মিতু নামের এক ক্রেতা রাফিকে ফোন করে রেগে গেছে," নীরা বলল। "তার পাইকারি অর্ডারটা, ১০৫১, নাকি বাতিল হয়ে গেছে। রাফি বলছে সে বাতিল করেনি।"

আরিফ অর্ডারের তালিকা খুলল। ১০৫১-এর পাশে লেখা: *বাতিল*। কে করল? অধ্যায় ৫-এর পর সে জানে প্রথম প্রশ্নটা কী: **session চুরি হয়েছে?** সে log খুলল।

```text
11:42:09Z GET  /manage/orders/1051/cancel   sid_hash=3fa9c2b1d7e5a4c8f2b6d0e9a1c3f507 user=user-1 ip=192.0.2.21
          (reverse proxy-র access log থেকে মিলিয়ে) referer=https://courier-track.example/
```

(IP নমুনা, ডকুমেন্টেশনের জন্য আলাদা করা range থেকে। সময় UTC, অধ্যায় ৫-এর মতোই। `sid_hash` raw session ID নয়; এটি keyed HMAC-ভিত্তিক দীর্ঘ correlation value-এর illustrative উদাহরণ। বাস্তবে key management, retention ও log access আলাদা করে সুরক্ষিত রাখতে হয়।)

প্রতিটা সংকেত আরিফ মিলিয়ে দেখল। IP রাফির দোকানের Wi-Fi-র একই `/24`। User-Agent রাফির browser-এর। অধ্যায় ৫-এ যে context-signal policy ছিল, সেটি কোনো সতর্কতা দেয়নি। রাফি সকাল থেকে কাজ করছিল, শেষ request তিন মিনিট আগের, তাই ৩০ মিনিটের idle ঘড়িও ঠিক ছিল, ৮ ঘণ্টার absolute ঘড়িও। Session সম্পূর্ণ বৈধ। কেউ cookie চুরি করেনি। রাফির password Argon2id-র পেছনে।

শুধু একটা জিনিস খাপ খাচ্ছে না। `referer`। `courier-track.example` KenaKata-র কোনো পাতা নয়।

আরিফ রাফিকে ফোন করল। "কাল বিকেলে কোনো লিঙ্কে ক্লিক করেছিলেন?"

"SMS এসেছিল, পার্সেল ট্র্যাকিংয়ের," রাফি বলল। "ক্লিক করতেই একটা নতুন ট্যাব ঝট করে খুলল, কী যেন লেখা এল, বন্ধ করে দিলাম। ভাবলাম আজেবাজে।"

আরিফ চুপ করে গেল। গত সপ্তাহের কাজটা তার মনে পড়ছিল। নীরা চেয়েছিল, নতুন অর্ডারের ইমেইলে একটা "বাতিল করুন" লিঙ্ক থাকুক, যাতে রাফি ফোন থেকে এক ক্লিকে বাতিল করতে পারে। আরিফ বানিয়েছিল:

```http
GET /manage/orders/1051/cancel HTTP/1.1
Cookie: __Host-id=<browser-এর পাঠানো session cookie>
```

এই request-এ আগের স্তরগুলোর কোনো ভুল ছিল না, অন্তত যে ভুল আমরা এতদিন খুঁজেছি। অধ্যায় ৪-এর authentication check ঠিকঠাক কাজ করেছে। session বৈধ ছিল। `user-1` সত্যিই অর্ডার বাতিল করতে পারে, অর্থাৎ অধ্যায় ২-এর authorization-ও আপত্তি তোলেনি। অধ্যায় ৫-এর সব নতুন যন্ত্র (মেয়াদ, rotate, context signal) নিজের কাজ ঠিকঠাক করেছে।

আর যে প্রশ্নটা অধ্যায় ৪ থেকে ঝুলছিল, আর অধ্যায় ৫-এর শেষে আরও তীক্ষ্ণ হলো, সেটা এখন ঘরের ভেতরে বসে আছে। server-এর চোখে দুটো request হুবহু এক: একটা রাফি নিজে করেছে, আরেকটা রাফির browser-কে দিয়ে অন্য কেউ করিয়েছে। **তফাতটা server কীভাবে ধরবে?**

---

## এই অধ্যায়ে কী বুঝবে

- Browser-এর স্বয়ংক্রিয় credential পাঠানো কীভাবে CSRF-এর সুযোগ তৈরি করে।
- CSRF token, Origin/Fetch Metadata, SameSite এবং re-authentication কীভাবে স্তর হিসেবে কাজ করে।
- **CORS (Cross-Origin Resource Sharing)**, authentication ও CSRF defense কেন একে অন্যের বিকল্প নয়।

## ১. ঘটনাটা ছবিতে

আক্রমণকারী যদি cookie না জানে, তাহলে request-এ cookie গেল কীভাবে? ছবিতে দেখো কে কী করল, আর কে কী কখনো দেখল না।

```mermaid
sequenceDiagram
    actor Rafi as রাফি
    participant B as রাফির browser
    participant E as আক্রমণকারীর পাতা
    participant K as KenaKata server

    Rafi->>B: SMS-এর লিঙ্কে ক্লিক
    B->>E: GET courier-track.example/p/88213
    E-->>B: HTML আর JavaScript
    Note over B,E: JavaScript নতুন ট্যাবে KenaKata-র cancel URL খুলতে বলে
    B->>B: cookie jar-এ kenakata.example-এর cookie খোঁজে
    B->>K: GET /manage/orders/1051/cancel<br/>Cookie: __Host-id=...<br/>Referer: https://courier-track.example/
    K->>K: session বৈধ, user-1। অর্ডার বাতিল
    K-->>B: 200, অর্ডার বাতিল হয়েছে
    Note over E: আক্রমণকারী cookie দেখেনি, response-ও দেখেনি
```

ছবির তিনটা জিনিস মনে রাখার মতো।

**এক,** আক্রমণকারী cookie চুরি করেনি। দেখেওনি। `HttpOnly` থাকলেও এই ঘটনা ঘটত, কারণ cookie পড়ার দরকারই পড়েনি। request-এ cookie জুড়েছে **browser** নিজে।

**দুই,** request এসেছে রাফির browser থেকে, রাফির network থেকে, রাফির session নিয়ে। অধ্যায় ৫-এর context signal (network, User-Agent) এখানে অন্ধ, কারণ বদলানোর মতো কিছু বদলায়নি। "অন্য জায়গা থেকে এসেছে" ধরনের কোনো সংকেত নেই। শুধু `referer` একটা ক্ষীণ ছাপ রেখে গেছে।

**তিন,** আক্রমণকারী response-ও দেখেনি। এই হামলা অন্ধ। সে কেবল একটা কাজ করিয়ে নিয়েছে।

এই আক্রমণের নাম **CSRF** (Cross-Site Request Forgery)। OWASP-র ভাষ্য অনুযায়ী: ক্ষতিকর কোনো সাইট, ইমেইল বা প্রোগ্রাম authenticated user-এর browser দিয়ে বিশ্বস্ত সাইটে একটা অনাকাঙ্ক্ষিত কাজ করিয়ে নেয়, আর সাইট আসল আর জাল request আলাদা করতে পারে না।

অধ্যায় ১-এর wristband-এ ফিরি। session ID রাফির কবজিতে বাঁধা। আক্রমণকারী সেটা খোলেনি। সে রাফির **হাতকে** বলেছে, "ওই দরজায় টোকা দাও।" হাত টোকা দিয়েছে, আর কবজির wristband দেখিয়ে দিয়েছে। দরজার পাহারাদার (`requireAuthentication`) wristband দেখল, নম্বর মিলল, দরজা খুলে দিল। সে একটা প্রশ্নই করে: *এই wristband কার?* সে কখনো জিজ্ঞেস করে না: *এই টোকাটা কি রাফি নিজে দিতে চেয়েছিল?*

Authentication বলে *কে*। CSRF-এর পুরো গল্প *ইচ্ছা* নিয়ে।

---

## ২. Browser কেন পাঠাল?

অধ্যায় ৪-এর cookie-attribute অংশে আমরা দেখেছিলাম browser কখন cookie পাঠায়। ছবিটা আবার মনে করো। প্রশ্নগুলো ছিল: host মেলে? path মেলে? `Secure` আর সংযোগ মেলে? আর শেষে `SameSite`। একটা প্রশ্ন কোথাও ছিল না: **এই request কে শুরু করেছে, রাফি নিজে, না অন্য কোনো পাতা?** (`SameSite` একটা মোটা রেখা টানে, অন্য *site* থেকে এসেছে কি না। তার বেশি কিছু নয়।)

কারণ cookie jar বাঁধা থাকে **গন্তব্যের** সঙ্গে (`kenakata.example`), **উদ্যোক্তার** সঙ্গে নয়। অধ্যায় ৪-এ এই বৈশিষ্ট্যের নাম বলেছিলাম **ambient authority**: কে request শুরু করল, সেটা না দেখেই browser authority জুড়ে দেয়। OWASP একই কথা বলে: browser-এর প্রতিটা request-এ cookie আপনা-আপনি যায়, তাই সাইট যদি requester-এর পরিচয় আর ক্ষমতা শুধু cookie দিয়ে যাচাই করে, এই আক্রমণ চলে।

"কিন্তু same-origin policy?" প্রশ্নটা আসবেই। SOP আর CORS মূলত অন্য origin-এর **response পড়া** আটকায়। আক্রমণকারীর response পড়ার দরকারই নেই। আর MDN-এর বিবরণ অনুযায়ী `<form>` submit-এর মতো "simple request" cross-origin পাঠানো যায়, কারণ web-এর শুরু থেকে form তা পারত, আর compatibility-র কারণে সেটা ভাঙা হয়নি।

এই দৃশ্যকে তিনটি প্রশ্নে নামিয়ে আনা যায়: (১) request কি server-side state বদলাতে পারে? (২) browser কি নিজে থেকে authentication credential জুড়ে দেয়? (৩) আক্রমণকারী কি server-এর গ্রহণযোগ্য request তৈরি করাতে পারে, এমন কোনো গোপন প্রমাণ ছাড়াই যা তার জানা নেই? প্রতিরক্ষা এক বা একাধিক শর্ত ভাঙবে। এটি শেখানোর জন্য সরল model, প্রতিটি বাস্তব CSRF ঘটনার সর্বজনীন সংজ্ঞা নয়:

| # | শর্ত | KenaKata-র আজকের অবস্থা | কোন প্রতিরক্ষা এটা ভাঙে |
| - | ---- | ----------------------- | ----------------------- |
| ১ | HTTP request দিয়ে server-এ state বদলায় | হ্যাঁ, অর্ডার বাতিল | নিরাপদ method-এ state বদলানো বন্ধ (§৪.১) |
| ২ | request কার, তা যাচাই হয় শুধু cookie দিয়ে | হ্যাঁ, `__Host-id` | request-এর উৎস যাচাই (§৪.২), আর cookie ছাড়া আরেকটা প্রমাণ (§৪.৩) |
| ৩ | request গ্রহণ করতে server-এর কাছে কোনো অজানা secret proof লাগে না | হ্যাঁ, URL-ই যথেষ্ট | session-নির্দিষ্ট, অনুমান-অযোগ্য CSRF token (§৪.৩) |

`SameSite` শর্ত ২-কে আংশিক ভাঙে: কিছু ক্ষেত্রে cookie-ই পাঠাতে দেয় না।

---

## ৩. আরিফের প্রথম ফিক্স, আর তার ফাটল

আরিফের প্রথম চালটা সবচেয়ে স্বাভাবিক: GET বদলে POST।

নীরা আপত্তি করল। "ইমেইলের ভেতর তো POST বাটন বসানো যায় না।" আরিফের সমাধান: ইমেইলের লিঙ্ক GET-ই থাকুক, কিন্তু সেটা শুধু অর্ডারের পাতা খুলুক, যেখানে "বাতিল করুন" একটা POST বাটন। এক ক্লিকের বদলে দুই ক্লিক। নীরা গজগজ করল, তারপর রাজি হলো।

(এই "দুই ক্লিক" নিজে CSRF-এর প্রতিরক্ষা নয়। কাজ করছে দ্বিতীয় ধাপের POST-টা, যেটা নিচে ধাপে ধাপে সুরক্ষিত হবে। শুধু ধাপ বাড়ালে আক্রমণকারী ধাপগুলোও পাঠাতে পারে।)

অধ্যায়ের শুরুতে আমরা দেখেছি, `POST` form-এর হামলা `SameSite=Lax` আটকেছে, `POST` নিজে নয়।

তারপর ক্ষতি হওয়ার পর আরিফ অধ্যায় ৪-এর সতর্কবাণীটা সত্যিই পড়ল: `SameSite` CSRF-এর শক্ত সমাধান নয়, বড়জোর defense in depth। সে আবার আক্রমণকারীর চেয়ারে বসল। প্রশ্ন: **Lax-কে ফাঁকি দেওয়ার পথ কয়টা?**

| হামলার ধরন | `Lax` cookie যায়? | কেন |
| ---------- | ------------------ | --- |
| অন্য সাইটের auto-submit `POST` form | না | Lax: cross-site `POST`-এ cookie যায় না |
| অন্য সাইট থেকে top-level navigation, method `GET` | **হ্যাঁ** | Lax: top-level navigation আর safe method, দুটোই মিলেছে। **আজকের ঘটনা** |
| অন্য সাইটের `<img>`, `<iframe>` (subresource, nested) | না | top-level navigation নয় |
| **সহোদর subdomain** থেকে যে কোনো request, `POST`-ও | **হ্যাঁ** | সেটা same-site। `SameSite`-এর চোখে "cross" নয় |
| `GET` পাঠানো হলো, কিন্তু framework-এর *method override* সেটাকে `POST` বানিয়ে নেয় | **হ্যাঁ** | top-level `GET` হিসেবে cookie যায়, framework ভাবে `POST` |
| `SameSite` না বোঝা বা না মানা client (পুরোনো বা embedded browser) | ধরে নাও হ্যাঁ | OWASP: এমন client-এ cookie যেন `SameSite` ছাড়াই আচরণ করে |

দ্বিতীয় সারির দিকে তাকাও। অধ্যায় ৪ §৭-এ আমরা `Lax` বেছেছিলাম যেন রাফির কর্মীরা WhatsApp-এর লিঙ্ক ধরে এসে login পাতায় আটকে না যায় (অধ্যায় ৫-এ নীরার "login-এর পর ঠিক ওই পাতায় ফিরুক" দাবিটাও এই লিঙ্কের গল্প)। ঠিক সেই ছাড়টাই আজ আক্রমণকারীর পথ। এটা `Lax`-এর দোষ নয়। এটা `Lax`-এর **চুক্তি**: browser বলছে, "তুমি যদি `GET`-এ state বদলাও, ঝুঁকিটা তোমার।"

চতুর্থ সারির `blog.kenakata.example` অধ্যায় ৪-এর মতোই কাল্পনিক, KenaKata-র আসল নকশা নয়। ধরো কাল কেউ সেখানে একটা পুরোনো CMS বসাল, আর তাতে একটা দুর্বলতা আছে। সেখান থেকে যাওয়া request মূল সাইটের কাছে same-site, তাই `Lax` কেন, `Strict` cookie-ও সঙ্গে যায়। এই জন্যই অধ্যায় ৪-এ `Domain` না দেওয়ার কথা বলেছিলাম: না দিলে cookie সহোদরদের সঙ্গে ভাগ হয় না, কিন্তু সহোদর যে *পাঠাতে* পারে, সেটা `Domain` না দিলেও বদলায় না। এই দুটো আলাদা প্রশ্ন।

### Referer দেখলে কেমন হয়?

আরিফের দ্বিতীয় চিন্তা: log-এ তো `referer`-ই ছিল। server সেটা দেখে অচেনা হলে ফিরিয়ে দিক। ধারণাটা আধা-ঠিক। সমস্যা তিনটা। `Referer` অনেক জায়গায় থাকে না (privacy, proxy, load balancer, OWASP-র তালিকা)। যাচাইয়ে সহজ ভুল হয়: `kenakata.example.evil.example` ধরনের নাম মিলে যায় যদি prefix দিয়ে মেলাও। আর `Origin: null` এলে কী করবে? OWASP স্পষ্ট বলে, header না থাকা বা `Origin: null` same-origin-এর প্রমাণ নয়।

তার চেয়ে ভালো সংকেত আছে। browser নিজে বসায়, আর পাতার JavaScript বদলাতে পারে না। সেটাই এখন দরকার।

---

## ৪. ভালো সমাধান: একটা দেয়াল নয়, কয়েকটা

আরিফ ঠিক করল, একটা প্রতিরক্ষার ওপর ভরসা নয়। প্রতিটা দেয়াল আলাদা ভাবে ব্যর্থ হয়, তাই স্তর চাই। OWASP-ও একই পথ দেখায়: framework-এর নিজস্ব CSRF সুরক্ষা থাকলে সেটা, নইলে token, তার সঙ্গে অন্তত একটা defense in depth।

### ৪.১ স্তর ০: নিরাপদ method-এ state বদলাবে না

`GET`, `HEAD`, `OPTIONS` হলো safe method। নিয়ম সোজা: এগুলোতে কিছু বদলানো চলবে না। কারণ `Lax`-এর চুক্তি তো আছেই, আর এর পরের স্তরগুলোও safe method-কে সরাসরি পার করে দেয়। Go-র standard library-র নিজের ডকুমেন্টেশনও সতর্ক করে: safe method-এর request দিয়ে state বদলালে এই সুরক্ষা কাজ করবে না। OWASP-র কথাও এক: state বদলাতে `GET` নয়।

অধ্যায় ৪-এর একটা ঋণ এখানে শোধ হয়। সেখানে বলেছিলাম, logout `POST`, `GET` নয়, কেন তা অধ্যায় ৬-এ। কারণ logout-ও state বদলায় (খাতা থেকে session মোছে)। `GET /logout` হলে যে কোনো সাইট রাফিকে চুপচাপ logout করিয়ে দিতে পারত। ক্ষতি ছোট, কিন্তু নীতি আজকের ঘটনার মতোই। আর ছোট ক্ষতি প্রায়ই বড় হামলার প্রথম ধাপ (§৫)।

একটা সতর্কতা: কিছু framework-এ *method override* থাকে (`?_method=POST` ধরনের)। MDN দেখায়, আক্রমণকারী `GET` পাঠিয়ে সেটাকে server-এর চোখে `POST` বানাতে পারে। ওই ফিচার কখনো চালু করলে `GET`-এ তা মানবে না।

ফলে ইমেইলের লিঙ্ক এখন অর্ডারের পাতা খোলে (শুধু দেখা), আর "বাতিল করুন" একটা `POST` বাটন। বাকি স্তরগুলো সেই `POST`-কে ঘিরে।

### ৪.২ স্তর ১: request কোথা থেকে এল? (Fetch Metadata আর Origin)

browser কিছু header নিজে বসায়, যেগুলো পাতার JavaScript বদলাতে পারে না। দুটো কাজে লাগবে:

| header | কী বলে | মান |
| ------ | ------ | --- |
| `Sec-Fetch-Site` | request-এর উৎস আর গন্তব্যের সম্পর্ক | `same-origin`, `same-site`, `cross-site`, `none` (user নিজে: bookmark, address bar) |
| `Origin` | উৎসের origin (scheme, host, port) | যেমন `https://courier-track.example` |

OWASP `Sec-Fetch-Site`-কে প্রধান সংকেত বলে। `Sec-` দিয়ে শুরু হওয়া request header browser-এর `fetch` API থেকে JavaScript বদলাতে পারে না, কারণ Fetch Standard এগুলোকে forbidden request header হিসেবে সংরক্ষণ করে। তবে non-browser client নিজের ইচ্ছায় এই header লিখতে পারে; তাই header একা authentication নয়। OWASP Fetch Metadata না থাকলে origin যাচাইয়ের fallback রাখতে বলে। KenaKata-র policy-তে `Sec-Fetch-Site` অনুপস্থিত হলে server-configured target origin-এর সঙ্গে `Origin` হুবহু মেলানো হবে; `Origin`-ও অনুপস্থিত বা `null` হলে request প্রত্যাখ্যাত হবে।

আরিফের সিদ্ধান্তগুলো ছবিতে। প্রশ্ন: একটা `POST` কোন কোন পরীক্ষা পেরোলে ঢুকতে পারে?

```mermaid
flowchart TD
    R["POST /manage/orders/1051/cancel আসছে"] --> M{"method কি GET, HEAD বা OPTIONS?"}
    M -- "হ্যাঁ" --> OK1["পার হয়। তাই এখানে state বদলানো চলবে না"]
    M -- "না" --> S{"Sec-Fetch-Site কী?"}
    S -- "same-origin" --> OK2["পরের স্তরে যায়: session ও token যাচাই"]
    S -- "cross-site, same-site, none বা অচেনা মান" --> NO1["403, প্রত্যাখ্যান"]
    S -- "header নেই" --> O{"Origin হুবহু https://kenakata.example?"}
    O -- "হ্যাঁ" --> OK3["পরের স্তরে যায়: session ও token যাচাই"]
    O -- "না, খালি বা null" --> NO2["403, প্রত্যাখ্যান"]
```

নীতিটা সংক্ষেপে:

**KenaKata-র policy table** (এটি শিক্ষামূলক সিদ্ধান্ত, কোনো framework-এর universal default নয়):

| State-changing request-এ কী দেখা গেল | সিদ্ধান্ত | কারণ |
| --- | --- | --- |
| `Sec-Fetch-Site: same-origin` | পরের স্তরে যাও | browser জানায় request একই origin থেকে এসেছে; token-ও যাচাই করতে হবে |
| `cross-site`, `same-site`, `none` বা অচেনা value | `403` | `none` user-initiated navigation-এর সংকেত হতে পারে, কিন্তু একে একা sensitive action-এর অনুমতি হিসেবে ধরা হবে না |
| `Sec-Fetch-Site` নেই, `Origin` exact configured origin-এর সঙ্গে মেলে | পরের স্তরে যাও | compatibility fallback |
| `Sec-Fetch-Site` নেই এবং `Origin` missing, `null` বা mismatch | `403` | KenaKata fail-safe বেছে নিয়েছে |

KenaKata-র policy ইচ্ছাকৃতভাবে কঠোর: OWASP-এর সাধারণ Fetch Metadata guidance forward compatibility-র জন্য অচেনা value ignore করতে বলে, কিন্তু এখানে unknown value-ও reject করা হচ্ছে। এতে ভবিষ্যতে নতুন value এলে বৈধ request-ও block হতে পারে; production-এ enforce করার আগে log-only rollout ও compatibility monitoring দরকার।

`GET`, `HEAD` ও `OPTIONS`-এর মতো safe method-এ state বদলানো যাবে না। HTTP semantics-এ `TRACE`-ও safe method হিসেবে সংজ্ঞায়িত, তবে KenaKata-র এই দৃশ্যে সেটি চালু endpoint নয়। Safe method-এ state বদলানো থাকলে এই guard-কে পাশ কাটানো যায়, তাই handler-এর আচরণই প্রথম invariant.

প্রতিটা সিদ্ধান্তের পেছনে একটা trade-off:

**`same-site` প্রত্যাখ্যান।** MDN-এর Express উদাহরণ `same-site`-ও ঢুকতে দেয়। OWASP বলে, শুধু তখনই দাও যখন সহোদর subdomain-গুলোকে বিশ্বাস করো। দুই উৎসের সিদ্ধান্ত আলাদা, কারণ threat model আলাদা। আমাদের কাল্পনিক `blog.` আছে, তাই আমরা কঠোর।

**target origin কনফিগে।** request-এর `Host` বা proxy-র header থেকে না নিয়ে server-এ লিখে রাখা। OWASP একে সবচেয়ে নিরাপদ পথ বলে, কারণ proxy-র পেছনে `Host` বদলে যায়। দাম: প্রতিটা পরিবেশে (dev, staging, production) আলাদা মান।

**দুটো header-ই না থাকলে প্রত্যাখ্যান (fail-safe)।** OWASP দুই পথ দেয়: fail-safe (সংবেদনশীল endpoint-এ সুপারিশ) আর fail-open (compatibility-র জন্য)। KenaKata-র এখনকার সব client browser, তাই আমরা fail-safe। দাম: `curl` বা ভেতরের কোনো script থেকে সরাসরি POST করলে `Origin` নিজে বসাতে হবে।

**`Vary` header।** এটা response-এ বসে, সিদ্ধান্তের পরে। প্রতিরক্ষায় কিছু বদলায় না। কিন্তু response যদি এই header-ভেদে আলাদা হয়, cache যেন তা গুলিয়ে না ফেলে। OWASP এটা rollout-এর পরামর্শে রাখে।

**Framework-এর built-in protection।** কোনো framework-এ CSRF protection থাকলে সেটি সাধারণত ভালো starting point, তবে তার default policy ও compatibility behavior বুঝে নিতে হবে। বিশেষ করে metadata header অনুপস্থিত হলে request allow হবে, reject হবে, না token যাচাইয়ে যাবে, সেটা স্পষ্ট করে নাও। KenaKata-র সিদ্ধান্ত হলো state-changing request-এ `Sec-Fetch-Site` অনুপস্থিত থাকলে exact configured `Origin` fallback; দুটোই অনুপস্থিত বা `Origin: null` হলে fail-safe reject।

**rollout।** OWASP-র পরামর্শ: আগে "শুধু log" মোডে চালিয়ে কী কী আটকাত তা পর্যালোচনা করো, তারপর enforce করো। speculative navigation (prefetch/prerender) আর header-মুছে-দেওয়া proxy legitimate request-এ false positive তৈরি করতে পারে। এই chapter কোনো production rollout বা browser test দাবি করছে না.

### ৪.৩ স্তর ২: CSRF token

header তো আছে, আবার token কেন? তিন কারণ। এক, header-ভিত্তিক সুরক্ষা browser আর মাঝের proxy-র আচরণের ওপর দাঁড়িয়ে। token দাঁড়িয়ে আমাদের নিজের application-এ। দুই, OWASP বলে, cookie-ভিত্তিক authentication-এর application-এ token এখনো জরুরি। তিন, সহোদর subdomain থেকে আসা request header-এর চোখে `same-site`, তাই ভুল করে ছাড় দিলে token-ই শেষ বেড়া।

নিয়মটা OWASP-র *synchronizer token pattern*। server একটা অনুমান-অযোগ্য মান বানিয়ে session-এর সঙ্গে রাখে, পাতায় বসায়, request-এর সঙ্গে ফেরত নেয়, আর মেলায়। আক্রমণকারী পাতাটা পড়তে পারে না (SOP), তাই মানটা জানে না। ছবিতে দুটো দৃশ্য: আসল ব্যবহার আর জাল চেষ্টা।

```mermaid
sequenceDiagram
    actor Rafi as রাফি
    participant B as রাফির browser
    participant K as KenaKata server
    participant A as আক্রমণকারীর পাতা

    Rafi->>B: অর্ডার ১০৫১-এর পাতা খোলে
    B->>K: GET /manage/orders/1051 আর Cookie
    K-->>B: HTML, ভেতরে hidden field csrf=T
    Note over B,K: T শুধু এই session-এর। অন্য origin-এর পাতা এই response পড়তে পারে না
    Rafi->>B: বাতিল বাটনে ক্লিক
    B->>K: POST cancel, csrf=T, Cookie, Sec-Fetch-Site: same-origin
    K->>K: guard পাস, session বৈধ, T মেলে
    K-->>B: 200

    Note over A,K: একই সময়ে আক্রমণকারীর পাতা
    A->>K: রাফির browser দিয়ে POST cancel, token অজানা
    K-->>A: 403, Sec-Fetch-Site: cross-site, token নেই
```

token-এর ধর্ম (OWASP): প্রতি session-এ আলাদা, গোপন, অনুমান-অযোগ্য (CSPRNG)। synchronizer pattern-এ token **cookie-তে নয়**। URL-এ নয়, log-এ নয়, কারণ `GET` URL browser history, log আর `Referer`-এ ফাঁস হয়। token-কে কোনো *মেয়াদ* দেওয়ার দরকার নেই, session-এর সঙ্গেই তার জীবন (অধ্যায় ৫-এর idle ৩০ মিনিট আর absolute ৮ ঘণ্টা), আর OWASP স্পষ্ট বলে, CSRF token কোনো access token নয়। per-request token বেশি নিরাপদ হলেও "Back" বাটনে ভাঙে, তাই আমরা per-session।

সার্ভারের session record-এ CSRF token-এর জন্য আলাদা field রাখতে হবে। প্রতিটি নতুন session-এর সময় session ID থেকে **স্বাধীন** একটি random মান তৈরি করে সেই session-এর সঙ্গে যুক্ত করতে হবে:

```mermaid
flowchart TD
    A["নতুন anonymous বা authenticated session তৈরি"] --> B["CSPRNG দিয়ে স্বাধীন, অনুমান-অযোগ্য CSRF token তৈরি"]
    B --> C["Server-side session record-এ token রাখা"]
    C --> D["বিশ্বস্ত page-এ hidden form field বা custom header-এ token দেওয়া"]
    D --> E["State-changing request-এ browser token ফেরত পাঠায়"]
    E --> F{"Session-এর token-এর সঙ্গে মেলে?"}
    F -->|হ্যাঁ| G["পরের authorization ও business validation"]
    F -->|না / নেই / session নেই| H["403, request প্রত্যাখ্যান"]
```

কেন স্বাধীন? token পাতার HTML-এ যায়, আর session ID `HttpOnly` cookie-তে। token যদি session ID-র হুবহু রূপ বা তা থেকে বানানো হতো, পাতায় বসানো মান থেকে session ID-র দিকে একটা সূত্র তৈরি হতো। আলাদা random মানে সেই সূত্র নেই। Login-এর আগে anonymous session তৈরি হলে তারও নিজস্ব token থাকবে। Login সফল হলে পুরোনো anonymous session বাতিল করে নতুন session ID ও নতুন token তৈরি করতে হবে। এতে অধ্যায় ৫-এ শেখা session fixation প্রতিরক্ষার সঙ্গে CSRF token lifecycle-ও মিলে যায়। login-এর আগের session-এর token কেন লাগে, সেটা §৫-এ।

Server-side validation ও page form:

Token যাচাইয়ের security invariant:

- Token session ID থেকে **স্বাধীন random value** হবে, session ID থেকে derive করা যাবে না।
- Server token-টিকে session-এর সঙ্গে যুক্ত রাখবে; form field বা custom request header দিয়ে ফেরত আসা মানের সঙ্গে মিলিয়ে দেখবে।
- Missing token, empty token, expired session বা mismatch হলে state-changing action চলবে না।
- Token-কে URL, log, analytics event বা `Referer`-এ ফাঁস হতে দেওয়া যাবে না। HTML form-এ বসালে framework-এর safe templating/escaping ব্যবহার করতে হবে।
- Per-session token সাধারণত browser-এর Back/refresh flow-কে per-request token-এর তুলনায় কম ভাঙে; token আলাদা expiry clock নয়, session lifecycle-এর অংশ.

বাস্তব implementation-এ framework-এর নিরাপদ HTML templating ব্যবহার করতে হবে, missing/empty token স্পষ্টভাবে reject করতে হবে, এবং comparison-এর জন্য প্রতিষ্ঠিত security library-র guidance অনুসরণ করতে হবে। এই chapter কোনো নির্দিষ্ট language API বা timing behavior-এর implementation claim করছে না.

stateless হলে (server-এ session নেই) OWASP-র বিকল্প *signed double-submit cookie*, যেখানে token session-এর সঙ্গে HMAC দিয়ে বাঁধা। সাধারণ (naive) double-submit OWASP-র হিসাবে **নিরুৎসাহিত**: সহোদর subdomain বা অন্য কোনো উপায়ে আক্রমণকারী cookie বসাতে পারলে মিলিয়ে দিতে পারে। আমাদের server-side session আছে, তাই synchronizer সবচেয়ে সরল।

### ৪.৪ স্তর ৩: cookie-র অভ্যাস (অধ্যায় ৪ থেকে যা আছে)

`SameSite=Lax` রাখো। `__Host-` রাখো। `Domain` দিয়ো না। OWASP এই শেষটা নিয়ে সরাসরি সতর্ক করে: `Domain` দিলে সব subdomain cookie ভাগ করে, আর dangling CNAME-ওয়ালা subdomain দখল হলে সেটা বিপদ।

MDN একটা সূক্ষ্ম কৌশল দেয়: পাতা দেখানোর জন্য একটা `Lax` cookie, আর state বদলানোর জন্য আরেকটা `Strict` cookie। আমরা নিইনি। guard আর token আগেই আছে, দুই cookie-র জটিলতা বাড়তি।

### ৪.৫ স্তর ৪: ভয়ংকর কাজে user-কে আবার জড়ানো

OWASP বলে, অতি সংবেদনশীল কাজে re-authentication বা one-time token ধরনের user-নির্ভর প্রতিরক্ষা ভাবো। আর **CAPTCHA নয়**: ওটা বট ঠেকানোর জন্য বানানো, আর মানুষের উপস্থিতির প্রমাণ অন্য session থেকে জোগাড় করা যায়। অর্ডার বাতিল সাধারণ কাজ, এখানে এটা অতিরিক্ত। কিন্তু টাকা বা ক্ষমতা বদলের কাজে লাগবে, অধ্যায় ২৩।

### ৪.৬ সব মিলিয়ে

```mermaid
flowchart LR
    Req["request"] --> G["Origin / Fetch Metadata policy<br/>কোথা থেকে এল?"]
    G --> A["Session authentication<br/>কে? (অধ্যায় ৪)"]
    A --> T["CSRF token validation<br/>পাতার token আছে?"]
    T --> H["handler<br/>অর্ডার বাতিল"]
    G -. "403" .-> X["প্রত্যাখ্যান"]
    A -. "401 বা login-এ redirect" .-> X
    T -. "403" .-> X
```

**Route policy** এক নজরে:

| কাজ | HTTP method | Authentication | CSRF protection |
| --- | --- | --- | --- |
| Login page দেখানো | `GET /login` | anonymous pre-session থাকতে পারে | পরের login `POST`-এর জন্য pre-session token issue |
| Login submit | `POST /login` | credentials যাচাইয়ের আগে anonymous | origin guard + pre-session token |
| Order page দেখা | `GET /manage/orders/{id}` | session | safe method; state বদলাবে না |
| Order বাতিল | `POST /manage/orders/{id}/cancel` | session + authorization | origin guard + session token |
| Logout | `POST /logout` | session | origin guard + session token |

Middleware-এর ক্রম implementation-এ বদলাতে পারে, কিন্তু security invariant বদলাবে না: state-changing handler-এর আগে origin policy, session/authentication, CSRF token এবং resource authorization প্রযোজ্য হবে। Login-এর ক্ষেত্রে authenticated session এখনও নেই, তাই anonymous pre-session token ব্যবহার করা হয়.

Origin/Fetch Metadata policy state-changing request-এ প্রথম দিকেই প্রযোজ্য। `/login`-এ authenticated session এখনও নেই, তাই login CSRF ঠেকাতে anonymous pre-session-এর token আলাদাভাবে যাচাই হয়।

এগুলো conceptual flow, executable implementation নয়। তাই এখানে compile/test ফলের দাবি করা হচ্ছে না। গুরুত্বপূর্ণ failure path হলো: safe method দিয়ে state change করার route থাকলে CSRF guard সেটিকে আটকাবে না; আর আক্রমণকারীর নিজের session-এর token victim-এর session-এ মেলার কথা নয়, কারণ token session-নির্দিষ্ট।

---

## ৫. Login আর logout-ও লক্ষ্য

অনেকে ভাবে, login পাতায় CSRF অর্থহীন, কারণ user তখনো authenticated নয়। OWASP বলে, এই ধারণা সবসময় ঠিক নয়। *Login CSRF*-এ আক্রমণকারী victim-কে **নিজের account-এ** login করিয়ে দেয়। victim তখন অজান্তে আক্রমণকারীর account-এ তথ্য লিখে ফেলতে পারে। OWASP-র উদাহরণ: shopping সাইটে victim নিজের কার্ডের তথ্য ওই account-এ রাখল, আর আক্রমণকারী পরে সেটা ব্যবহার করল।

KenaKata-র এখনকার একটাই দোকান, তাই ঝুঁকিটা কল্পনার। কিন্তু যেদিন একাধিক দোকান আসবে (অধ্যায় ১৬), রাফি অজান্তে অন্য কারও account-এ ক্রেতার তালিকা তুলে দিলে সেটা সত্যিকারের তথ্য ফাঁস।

OWASP-র সুপারিশ: login-এর আগে একটা *pre-session* বানিয়ে তাতে token বসাও, আর login-এর পর সেটা **ধ্বংস করে নতুন session** দাও (নইলে fixation)। এখানে অধ্যায় ৫-এর নকশা আমাদের সোজা সুবিধা দেয়। নীরার "login-এর পর ঠিক ওই পাতায় ফেরো" দাবির জন্য আরিফ anonymous session তৈরি করেছিল, আর login-এর মুহূর্তে পুরোনো session বাতিল করে নতুন session ID দিয়েছিল। pre-session আমাদের আগেই আছে। সেই session-এর record-এ (§৪.৩) একটা token-ও আছে। login পাতা সেটা form-এ বসায়, login handler সেটা মেলায়:

```mermaid
sequenceDiagram
    actor Rafi as রাফি
    participant B as রাফির browser
    participant K as KenaKata server

    B->>K: GET /login
    K-->>B: Anonymous session cookie + login form-এ pre-session token T₁
    Rafi->>B: Email/password ও T₁ submit
    B->>K: POST /login + anonymous cookie + T₁
    K->>K: Origin policy ও pre-session token যাচাই
    K->>K: Credentials যাচাই
    alt যেকোনো যাচাই ব্যর্থ
        K-->>B: প্রত্যাখ্যান, authenticated session তৈরি নয়
    else সব যাচাই সফল
        K->>K: Anonymous session invalid, নতুন session ID ও নতুন token T₂
        K-->>B: Set-Cookie: নতুন session ID, login সফল
    end
```

Login flow-তে origin policy আর pre-session token দুটোই প্রযোজ্য। ফলাফলগুলো implementation test হিসেবে নয়, প্রত্যাশিত behavior হিসেবে পড়ো:

| অবস্থা | ফল |
| ------ | --- |
| cross-site request, সঠিক token থাকলেও | `403`; token origin policy-কে override করে না |
| same-origin signal, কিন্তু anonymous session নেই | `403` |
| token নেই বা ভুল | `403` |
| আক্রমণকারীর **নিজের** anonymous token, victim-এর anonymous cookie-তে | `403` |
| সঠিক origin, pre-session ও credentials | Login সফল; নতুন session ID ও নতুন token। পুরোনো anonymous session ও token অকার্যকর |

শেষ সারির শেষ অংশটা অধ্যায় ৫-এর fixation-এর সঙ্গে মিলে যায়। আক্রমণকারী `X` বসিয়ে রাখলে সে `X`-এর token-ও জানে (একই পাতা চেয়ে নিলেই পাওয়া যায়)। কিন্তু login-এর মুহূর্তে পুরোনো anonymous session `X` বাতিল করে নতুন session ID তৈরি করা হয়, তাই ওই token আর কোনো কাজেই লাগে না।

এর দাম আছে। anonymous session ৩০ মিনিটে মরে (অধ্যায় ৫)। কেউ login পাতা খুলে ৪০ মিনিট পরে submit করলে এখন `403` পাবে, আর তার টাইপ করা email হারাবে। ভালো নকশায় সেই ক্ষেত্রে login পাতা নতুন token সহ আবার দেখানো হয়, email-টা ভরা অবস্থায়। এই অধ্যায়ের conceptual flow-তে সেই user-experience detail নির্দিষ্ট করা হয়নি। আর anonymous session নিজেই একটা পৃষ্ঠ, অধ্যায় ৫ §১৩-এ যা বলা হয়েছে: লক্ষ লক্ষ `GET /login` store ভরাতে পারে। token থাকলে সেই সমস্যা বাড়ে না, কিন্তু কমেও না।

**Logout?** সেটা এখন `POST`, আর origin guard ও session token দুটোর নিচে। নিরাপদ method দিয়ে logout হওয়া উচিত নয়; cross-site request বা token mismatch হলে session অক্ষত থাকবে; বৈধ same-origin request ও সঠিক token-এর পরই session বাতিল হবে। এগুলো প্রত্যাশিত behavior, চালানো test-এর ফল নয়.

---

## ৬. `Authorization` header হলে কী বদলায়?

অধ্যায় ৪-এর §৫-এ একটা প্রশ্ন ঝুলিয়েছিলাম: credential `Authorization` header-এ পাঠালে CSRF-এর কী হয়?

**যা কমে।** JavaScript-ভিত্তিক client যদি Bearer token নিজে `Authorization` header-এ বসায়, browser সাধারণত অন্য site-এর form থেকে সেই header আপনা-আপনি জুড়ে দেয় না। Custom header-এর cross-origin request-এ CORS preflight লাগে, এবং server-কে সেই origin ও header-কে স্পষ্টভাবে অনুমতি দিতে হয়। এতে cookie-র ambient authority-র ঝুঁকি কমে, কিন্তু token কোথায় রাখা হয়েছে, XSS, CORS configuration এবং credential-এর অন্য automatic behavior আলাদা করে দেখতে হয়.

**যা আসে।** তিনটা নতুন প্রশ্ন।

এক, মানটা এখন JavaScript-এর নাগালে থাকতে হয়। কোথায় রাখবে? সেখানে XSS ঢুকলে কী হবে? অধ্যায় ৮ আর ১১।

দুই, CORS ভুল সাজালে দরজা আবার খোলে। MDN বলে, কোনো origin থেকে CSRF সম্ভব হবে যদি server সেই origin-কে `Access-Control-Allow-Origin` আর `Access-Control-Allow-Credentials` দুটোতেই অনুমতি দেয়।

তিন, `Authorization` header সবসময় non-ambient নয়। RFC 7617 §2.2 অনুযায়ী HTTP Basic client একই protection space-এর পরের request-এ challenge ছাড়াই credential আগেভাগে পাঠাতে পারে। Browser-এর নির্দিষ্ট credential-storage behavior implementation-ভেদে যাচাই করতে হয়, তাই সব browser-এর আচরণ একই বলে দাবি করা হচ্ছে না। মূল শিক্ষা: header-এর নাম নয়, credential কে ও কখন স্বয়ংক্রিয়ভাবে request-এ যোগ করছে সেটাই CSRF model-এর জন্য গুরুত্বপূর্ণ.

KenaKata-র এখনকার নকশায় header-ভিত্তিক credential নেই। token আসে মোবাইল app-এর সঙ্গে (অধ্যায় ৯)। তাই এই অংশ একটা আগাম নোট, সিদ্ধান্ত নয়।

---

## ৭. Threat model: CSRF-এ আক্রমণকারীর হাতে কী থাকে

CSRF-এ কোনো credential চুরি হয় না, তাই অধ্যায় ৪-এর প্রশ্নগুলো একটু মানিয়ে নিতে হয়।

| প্রশ্ন | উত্তর |
| ------ | ----- |
| কী control করে? | একটা পাতা, যেটা রাফি খোলে। সেই পাতার JavaScript দিয়ে রাফির browser-কে request পাঠানোর নির্দেশ। request-এর URL, method আর body (browser-এর নিয়মের সীমার মধ্যে)। সহোদর subdomain-ও হতে পারে |
| কী steal করে? | কিছুই না। cookie, response বা token পড়তে পারে না। এই আক্রমণ চুরি নয়, **ব্যবহার করানো**। ব্যতিক্রম: token যদি URL, log, `Referer` বা XSS দিয়ে ফাঁস হয় |
| কী observe করে? | response দেখে না (SOP)। অন্ধ হামলা। সফল হলো কি না তার নিজের জানার উপায় সীমিত |
| কী modify করে? | parameter যা খুশি। পারে না: `Origin`, `Sec-Fetch-*` বদলাতে। OWASP বলে `Origin`/`Referer` forbidden header, শুধু browser বসাতে পারে। cross-origin-এ custom header পাঠাতেও CORS-এর অনুমতি লাগে |
| কী replay করে? | একই পাতা যে কোনো logged-in victim-এর বিরুদ্ধে চলে। victim-নির্দিষ্ট গোপন কিছু লাগে না। token থাকলে লাগে, আর সেটা সে জানে না |
| credential leak হলে? | CSRF-এ credential leak হয় না। ক্ষতি = victim-এর ক্ষমতা। OWASP: আক্রমণকারী শুধু application-এর দেওয়া সামর্থ্য আর victim-এর অধিকারই কাজে লাগাতে পারে |
| কতক্ষণ access? | একটা request। কিন্তু victim-এর session যতক্ষণ বৈধ আর সে ওই পাতা খোলে, ততক্ষণ প্রতিবার আবার। অধ্যায় ৫-এর নিয়মে সেটা idle ৩০ মিনিট (কাজের মাঝে ঘড়ি নতুন হয়) আর absolute ৮ ঘণ্টা। রাফি ৩০ মিনিট চুপ থাকলে session মরে, হামলা ব্যর্থ। session hijack-এর মতো টানা দখল নয় |
| server কি ধরতে পারবে? | সাধারণ log-এ প্রায় না (শুরুর log দেখো)। অধ্যায় ৫-এর context signal (network, User-Agent) চুপ থাকে, কারণ request victim-এর নিজের browser থেকেই আসে। Fetch Metadata আর token থাকলে হ্যাঁ: cross-site `POST` আর token mismatch গোনা যায়। OWASP বলে এমন ঘটনা log করতে, কিন্তু token-এর মান log-এ নয় |
| server কি revoke করতে পারবে? | CSRF নিজে credential চুরি করে না, তাই revocation মূল প্রতিরক্ষা নয়। Credential compromise-এর আলাদা প্রমাণ থাকলে session revoke করো; নইলে audit, ক্ষতি ফেরানো ও business recovery জরুরি |
| ক্ষতি কীভাবে সীমিত হয়? | স্তর (§৪), ছোট ভূমিকা (অধ্যায় ২, ১৬), ফেরানো-যায় এমন কাজ (soft cancel), সংবেদনশীল কাজে আবার প্রমাণ (অধ্যায় ২৩) |

---

## ৮. Trade-off: প্রতিটা দেয়ালের দাম

- **token:** প্রতিটা form-এ বসাতে হয়, server-কে session-এর সঙ্গে রাখতে হয়। per-request token বেশি নিরাপদ, কিন্তু "Back" ভাঙে।
- **Fetch Metadata:** header না থাকলে কী করবে, সেটা একটা নীতি-সিদ্ধান্ত (fail-safe বনাম fail-open)। proxy header মুছতে পারে। prefetch বা prerender-এর মতো speculative লোডিং-এ header নিয়ে অনিশ্চয়তা আছে (OWASP-র "Concerns")। webhook বা partner integration-কে আলাদা করে ছাড় দিতে হয়।
- **same-site প্রত্যাখ্যান:** সহোদর subdomain-কে বিশ্বাস না করলে নিরাপদ, কিন্তু পরে সত্যিকারের সহোদর সেবা এলে ছাড় দিতে হবে।
- **login-এর pre-session token:** পুরোনো login পাতা (৩০ মিনিটের বেশি) submit করলে `403`। user-এর টাইপ করা তথ্য যায়, আর ভালো নকশায় পাতা নতুন token সহ ফিরে আসা উচিত।
- **দুই ক্লিক:** নীরার এক ক্লিকের স্বপ্ন গেল। ইমেইলের লিঙ্ক এখন পাতা খোলে।
- **`Lax` বনাম `Strict`:** `Strict` শক্ত, কিন্তু লিঙ্ক ধরে আসা কর্মীরা login-এ ঠেকে। দুই cookie-র কৌশলে দুটোই মেলে, দামে জটিলতা।
- **নিজের কোড বনাম standard library:** নিজের কোড আমরা বুঝি, কিন্তু রক্ষণাবেক্ষণ আমাদের। standard library-র কোড অন্যদের পরীক্ষিত, কিন্তু তার ডিফল্ট আমাদের নীতির সঙ্গে নাও মিলতে পারে (§৪.২)।
- **log:** blocked request log করলে হামলার চেষ্টা দেখা যায়, কিন্তু log-এ token বা session ID লিখে ফেলা চলবে না (অধ্যায় ৪)।

---

## ৯. এই সমাধান কী সমাধান করে না

- **XSS।** OWASP-র সবচেয়ে বড় সতর্কতা: XSS সব CSRF প্রতিরক্ষা ভেঙে দিতে পারে। আক্রমণকারীর script আমাদের নিজের পাতায় চললে request আসবে `same-origin` থেকে, `Origin` ঠিক থাকবে, পাতা থেকে token-ও পড়া যাবে। প্রতিটা স্তর পাস করবে। অধ্যায় ৮।
- **`GET`-এ state বদল।** guard safe method-কে CSRF check থেকে ছাড় দেয়, আর synchronizer token-ও safe request-এ সাধারণত যাচাই হয় না। `GET /…/cancel` থাকলে স্তরগুলো পাশ কাটানো যেতে পারে। তাই safe method-এ state change না হওয়াটাই মৌলিক invariant.
- **client-side CSRF।** OWASP-র আলাদা একটা শ্রেণি: আমাদের নিজের JavaScript যদি URL-এর অংশ ধরে request বানায়, আক্রমণকারী সেই ইনপুট নিয়ন্ত্রণ করে নিজের পছন্দের request করিয়ে নিতে পারে। request তখন same-origin, token সহ, cookie সহ। `SameSite` আর token কিছুই ধরে না। এখন আমাদের JavaScript কম, কিন্তু SPA এলে (অধ্যায় ১১) এটা জীবন্ত হবে।
- **session চুরি (অধ্যায় ৫)।** আক্রমণকারীর হাতে cookie থাকলে সে পাতা চেয়ে নিয়ে token-ও পায়। এই স্তরগুলো **browser-কে অপব্যবহার** থেকে বাঁচায়, credential-ধারী আক্রমণকারী থেকে নয়। `curl`-এ `Sec-Fetch-Site` লিখে দেওয়া যায়।
- **অধ্যায় ৫-এর যন্ত্র।** মেয়াদ, rotate, revoke আর context signal, কোনোটাই CSRF ধরে না, কারণ কিছু চুরি বা বসানো হয়নি, request আসে victim-এর নিজের browser থেকে। শুধু idle timeout জানালা ছোট করে: রাফি ৩০ মিনিট চুপ থাকলে session মরে, তখন আসা জাল request ব্যর্থ। কিন্তু রাফি কাজের মাঝে থাকলে নয়।
- **authorization (অধ্যায় ২, ১৬)।** CSRF victim-এর ক্ষমতা ধার করে। অধ্যায় ৫-এর শুভ (`user-4`) Owner নয়, তাই তার browser দিয়ে পাঠানো refund `403` পেত। কিন্তু রাফি (`user-1`) Owner। ক্ষমতা যার যত বেশি, তাকে ধরে CSRF-এর ক্ষতিও তত বেশি। কারও ক্ষমতা কতটুকু, সেটা CSRF-এর প্রতিরক্ষা ঠিক করে না, ঠিক করে অধ্যায় ২ আর ১৬-এর নকশা।
- **clickjacking।** একটা আলাদা সমস্যা: আসল user আসল পাতায় আসল ক্লিক করে, কিন্তু পাতাটা অদৃশ্যভাবে অন্য ছবির নিচে। তখন request সত্যিই same-origin, token সহ। এর প্রতিরক্ষা আলাদা (OWASP-র Clickjacking Defense cheat sheet)। এখানে আলোচনা হয়নি।
- **প্রতারণা।** রাফিকে কেউ বুঝিয়ে নিজেই বাটন টেপাতে পারে। সেটা ঠিক এই ধরনের জাল-request নয়।

---

## ১০. সাধারণ ভুল ধারণা

**"`SameSite=Lax` দিয়েছি, তাই CSRF শেষ।"** `Lax` top-level `GET`-এ cookie পাঠায়, সহোদর subdomain ঠেকায় না, আর কিছু client-এ কাজই করে না। §৩-এর সারণি দেখো।

**"শুধু `POST` ব্যবহার করলেই চলবে।"** form-এর `POST` cross-origin পাঠানো যায়। অধ্যায়ের প্রথম পরীক্ষায় `POST` আটকেছে `SameSite`, `POST` নিজে নয়।

**"JSON API হলে CSRF হয় না।"** API যদি simple content type গ্রহণ করে বা browser-স্বয়ংক্রিয় credential ব্যবহার করে, ঝুঁকি থাকতে পারে। Custom header সাধারণত preflight বাধ্য করে, কিন্তু CORS exception ভুল হলে সুরক্ষা ভাঙতে পারে। API-এর authentication mode, accepted content types ও CORS policy একসঙ্গে যাচাই করতে হবে.

**"CORS বসিয়েছি, তাই নিরাপদ।"** CORS অন্য origin-কে response *পড়তে* দেয় কি না ঠিক করে। হামলা response ছাড়াই চলে। উল্টো, ভুল CORS (credentials সহ উদার `Allow-Origin`) দরজা খুলে দেয়।

**"`HttpOnly` cookie CSRF ঠেকায়।"** না। `HttpOnly` cookie পড়া আটকায়, পাঠানো নয়। অধ্যায় ৪-এ MDN-এর সূত্রে দেখেছি, script-এর শুরু করা request-এও তা যায়।

**"HTTPS থাকলে CSRF হয় না।"** হামলা victim-এর নিজের বৈধ HTTPS সংযোগেই চলে। TLS পথ রক্ষা করে, কে request চাইল তা নয়।

**"`Referer` দেখলেই হলো।"** §৩। পুরোপুরি ভরসার নয়।

**"একটা নিশ্চিতকরণ পাতা থাকলেই হলো।"** ধাপ বাড়ালে আক্রমণকারীও ধাপ পাঠায়। কাজ করে ধাপের ভেতরের token।

**"token-এর মেয়াদ বা timestamp দরকার।"** OWASP: এটা একটা প্রচলিত ভুল ধারণা। CSRF token access token নয়। session নতুন হলে token নতুন।

**"সাধারণ double-submit cookie যথেষ্ট।"** OWASP এটাকে নিরুৎসাহিত করে। cookie বসাতে পারা আক্রমণকারী (সহোদর subdomain, অনিরাপদ পথ) মিলিয়ে দিতে পারে।

**"session-এর মেয়াদ কম রাখলে CSRF হবে না।"** idle timeout জানালা ছোট করে, বন্ধ করে না। কাজের মাঝে থাকা user-এর session তখনো বৈধ।

**"CAPTCHA CSRF আটকায়।"** OWASP স্পষ্ট বলে না। CAPTCHA বট আটকাতে বানানো।

**"token আছে, তাই XSS-এও নিরাপদ।"** না। §৯।

**"login পাতায় CSRF লাগে না।"** §৫।

---

## ১১. এক নজরে: CSRF আর তার প্রতিরক্ষা

| প্রশ্ন | উত্তর |
| ------ | ----- |
| এটা কী? | victim-এর browser দিয়ে বিশ্বস্ত সাইটে জাল request করানো, যা সাইট আসল থেকে আলাদা করতে পারে না |
| কেন সম্ভব? | ambient authority: browser কে শুরু করল না দেখেই cookie জুড়ে দেয় |
| কী solve করে (token আর guard)? | অন্য origin বা site থেকে আসা জাল request, যেখানে আক্রমণকারী token জানে না |
| কী solve করে না? | XSS, `GET`-এ state বদল, client-side CSRF, credential-ধারী আক্রমণকারী, authorization, clickjacking (§৯) |
| কোথায় থাকে? | guard: server-এর middleware। token: session-এর রেকর্ডে আর পাতার HTML-এ। `Sec-Fetch-Site`, `Origin`: browser বসায় |
| কে ব্যবহার করে? | browser header বসায়, পাতা token ফেরত পাঠায়, server যাচাই করে |
| কীভাবে abuse হয়? | `GET`-এ state, XSS দিয়ে token পড়া, URL বা log-এ token ফাঁস, সহোদর subdomain, header-এর fail-open নীতি |
| চুরি হলে ক্ষতি? | token ফাঁস হলে ওই session-এ জাল request সম্ভব। credential ফাঁস নয় |
| মেয়াদ কত, কেন? | token-এর = session-এর: idle ৩০ মিনিট, absolute ৮ ঘণ্টা (অধ্যায় ৫)। login-এর আগের anonymous session-এর token ৩০ মিনিট। আলাদা মেয়াদ নেই |
| বাতিল করা যায়? | token session-এর সঙ্গেই মরে। login-এ `Rotate` নতুন session আর নতুন token দেয়, পুরোনোটা (anonymous সহ) মৃত। আলাদা revoke-এর দরকার নেই |
| বিকল্প? | signed double-submit (stateless), custom header (JS client), `Strict` cookie, framework-এর বিল্ট-ইন |
| কখন লাগে? | cookie-ভিত্তিক authentication, বিশেষ করে form আর state বদলানো কাজ |
| কখন লাগে না? | ambient নয় এমন credential (header-এ হাতে-বসানো), তবে §৬-এর তিন প্রশ্ন সামলাতে হয় |
| Production trade-off? | §৮ |

---

## আমরা কী শিখলাম? (What did we learn?)

- CSRF credential চুরি নয়। এটা browser-কে দিয়ে ইচ্ছার বাইরে কাজ করানো। authentication বলে *কে*, CSRF-এ প্রশ্ন *ইচ্ছা*।
- কারণ ambient authority: cookie বাঁধা থাকে গন্তব্যের সঙ্গে, উদ্যোক্তার সঙ্গে নয়।
- এই দৃশ্যে হামলা সম্ভব হয় কারণ request state বদলায়, browser ambient cookie পাঠায়, আর server-এর কাছে attacker-এর অজানা secret proof লাগে না। Safe method-এ state বদলানো বন্ধ করা, origin signal যাচাই এবং session-নির্দিষ্ট token চাওয়া এই পথ ভাঙে.
- `SameSite=Lax` একটা ভালো প্রথম দেয়াল, কিন্তু শেষ দেয়াল নয়: top-level `GET`, সহোদর subdomain, পুরোনো client।
- নিরাপদ method-এ state বদলানো হলে পরের সব স্তর অন্ধ। logout তাই `POST`।
- request-এর উৎস দেখো browser-এর নিজে-বসানো header দিয়ে (`Sec-Fetch-Site`, `Origin`), আর target origin কনফিগে লেখো।
- token session-এর রেকর্ডে, স্বাধীন random, cookie-তে নয়, URL-এ নয়। আক্রমণকারী পাতা পড়তে পারে না, তাই token জানে না।
- এই সব স্তর **browser-এর অপব্যবহার** ঠেকায়। credential-ধারী আক্রমণকারী বা আমাদের নিজের পাতায় চলা script ঠেকায় না।

## পাঁচ মিনিটের পরীক্ষা

নিজের project-এর **staging**-এ (production-এ নয়):

**এক।** সব `GET` route-এর তালিকা করো। কোনটা ডেটা বদলায়? (বাতিল, মোছা, নিশ্চিত করা, logout।)

**দুই।** একটা `POST` ডাকো cross-site header সহ। এটা তোমার *server-এর নিয়ম* পরীক্ষা করে, browser-এর আচরণ নয়:

| পরীক্ষা | কী যাচাই করবে | প্রত্যাশিত ফল |
| --- | --- | --- |
| সব `GET` route পর্যালোচনা | কোনো state change, logout, cancel বা delete আছে কি না | সব side effect unsafe method-এ সরানো |
| `Sec-Fetch-Site: cross-site` | origin policy | state-changing request reject |
| `Sec-Fetch-Site` নেই, `Origin` untrusted | fallback policy | reject |
| `Sec-Fetch-Site` ও `Origin` দুটোই নেই | fail-safe policy | reject; compatibility trade-off নথিভুক্ত |
| same-origin request, token নেই/ভুল | synchronizer token | reject |
| login-এ token নেই বা পুরোনো pre-session | login CSRF ও lifecycle | authenticated session তৈরি নয় |
| login সফল, পুরোনো anonymous token পুনর্ব্যবহার | session rotation | পুরোনো session/token অকার্যকর |
| বাস্তব browser DevTools-এ form submit | browser-generated cookie ও Fetch Metadata | observed behavior নথিভুক্ত; কোনো নির্দিষ্ট browser matrix এই review-তে পরীক্ষা করা হয়নি |

এগুলো staging-এ manual verification-এর checklist, এখানে চালানো test-এর দাবি নয়। Browser-এ বাস্তবে পরীক্ষা করার সময় browser version, cookie attribute, proxy/header behavior ও environment লিখে রাখো.

`403` না এলে guard নেই।

**তিন।** `Sec-Fetch-Site` বাদ দিয়ে `Origin: https://evil.example` দাও। তারপর দুটোই বাদ দাও। দ্বিতীয়টায় কী হয়, সেটাই তোমার fail-safe বনাম fail-open নীতি।

**চার।** `Sec-Fetch-Site: same-origin` দাও, কিন্তু token ছাড়া। `403` আসা উচিত।

**পাঁচ।** login পাতার form-এ hidden token আছে কি না দেখো। token ছাড়া `POST /login` (বৈধ `Origin` সহ) কী পায়? login-এর পর পুরোনো anonymous cookie আর তার token দিয়ে কিছু করা যায় কি না, দেখো।

**ছয়।** একটা আসল browser-এ DevTools খুলে নিজের একটা `POST` দেখো: `Sec-Fetch-Site` আর `Origin` কী পাঠাচ্ছে? সব form-এ hidden token আছে কি না, page source-এ দেখো।

## একটা অস্বস্তিকর প্রশ্ন (One uncomfortable question)

আজ আমরা তিন স্তরের দেয়াল বানালাম। প্রতিটা দেয়ালের পেছনে একটা অদৃশ্য ধারণা আছে: **পাতায় যে script চলছে, সেটা আমাদের নিজের।**

ধরো কোনো এক input ঘরে কেউ একটা `<script>` লিখে দিল, আর আমাদের পাতা সেটাকে নিজের কোড বলে চালিয়ে দিল। তখন request আসবে `same-origin` থেকে। `Origin` ঠিক। পাতা থেকে token পড়াও যাবে। তাহলে কোন দেয়ালটা তখন দাঁড়িয়ে থাকবে?

আর `HttpOnly`-এর কী হবে? script cookie পড়তে পারবে না। কিন্তু request তো করতে পারবে, আর browser সেই cookie জুড়ে দেবে।

## পরের অধ্যায়

[অধ্যায় ৭: CORS: Browser কোন Response পড়তে পারবে?](../07-cors/README.md)। Browser অন্য origin-এর response কখন JavaScript-কে পড়তে দেয়, আর CORS কেন authentication, authorization বা CSRF defense নয়, সেটা দেখব।

---

## তথ্যসূত্র

এই অধ্যায়ের Technical Review ও Security Review হলো source-based conceptual review। কোনো application code compile/test, automated security test, বাস্তব browser compatibility matrix বা production configuration পরীক্ষা করা হয়নি। তাই এই review production security certification নয়.

- [OWASP: Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html): synchronizer token, signed double-submit, Fetch Metadata, Origin/Referer validation, custom header, login CSRF, client-side CSRF, SameSite-এর সীমা ও rollout trade-off।
- [MDN: Cross-site request forgery (CSRF)](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF): simple request, CORS, SameSite=Lax/Strict এবং method override-এর সীমা।
- [W3C: Fetch Metadata Request Headers](https://www.w3.org/TR/fetch-metadata/): `Sec-Fetch-Site`-এর মান, `none`-এর অর্থ, এবং `Sec-` prefix কেন JavaScript-এর মাধ্যমে পরিবর্তনযোগ্য নয়। এটি Working Draft, চূড়ান্ত Recommendation নয়।
- [WHATWG: Fetch Standard, forbidden request-header names](https://fetch.spec.whatwg.org/): `Sec-` prefix browser API থেকে set/modify করা নিষিদ্ধ করে; non-browser client-এর ওপর এই নিশ্চয়তা প্রযোজ্য নয়।
- [RFC 7617 §2.2: Reusing Credentials](https://www.rfc-editor.org/rfc/rfc7617.html#section-2.2): HTTP Basic client-কে একই protection space-এ credential preemptively পাঠানোর অনুমতি দেয়; নির্দিষ্ট browser-এর আচরণ এই review-তে পরীক্ষা করা হয়নি।
- [RFC 9110 §9.2.1: Safe Methods](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.1): safe method-এর অর্থ এবং কেন GET-এ state change করা যাবে না।
- [Go `net/http` `CrossOriginProtection`](https://pkg.go.dev/net/http@go1.25#CrossOriginProtection): optional implementation reference, chapter-এর implementation নির্দেশনা নয়।
- [Go vulnerability report GO-2025-3955](https://pkg.go.dev/vuln/GO-2025-3955): Go 1.25.0-এর `AddInsecureBypassPattern`-এর নির্দিষ্ট bypass issue; Go 1.25.1-এ সংশোধিত।
- [OWASP: Clickjacking Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html): CSRF-এর পাশের কিন্তু আলাদা framing/UI-redress threat.

