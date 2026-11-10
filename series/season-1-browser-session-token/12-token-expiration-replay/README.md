---
chapter: 12
season: 1
title: "Token Expiration এবং Replay Window"
status: draft
---

# Token Expiration এবং Replay Window

একটা মঙ্গলবার সকাল। KenaKata-র দোকানমালিক রাফি dashboard-এ order দেখছে। Mobile app-এর জন্য আরিফ আগের অধ্যায়ে access token ব্যবহার শুরু করেছে। আজ support team জানাল, password বদলানোর পরও পুরোনো একটি request দিয়ে কিছুক্ষণ order দেখা গেছে। আরেকটি report-এ log-এ থাকা bearer token পরে অন্য client থেকেও ব্যবহার হয়েছে।

এই incident একটি policy-র প্রশ্ন তোলে: password বদলালে আগে issue করা access token-ও কি invalidate হয়? এর উত্তর architecture ও revocation policy-র ওপর নির্ভর করে। Password বদলালেই সব system-এ already-issued access token স্বয়ংক্রিয়ভাবে বাতিল হয় না।

প্রশ্নটি হলো: **একটি token কতক্ষণ বিশ্বাসযোগ্য, আর সেই সময়ের মধ্যে token কপি হয়ে গেলে কী হতে পারে?**

Token-এর signature যাচাই সফল হওয়া মানে trusted key ও গ্রহণযোগ্য algorithm-এর প্রেক্ষিতে integrity check পাস করেছে। কিন্তু সঠিক key নির্বাচন, issuer ও audience validation এবং application policy ছাড়া শুধু signature দেখে token-কে trusted বলা যায় না। Signature ঠিক থাকলেও token এখনো ব্যবহারযোগ্য কি না, revoke করা হয়েছে কি না, কিংবা request-টি আসল ব্যবহারকারীই পাঠিয়েছেন কি না, তা আলাদা প্রশ্ন। তাই token-এর lifetime এবং replay-কে আলাদা করে বুঝতে হবে।

এই অধ্যায়ে KenaKata-র বর্তমান first-party mobile/API access-token model নিয়ে আলোচনা করছি। Refresh-token-এর ধারণা ও mobile client-এর lifecycle অধ্যায় ১৩–১৫-তে আসবে। OAuth authorization flow এবং তার protocol-specific নিয়ম অধ্যায় ১৭–২১-তে শেখানো হবে; সেগুলোকে KenaKata-র বর্তমান first-party login-এর implementation হিসেবে ধরে নিও না।

## এই অধ্যায়ে কী বুঝবে

- Token expiration ঝুঁকির সময়সীমা কমায়, কিন্তু replay-কে নিজে থেকে অসম্ভব করে না।
- Access-token lifetime, refresh-token lifetime এবং session lifetime এক ঘড়ি নয়।
- Revocation, audience/scope restriction ও sender-constraining কখন অতিরিক্ত কাজে লাগে.

## ১. Expiration কোন সমস্যার সমাধান করে?

Access token হলো API-তে access চাওয়ার credential। Bearer token-এর ক্ষেত্রে যার কাছে token আছে, সে token উপস্থাপন করতে পারে। Server token-টি বৈধ বলে গ্রহণ করলে request-টি সংশ্লিষ্ট access পেতে পারে, তবে endpoint-এর authorization check-ও পাস করতে হবে।

Token log-এ চলে গেলে, compromised device থেকে কপি হলে, বা অন্যভাবে attacker-এর হাতে গেলে, token বৈধ থাকা পর্যন্ত attacker সেটি replay করার চেষ্টা করতে পারে।

**Expiration সেই ঝুঁকির সময়সীমা কমায়।** মেয়াদ শেষ হলে server-কে token আর গ্রহণ করা উচিত নয়। কিন্তু expiration কোনো token-কে চুরি-প্রতিরোধী করে না। মেয়াদ শেষ হওয়ার আগের সময়টুকুতে token ব্যবহারযোগ্য থাকতে পারে।

ধরা যাক, token সকাল ১০টায় issue হলো এবং ১০টা ১৫ মিনিটে expire হবে। সকাল ১০টা ০৫ মিনিটে attacker সেটি পেলে, অন্য কোনো control-এ আটকে না গেলে তার হাতে প্রায় ১০ মিনিটের replay opportunity থাকতে পারে।

এখানে **replay window** বলতে বুঝব: কোনো credential attacker-এর হাতে যাওয়ার পর সেটি গ্রহণযোগ্য থাকা পর্যন্ত যে সময় এবং পরিস্থিতির সুযোগ attacker পায়। এটি শুধু token lifetime-এর আরেক নাম নয়। চুরির সময়, revoke করার ব্যবস্থা, server validation এবং token কোন resource-এ গ্রহণ করা হয়, সবই ঝুঁকির পরিধি বদলায়। সাধারণ bearer token-এর ক্ষেত্রে token যার কাছে আছে, সে সেটি উপস্থাপন করতে পারে; token-টি মূল user-এর কাছেই আছে কি না, signature একা তা প্রমাণ করে না।

## ২. Lifetime-এর তিনটি আলাদা ঘড়ি

- **Token lifetime:** token কখন থেকে কখন পর্যন্ত গ্রহণযোগ্য হওয়ার কথা।
- **Session lifetime:** ব্যবহারকারীর login/session কতক্ষণ সক্রিয় থাকবে।
- **Replay opportunity:** চুরি করা credential attacker কতক্ষণ এবং কোন পরিস্থিতিতে ব্যবহার করতে পারবে।

এগুলো সম্পর্কিত, কিন্তু এক নয়।

| বিষয় | কী নিয়ন্ত্রণ করে | কী নিশ্চিত করে না |
| --- | --- | --- |
| Access token expiration | Token-এর স্বাভাবিক গ্রহণযোগ্যতার শেষ সময় | মেয়াদ শেষ হওয়ার আগে replay হবে না |
| Server-side revocation / denylist | নির্দিষ্ট token-কে মেয়াদের আগেও reject করার সুযোগ | সব resource server সঙ্গে সঙ্গে নতুন state জানবে |
| Session timeout | Login/session-এর idle বা সর্বোচ্চ সময় | আলাদা access token সঙ্গে সঙ্গে বাতিল হবে |
| Short-lived token | চুরি হওয়া token-এর সম্ভাব্য ব্যবহার-সময় কমায় | Token চুরি বা active XSS পুরোপুরি আটকায় |

একটি গুরুত্বপূর্ণ প্রশ্ন: ব্যবহারকারী logout করলে বা account compromise ধরা পড়লে, ইতিমধ্যে issue করা access token কি সঙ্গে সঙ্গে অকার্যকর হওয়া দরকার? যদি হ্যাঁ হয়, শুধু JWT-র exp claim-এর ওপর নির্ভর করলে চলবে না। Stateless validation-এ server signature ও claims দেখে সিদ্ধান্ত নিতে পারে, কিন্তু প্রতিটি request-এ server-side revocation state না দেখলে logout event নিজে থেকে token-টিকে বাতিল করে না।

প্রতিটি request-এ central state check করলে revocation দ্রুত হতে পারে, কিন্তু latency, availability এবং consistency-এর trade-off আসে। কোন পদ্ধতি ঠিক, তা threat model ও product requirement-এর ওপর নির্ভর করে।

## Lifetime ও replay window এক নজরে

নিচের চিত্রে token-এর মেয়াদ এবং token চুরি হয়ে গেলে replay-এর সুযোগ আলাদা করে দেখানো হয়েছে। Expiry একটি নির্দিষ্ট সময়ের পর token গ্রহণ বন্ধ করতে পারে; মেয়াদ শেষ হওয়ার আগের replay ঠেকাতে validation, revocation বা sender-constraining-এর মতো অতিরিক্ত control লাগতে পারে।

```mermaid
flowchart TD
    A[Access token issue] --> B[Token active এবং policy অনুযায়ী গ্রহণযোগ্য]
    B --> C{Attacker token পেয়ে গেছে?}
    C -->|না| D[স্বাভাবিক request চলতে পারে]
    C -->|হ্যাঁ| E[Expiry-এর আগে replay চেষ্টা হতে পারে]
    E --> F{অন্য control replay ঠেকায়?}
    F -->|হ্যাঁ| G[Request reject]
    F -->|না| H[Token-এর scope অনুযায়ী অপব্যবহার হতে পারে]
    B --> I[Expiry time পেরোয়]
    I --> J[সঠিক validation হলে token reject]
```

**চিত্রের মূল কথা:** Expiry হলো সময়ের সীমা; replay protection হলো কে token ব্যবহার করছে বা token ইতিমধ্যে ব্যবহার হয়েছে কি না, সেই ঝুঁকি সামলানোর আলাদা প্রশ্ন।
## ৩. JWT-এর exp কী বলে, কী বলে না?

JWT-তে exp হলো expiration time claim। RFC 7519 অনুযায়ী, claim উপস্থিত থাকলে বর্তমান সময় exp-এর সমান বা তার পরে হলে token গ্রহণ করা যাবে না। সময়টি NumericDate হিসেবে প্রকাশিত হয়।

একটি illustrative payload:

~~~json
{
  "iss": "https://auth.kenakata.example",
  "sub": "user-1842",
  "aud": "kenakata-orders-api",
  "iat": 1792915200,
  "exp": 1792916100
}
~~~

এটি শুধু claim-এর আকৃতি বোঝানোর উদাহরণ, বাস্তব token নয়।

- iss বলে token কে issue করেছে।
- sub subject-কে চিহ্নিত করে।
- aud token কোন intended recipient-এর জন্য, তা জানায়।
- iat token কখন issue হয়েছে, তা জানায়।
- exp token-এর expiration time জানায়।

সময়-সংক্রান্ত claim payload-এ আছে বলেই বিশ্বাস করা যাবে না। JWT হলে signature বা অন্য উপযুক্ত integrity protection trusted key ও গ্রহণযোগ্য algorithm দিয়ে যাচাই করতে হবে, প্রত্যাশিত issuer ও audience মিলতে হবে, এবং application/profile অনুযায়ী প্রয়োজনীয় claim যাচাই করতে হবে। Opaque token হলে সংশ্লিষ্ট server-side validation বা introspection policy প্রযোজ্য হবে। exp claim RFC 7519-এ optional; তাই exp না থাকলে কী হবে, সেটি application/profile-এর policy-তে স্পষ্ট থাকা চাই। কোনো application যদি access token-এ expiration বাধ্যতামূলক করে, তবে exp অনুপস্থিত JWT reject করা সেই policy-র অংশ।

**Clock skew:** Client, issuer এবং resource server-এর ঘড়ি সামান্য আলাদা হতে পারে। কিছু validation system সীমিত clock-skew allowance দেয়। এই allowance যেন বড় না হয়, কারণ অতিরিক্ত tolerance কার্যকর replay window বাড়াতে পারে। সময়ের অসামঞ্জস্য সামলানো এবং token-এর মেয়াদ ইচ্ছেমতো বাড়িয়ে দেওয়া এক জিনিস নয়।

## ৪. Short-lived token: কত ছোট হলে ভালো?

“Access token-এর মেয়াদ ৫ মিনিট রাখলেই নিরাপদ” এমন universal rule নেই। ছোট lifetime চুরি হওয়া token-এর ব্যবহার-সময় কমাতে পারে, কিন্তু token renewal-এর প্রয়োজন বাড়ায়, network dependency বাড়াতে পারে, এবং refresh mechanism ভুলভাবে বানালে নতুন ঝুঁকি তৈরি হয়।

Lifetime বেছে নেওয়ার সময় বিবেচনা করো:

1. Token দিয়ে কী ধরনের কাজ করা যায়? Order পড়া, নাকি refund বা payment-এর মতো sensitive action-ও?
2. Token কোথায় থাকে এবং কীভাবে চুরি হতে পারে? Mobile device, browser storage, debug log, crash report, নাকি proxy?
3. Token revoke করার প্রয়োজন হলে ব্যবস্থা কত দ্রুত কার্যকর হতে পারে?
4. API প্রতিটি request-এ token-এর বর্তমান status যাচাই করে, নাকি মূলত signature ও claims-এর ওপর নির্ভর করে?
5. User experience এবং availability-এর কী সীমা আছে?

অধিক privilege-যুক্ত token-এর সম্ভাব্য ক্ষতি বেশি। তাই token-এর privilege ন্যূনতম রাখা, audience সীমিত করা এবং lifetime সংক্ষিপ্ত রাখা একে অন্যের বিকল্প নয়; এগুলো layered control।

### 🧠 Decision Game: ১৫ মিনিট না ২ ঘণ্টা?

KenaKata-র mobile app একটি access token পায়। API-টি order পড়তে পারে এবং order cancel-ও করতে পারে। Team বলছে, token-এর মেয়াদ ২ ঘণ্টা থেকে ১৫ মিনিটে নামালেই replay problem মিটে যাবে।

তুমি কোন সিদ্ধান্ত নেবে?

- A. Lifetime ১৫ মিনিট করলেই হবে; অন্য control দরকার নেই।
- B. Lifetime কমানোকে risk-reduction control হিসেবে নেব, পাশাপাশি privilege, token exposure, revocation এবং sensitive action-এর authorization review করব।
- C. JWT signature থাকলে expiration নিয়ে ভাবার দরকার নেই।
- D. Token lifetime বাড়াব, যাতে client-কে কমবার authenticate করতে হয়।

<details>
<summary>উত্তর ও reasoning দেখো</summary>

**সেরা উত্তর: B।**

A ঝুঁকি কমানোর একটি উপায়কে পূর্ণ সমাধান ধরে নিচ্ছে। Token ১৫ মিনিট valid থাকলেও attacker সেই সময়ের মধ্যে replay করতে পারে। Token কী করতে পারে এবং কীভাবে revoke করা যায়, সেটিও গুরুত্বপূর্ণ।

C ভুল, কারণ signature integrity ও issuer-সম্পর্কিত validation-এ সাহায্য করে; এটি expiration, revocation অথবা presenter-এর পরিচয় নিজে থেকে প্রমাণ করে না।

D user experience-এর একটি দিক বিবেচনা করে, কিন্তু stolen-token exposure window বাড়াতে পারে। দীর্ঘ lifetime বেছে নেওয়া হলে তার ঝুঁকি ও compensating controls মূল্যায়ন করতে হবে।

**Edge case:** Token যদি শুধু read-only, অল্প privilege-এর endpoint-এ সীমাবদ্ধ থাকে, risk profile sensitive write operation-এর token থেকে আলাদা হতে পারে। তবুও replay-এর ঝুঁকি শূন্য হয় না।

</details>

## ৫. Replay কীভাবে ঘটে?

Replay মানে কোনো বৈধ credential বা request আবার ব্যবহার করা। Attacker-কে নতুন token তৈরি করতে হবে না; আগে থেকে বৈধ একটি token পেয়ে সেটি আবার উপস্থাপন করলেই হতে পারে।

~~~mermaid
sequenceDiagram
    participant App as KenaKata Mobile App
    participant API as Orders API
    participant Attacker as Attacker

    App->>API: Request + Bearer token
    API->>API: Signature, claims ও policy যাচাই
    API-->>App: অনুমোদিত response

    Note over App,Attacker: Token log বা compromised device থেকে ফাঁস হয়

    Attacker->>API: একই Bearer token দিয়ে request
    API->>API: Token এখনও valid কি?
    alt Token valid এবং অন্য control পাস
        API-->>Attacker: Request-এর অনুমোদিত ফল
    else Token expired / revoked / policy fail
        API-->>Attacker: Reject
    end
~~~

এটি সরলীকৃত ধারণাগত flow। API-কে শুধু token গ্রহণযোগ্য কি না নয়, requested resource-এর authorization-ও যাচাই করতে হবে। Token বৈধ হলেও user-এর access নেই এমন order পড়তে দেওয়া যাবে না।

Replay-এর উৎস হতে পারে:

- Application log-এ ভুল করে লেখা Authorization header।
- Compromised endpoint বা backup-এ থাকা token।
- Browser-এর malicious JavaScript বা compromised device।
- Token অন্য কোনো channel-এ কপি হয়ে যাওয়া।

HTTPS transit-এ token সুরক্ষিত রাখতে গুরুত্বপূর্ণ, কিন্তু endpoint, log, device বা application-level exposure-এর সব ঝুঁকি দূর করে না। Raw token log না করা, secret redaction, access control এবং incident response-এর ব্যবস্থা তাই প্রাসঙ্গিক।

## ৬. Expiration বনাম revocation

Expiration সাধারণত সময়ভিত্তিক: নির্দিষ্ট সময় পার হলে token আর গ্রহণযোগ্য নয়। Revocation হলো সময় শেষ হওয়ার আগেই token বা সংশ্লিষ্ট grant অকার্যকর করার ব্যবস্থা।

JWT signature-ভিত্তিক local validation-এর সুবিধা হলো প্রতিটি request-এ issuer-এর সঙ্গে যোগাযোগ না করেও token যাচাই করা যায়। কিন্তু server-এর কাছে revocation state না থাকলে ইতিমধ্যে issue করা token তার expiration পর্যন্ত গ্রহণযোগ্য থাকতে পারে। Exact behavior architecture ও policy-র ওপর নির্ভর করে।

সম্ভাব্য design option:

- **Short lifetime:** চুরি হওয়া token-এর স্বাভাবিক valid period কমায়।
- **Server-side denylist বা revocation state:** নির্দিষ্ট token-কে expiration-এর আগেও reject করতে পারে, যদি resource server সেই state দেখে।
- **Introspection বা central status check:** প্রতিটি token-এর active state central service থেকে যাচাই করা যায়; এতে dependency ও latency বাড়ে।
- **Account-level invalidation বা credential versioning:** একাধিক token invalidate করার policy বানানো যায়, তবে সব resource server-এ সেই state কার্যকর হতে হবে।
- **Sender-constrained token:** token-কে নির্দিষ্ট client-এর cryptographic key-এর সঙ্গে bind করার পদ্ধতি কিছু token-theft ও replay scenario কঠিন করে। OAuth context-এ **DPoP (Demonstrating Proof of Possession, RFC 9449)** ও **mTLS (mutual TLS, RFC 8705)** এর উদাহরণ। DPoP-ও সব replay আটকানোর নিশ্চয়তা নয়: proof নিজেই স্বল্প validity window-তে replay হতে পারে, তাই server-side proof validation এবং প্রয়োজনমতো `jti` (JWT ID claim) replay tracking গুরুত্বপূর্ণ। এগুলো সাধারণ bearer-token validation-এর সমার্থক নয়; সঠিক key handling ও protocol validation প্রয়োজন।

কোনো একটিকে সব ব্যবস্থার universal replacement ভাববে না। জরুরি invalidation-এর প্রয়োজন, architecture-এর availability এবং attacker-এর ক্ষমতা অনুযায়ী control বেছে নিতে হয়।

## ৭. Token expire হলে user-এর কী হবে?

Access token expire হওয়া মানেই user-এর account বন্ধ, password বদলানো, বা server-side session শেষ হয়ে যাওয়া নয়। এটি প্রথমত ঐ token-এর গ্রহণযোগ্যতার সময় শেষ হওয়া।

KenaKata-তে access token expire হলে API-র উচিত সেটি reject করা। Client product-এর authentication design অনুযায়ী user-কে আবার login করতে বলতে পারে, অথবা অনুমোদিত renewal flow থাকলে সেটি ব্যবহার করতে পারে। **Access token renew করার পদ্ধতি token-এর validation rule বদলে দেয় না।**

এখানে পরের অধ্যায়ের সঙ্গে সংযোগ আছে। Refresh token দীর্ঘমেয়াদি credential হতে পারে, তাই storage, rotation, reuse detection এবং revocation আলাদা security concern। Refresh token দিয়ে নতুন access token আনা যায় বলেই access token-এর lifetime অর্থহীন নয়; আবার access token short-lived বলেই refresh-token flow নিরাপদ হয়ে যায় না।

Logout-এর ক্ষেত্রেও স্পষ্ট policy দরকার: client-এর storage থেকে token মুছে ফেলা client-side cleanup, কিন্তু attacker আগে token কপি করে থাকলে তার কপি নিজে থেকে মুছে যায় না। Server-side revocation বা অল্প lifetime প্রয়োজন কি না, তা threat model অনুযায়ী স্থির করতে হবে।

## ৮. KenaKata-র জন্য সিদ্ধান্ত

এই পর্যায়ে KenaKata-র team নিচের security invariant-গুলো ধরে রাখবে:

1. **প্রতিটি protected API request-এ token type অনুযায়ী validation হবে।** JWT হলে trusted key, allowed algorithm, issuer, audience, প্রযোজ্য time claims ও প্রয়োজনীয় policy যাচাই হবে; opaque token হলে server-side validation/introspection policy অনুসরণ করা হবে। JWT decode করাই validation নয়।
2. **Expired token গ্রহণ করা হবে না।** Clock-skew allowance থাকলে তা সীমিত ও নথিভুক্ত হবে।
3. **Token-এর privilege ন্যূনতম হবে।** Token valid হলেও resource-level authorization আলাদা করে করতে হবে।
4. **Bearer token-কে secret credential হিসেবে বিবেচনা করা হবে।** Authorization header, raw token বা refresh credential log করা হবে না।
5. **Expiration ও revocation-এর পার্থক্য design-এ স্পষ্ট থাকবে।** জরুরি invalidation দরকার হলে শুধু exp-এর অপেক্ষা যথেষ্ট কি না, তা threat model দিয়ে বিচার করতে হবে।
6. **Short lifetime-কে একমাত্র প্রতিরক্ষা বলা হবে না।** Token exposure reduction, replay detection এবং authorization একসঙ্গে বিবেচনা করতে হবে।

### 🔍 Incident investigation: log-এ token পাওয়া গেল

Support team একটি API log-এ bearer token-এর অংশ পেয়েছে এবং একই token দিয়ে অস্বাভাবিক request হয়েছে। কোন তথ্যটি একা নিশ্চিত প্রমাণ নয়?

IP address, request timestamp, এবং token-এর sub claim, এগুলোর কোনোটিই একা নির্দিষ্ট মানুষটি request পাঠিয়েছে তা নিশ্চিত করে না।

IP address shared network, proxy, VPN বা mobile network-এর কারণে একাধিক মানুষের হতে পারে। Timestamp ঘটনার সময়ের signal, মানুষের পরিচয়ের প্রমাণ নয়। sub claim token-এর subject জানায়, কিন্তু token চুরি হলে সেটি উপস্থাপনকারী ব্যক্তি সেই subject-ই, এমন নয়। একাধিক log source, request context, token status এবং independent evidence মিলিয়ে দেখতে হবে। Log-এ raw credential থাকলে সেটিকে সম্ভাব্য secret exposure হিসেবেও বিবেচনা করো।

## ৯. নিজেকে যাচাই করো 🎯

1. JWT signature valid হলেও token reject করার অন্তত তিনটি কারণ কী?
2. Access token-এর lifetime ৫ মিনিট করা হলে কোন ঝুঁকি কমে, আর কোন ঝুঁকি থেকে যায়?
3. User logout করলে client-side token মুছে ফেলার বাইরে server-side invalidation দরকার কি না, কীভাবে সিদ্ধান্ত নেবে?
4. Expiration এবং revocation-এর মধ্যে পার্থক্য কী?
5. Token replay তদন্তে IP address ও timestamp কেন একা মানুষের পরিচয়ের প্রমাণ নয়?

<details>
<summary>নিজের উত্তর মিলিয়ে দেখো</summary>

1. Token expired, issuer/audience mismatch, signature বা algorithm validation failure, required claim অনুপস্থিত, revoked state, অথবা endpoint-এর authorization policy fail করতে পারে।
2. Token-এর স্বাভাবিক valid period ছোট হয়; কিন্তু valid window-তে replay, active XSS, token leakage এবং token দিয়ে করা unauthorized action-এর ঝুঁকি পুরোপুরি যায় না।
3. Incident response-এর প্রয়োজনীয় গতি, token lifetime, architecture, revocation propagation এবং token-এর privilege বিবেচনা করতে হবে।
4. Expiration নির্ধারিত সময়ের শেষে token reject করে; revocation মেয়াদ শেষ হওয়ার আগেও token-কে অকার্যকর করার ব্যবস্থা।
5. IP shared বা পরিবর্তনশীল হতে পারে; timestamp শুধু সময় দেখায়; token-এর subject claim-ও token presenter-কে প্রমাণ করে না।

</details>

## 💡 মূল শিক্ষা

**Expiration ঝুঁকির জানালা ছোট করে, কিন্তু replay অসম্ভব করে না।** নিরাপদ design-এর জন্য token lifetime-এর সঙ্গে validation, least privilege, exposure control এবং প্রয়োজন অনুযায়ী revocation বা sender-constraining একসঙ্গে ভাবতে হয়।

## References

- [RFC 7519 — JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519), বিশেষ করে exp, nbf, iat, jti এবং NumericDate claim।
- [RFC 8725 — JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725), algorithm, issuer, audience validation ও JWT security pitfalls।
- [RFC 6750 — OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750), OAuth bearer-token model-এর reference; এটি KenaKata-র first-party flow-কে OAuth flow হিসেবে চিহ্নিত করে না।
- [OWASP OAuth 2.0 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html), token replay protection, sender-constrained token এবং refresh-token rotation guidance। OAuth-নির্দিষ্ট অংশগুলোকে KenaKata-র বর্তমান first-party flow-এর specification ধরে নেওয়া যাবে না।
- [RFC 9449 — OAuth 2.0 Demonstrating Proof of Possession (DPoP)](https://www.rfc-editor.org/rfc/rfc9449), OAuth token-কে proof-of-possession key-এর সঙ্গে bind করা এবং DPoP proof replay-এর সীমা।
- [RFC 7009 — OAuth 2.0 Token Revocation](https://www.rfc-editor.org/rfc/rfc7009), OAuth token revocation endpoint-এর protocol; KenaKata-তে OAuth flow চালুর পর প্রাসঙ্গিক হবে।

Protocol বা security claim নিয়ে পরিবর্তন করার আগে authoritative source-এর বর্তমান লেখা খুলে যাচাই করো। এই অধ্যায় ধারণাগত শিক্ষা; এটি production token-validation implementation-এর specification নয়।

---

