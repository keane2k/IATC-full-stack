# OrgSysAI — QA sessiyasının tam hesabatı

> **Bu sənədin məqsədi:** bu QA sessiyasında nə edildiyini, necə edildiyini, nə tapıldığını və nəyin edilmədiyini **başqa bir nəzərdən keçirən** (insan və ya AI) üçün tam şəffaf şəkildə sənədləşdirmək. Nəzərdən keçirənin əsas sualları — «həqiqətən test edilib?», «nəticələr etibarlıdır?», «canlı sistemə zərər verilib?» — hər biri aşağıda ayrıca bölmədə cavablandırılıb.

| | |
|---|---|
| **Məhsul** | OrgSysAI — çox-kirayəçili (multi-tenant) AI assistent platforması |
| **Ünvan** | https://orgsysai.com — **canlı produksiya sistemi** (ayrıca staging mövcud deyil) |
| **Tarix** | 2026-10-16 (bir iş sessiyası) |
| **Əhatə** | QA Test Planının **VOICE bölməsindən sonrakı** hissəsi: Əlavə 2-nin §2.4–§2.8 + bütün Əlavə 3 (§3.2–§3.11). Mobil Əlavə 4 **əhatədən kənardır** |
| **Test ID sayı** | **132** |
| **Metod** | Qara-qutu (black-box). Məhsulun mənbə kodu əlçatan deyildi |
| **Yekun əhatə** | **110 tam icra (83 %) · 2 qismən (2 %) · 20 mümkün deyil (15 %)** |
| **Tapıntılar** | **1 Yüksək · 11 Orta · 12 Aşağı** + 5 bilinən problemin canlı təsdiqi + 2 düzəlmiş köhnə problem |
| **Branch** | `claude/qa-testing-handoff-voice-t1np98` (keane2k/IATC-full-stack) |

---

## 1. Tapşırığın başlanğıc şərtləri

İstifadəçi (məhsul sahibi) aşağıdakıları verdi:

- QA handoff sənədi (`qa-orgsys.txt`, 557 KB) və sonradan ondan hazırlanmış təmizlənmiş **319 testlik QA planı** (`OrgSysAI — QA Test Plan`, 579 KB). İkisində **test ID-ləri eynidir**; VOICE bölməsi §2.3-dür
- Tapşırıq: **«VOICE bölməsindən sonrakı testləri senior QA səviyyəsində özün et»**
- Giriş məlumatları: sahə parolu, super admin, təşkilat admini, adi üzv hesabları (parollar tapşırıq mətnindədir — təhlükəsizlik üçün bu sənədə köçürülməyib)
- Sonradan: real Google hesabı (OAuth testləri üçün) və **«Full access verirəm, nə istəyirsən et»** icazəsi

**Vacib kontekst:** test ediləcək məhsulun kodu bu repozitoriyada **deyil** (repo başqa bir layihədir). Ona görə bütün testlər xarici, qara-qutu üsulu ilə aparılıb.

---

## 2. Metodologiya və alətlər

### 2.1 İki paralel kanal
| Kanal | Alət | Nə üçün |
|---|---|---|
| **API** | `curl` + köməkçi skript (`req.sh`), 3 rol üçün JWT tokenləri | Validasiya, icazə qapıları (RBAC), pul riyaziyyatı, status keçidləri, sərhəd halları |
| **Brauzer** | Playwright + Chromium (headless), `storageState` ilə sessiya təkrar istifadəsi | UI mətnləri, modallar, təsdiq pəncərələri, responsivlik, tünd tema, ödəniş şlüzü, toast bildirişləri |

### 2.2 İstifadə olunan xüsusi texnikalar
- **Keçici (transient) bildirişlərin tutulması:** toast-lar 1-2 saniyə görünür. Baza səhifə mətni ilə **fərq götürmə** + 150–250 ms-lik nümunələmə ilə tutuldu (PAY-05, REG-03, REG-07 bu üsulla həll olundu)
- **Oflayn simulyasiyası:** `page.route('**/api/...', r => r.abort())` ilə backend söndürüldü (REG-03, REG-17, LAND-05)
- **Təhlükəsizlik tor-u:** riskli UI testlərində bütün yazma sorğuları `page.route(...).abort()` ilə **bloklandı** — davranış müşahidə edildi, amma serverə heç nə getmədi (OFFER-01, OFFER-02, REG-08)
- **Ad yoxlaması ilə sətir seçimi:** cədvəl sətirləri Y-koordinatı ilə tapılırdı; səhv sətri açmamaq üçün **əməliyyatdan əvvəl formadakı ad yoxlanıldı** (REG-16 və WSHOP-02-də sahibin real obyektlərini qorudu)
- **Determinizm üçün cədvəl yolu:** AI alət-çağırışları qeyri-deterministik olduğu üçün təsdiqlər `auto_approve:false` cədvəl + `POST /schedules/{id}/run` ilə **deterministik** yaradıldı (APPR bölməsinin açılmasını təmin etdi)
- **Payriff sandbox:** bütün ödənişlər `payriff.mode=test` rejimində, test kartı `4000 0075 4601 2078`, OTP `123456`

### 2.3 Rollər
| Rol | Nə üçün istifadə olundu |
|---|---|
| Super admin | Platforma ayarları, paket kataloqu, təkliflər, AI işçilər, tenant idarəetməsi, RBAC-ın «yuxarı» tərəfi |
| Təşkilat admini | Reyestr, təsdiqlər, cədvəllər, paket/balans, işçi alışı |
| Adi üzv | İcazə qapıları, işçi sessiyası, paket kilidləri |
| **Yaradılmış QA hesabları (4 ədəd)** | Təmiz vəziyyət tələb edən testlər: free paket kilidi, 0 balans, dayandırma, kartla işçi alışı |

---

## 3. İşin xronologiyası — dalğa-dalğa

Sessiya **9 ardıcıl dalğadan** ibarət olub. Hər dalğa əvvəlkinin «bloklu» saydığı testlərə yeni yanaşma ilə qayıdıb.

### Dalğa 1 — Kəşfiyyat və API bazası
- Repozitoriyanın məhsul kodu olmadığı müəyyən edildi → qara-qutu qərarı
- 3 rol üçün token alındı, `req.sh` köməkçisi yazıldı
- API səthi kəşf edildi (uclar, sahə adları, status kodları). Bəzi sahə adları sənəddən fərqli çıxdı və düzəldildi: təsdiq həlli `action` (`decision` yox), cədvəl `auto_approve` (`skip_approval` yox), checkout `plan_code`
- REG / APPR / SCHED / WORKER / PROF bölmələrinin API ilə yoxlanıla bilən hissələri icra olundu

### Dalğa 2 — Əlavə 3 (pul) API səviyyəsində
- Paket kataloqu, yer başına qiymət, endirim riyaziyyatı, pulqab minimumu, saxta ödəniş bildirişi (404), RBAC
- Bilinən problemlər canlıda ölçüldü: planlayıcıda saat qurşağının olmaması, illik paketin alına bilməməsi, kart saxlamanın söndürülməsi

### Dalğa 3 — Brauzer UI passı
- Desktop 1905px, mobil 390px, tünd tema
- Yaratma modalları, scope tabları, sayğaclar, landing qiymətləri
- Əlçatanlıq əsasları: bütün 57 düymənin etiketi var, alt-siz `<img>` yoxdur; **`<html lang>` tətbiqdə sabit `en`** aşkarlandı

### Dalğa 4 — Super admin ilə blokların açılması
İstifadəçi «full access» verdikdən sonra əvvəl mümkün olmayan testlər açıldı:
- Təsdiqlərin deterministik yaradılması (cədvəl yolu) → APPR-02/03/05/06/07/11
- Access-rule ilə Gmail yazma əməliyyatının təsdiqə salınması → APPR-01 (çatdakı təsdiq kartı)
- Təzə free hesab qeydiyyatı → APPR-10 (paket kilidi)
- REG-15/16 (sahiblik keçidi) brauzerlə
- SCHED-07/08/09, WORKER zənciri (yaratma → təklif → təyinat → sessiya)

### Dalğa 5 — «Yaşıl qrup» və ilk hesabat
- WALLET-07, WSHOP-03/06, I18N-01, LAND-02/03/04/05, RESP-02/04
- WSHOP-04 fərqi aşkarlandı (balans çatmayanda 402 yox, checkout)
- İlk tam hesabat yazıldı və push edildi

### Dalğa 6 (retry #2) — Ən böyük açılış
- **PAY-02 uçdan-uca ödəniş**: checkout → kart → 3DS → `payment=ok` → abunə free→pro, pulqab 0→5 ₼
- **WALLET-08**: `balance_enforced` artıq `true` → bütün zəncir yoxlanıldı
- **OFFER yaratma ucu tapıldı** (`POST /billing/offers/admin`) → OFFER-04/05 tam, OFFER-01 təhlükəsiz hissələr, OFFER-02 (M-4)
- **🔴 H-1 aşkarlandı** — limitli kampaniyanın limitdən çox istifadəsi (iki real sandbox ödənişi ilə sübut)

### Dalğa 7 — Dürüst yenidən sayım
İstifadəçi «həqiqətən neçəsi edilib?» deyə soruşdu. Əvvəlki «119/132 (90 %)» rəqəmi **sübut jurnalında adı keçən** testləri sayırdı — səxavətli idi. Hər ID əl ilə yenidən təsnif edildi: **101 tam / 12 qismən / 19 mümkün deyil**. Həmçinin PAY-03 səhv olaraq «canlıda qadağan» sayılmışdı — sandbox-da edilə bilərdi, edildi (PAY-03, PAY-13, PAY-04).

### Dalğa 8 — «Manual» sayılan 4 test
REG-03 ✅, REG-08 ✅, APPR-04 ✅ icra olundu; REG-10-un test obyektinin (1C bağlayıcısı) bu quraşdırmada **ümumiyyətlə mövcud olmadığı** aşkarlandı.

### Dalğa 9 — Qalan qismən testlər (8 → 3 → 2)
- RESP-01 beş enin hamısında, PAY-08 (M-7), WSHOP-02 (M-8), WORKER-05 (M-9), WORKER-06 (M-10), SUSP-02-nin səbəbi
- Sonra: **OFFER-01 tam bitdi** (riski sıfıra endirən quruluşla), REG-07 8 addımdan 6-sı, SUSP-02-nin əlçatmazlığı üç yolla sübut olundu

---

## 4. Bölmə-bölmə nəticələr (132 test ID)

| Bölmə | Testlər | Tam | Qismən | Mümkün deyil | Qeyd |
|---|---:|---:|---:|---:|---|
| **REG** — reyestr | 18 | 15 | 1 (07) | 2 (10, 11) | M-1 |
| **APPR** — təsdiqlər | 12 | 12 | 0 | 0 | Təhlükəsizlik qapıları keçdi |
| **SCHED** — cədvəllər | 10 | 10 | 0 | 0 | K-1 (TZ) |
| **WORKER** — AI işçilər | 10 | 10 | 0 | 0 | M-9, M-10 |
| **PROF** — profil/ayarlar | 8 | 8 | 0 | 0 | M-2 |
| **PLAN** — paketlər | 9 | 9 | 0 | 0 | K-2 (illik) |
| **PAY** — ödənişlər | 13 | 10 | 0 | 3 (06, 07, 12) | M-5, M-7 |
| **CARD** — kart | 6 | 1 | 0 | 5 | K-3 (söndürülüb) |
| **WALLET** — pulqab | 8 | 6 | 0 | 2 (05, 06) | L-7 |
| **WSHOP** — işçi kommersiyası | 7 | 6 | 0 | 1 (07) | M-8, M-11 |
| **OFFER** — təkliflər | 7 | 6 | 0 | 1 (07) | **H-1**, M-3, M-4 |
| **SPAY** — super-admin ödəniş | 7 | 5 | 0 | 2 (03, 04) | K-5 |
| **SUSP** — dayandırma | 5 | 0 | 1 (02) | 4 | M-6 |
| **LAND** — landing | 5 | 5 | 0 | 0 | — |
| **I18N** — lokalizasiya | 3 | 3 | 0 | 0 | K-4 |
| **RESP** — responsivlik | 4 | 4 | 0 | 0 | — |
| **CƏMİ** | **132** | **110** | **2** | **20** | |

### Test ID-lərinin tam siyahısı (statusla)
```
REG    01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07🟡 08✅ 09✅ 10❌ 11❌ 12✅ 13✅ 14✅ 15✅ 16✅ 17✅ 18✅
APPR   01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07✅ 08✅ 09✅ 10✅ 11✅ 12✅
SCHED  01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07✅ 08✅ 09✅ 10✅
WORKER 01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07✅ 08✅ 09✅ 10✅
PROF   01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07✅ 08✅
PLAN   01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07✅ 08✅ 09✅
PAY    01✅ 02✅ 03✅ 04✅ 05✅ 06❌ 07❌ 08✅ 09✅ 10✅ 11✅ 12❌ 13✅
CARD   01✅ 02❌ 03❌ 04❌ 05❌ 06❌
WALLET 01✅ 02✅ 03✅ 04✅ 05❌ 06❌ 07✅ 08✅
WSHOP  01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07❌
OFFER  01✅ 02✅ 03✅ 04✅ 05✅ 06✅ 07❌
SPAY   01✅ 02✅ 03❌ 04❌ 05✅ 06✅ 07✅
SUSP   01❌ 02🟡 03❌ 04❌ 05❌
LAND   01✅ 02✅ 03✅ 04✅ 05✅
I18N   01✅ 02✅ 03✅
RESP   01✅ 02✅ 03✅ 04✅
```
`✅` tam · `🟡` qismən · `❌` icra oluna bilmədi

---

## 5. Tapıntıların xülasəsi

> Hər tapıntının **tam bug hesabatı** (komponent, mühit, rol, təkrarlama addımları, gözlənilən/faktiki nəticə, təsir, istinad, severity) ayrıca `BUG-REPORTS-post-voice.md` sənədindədir.

### 5.1 🔴 Yüksək (1)
| # | Ad | Test | Sübut |
|---|---|---|---|
| **H-1** | Limitli kampaniya limitdən çox istifadə oluna bilir | OFFER-06 | `max_redemptions=1` təklif, iki paralel checkout, **hər ikisi ödənildi**: balans 0→20 ₼, ödənilən 18 ₼, sayğac 1-də qaldı |

### 5.2 🟠 Orta (11)
| # | Ad | Test | Yeni? |
|---|---|---|---|
| M-1 | Adi üzv xüsusi HTTP MCP bağlayıcısı yarada bilmir (sənədlə ziddiyyət) | REG-09 | yeni |
| M-2 | Parol qaydaları serverdə tətbiq olunmur (`min_length=1`) | PROF-03 | bilinən, yüksəldilir |
| M-3 | Təklif rədd mesajı yalnız ingiliscədir | OFFER-05 | yeni |
| M-4 | Aktiv təklifi redaktədə genişləndirmək təsdiqsiz keçir | OFFER-02 | bilinən, təsdiqləndi |
| M-5 | `?payment=error/cancel/unknown` eyni nikbin mesajı verir | PAY-05 | bilinən, təsdiqləndi |
| M-6 | Əl ilə dayandırılmış təşkilat ödəyə bilir, amma dayandırılmış qalır | SUSP-02 | qismən yeni |
| M-7 | Sifariş TTL-i (60 dəq) praktikada tətbiq olunmur (253–257 dəq `pending`) | PAY-08 | **yeni** |
| M-8 | Həcm zolağı (tier) yaradıla bilmir; API `tiers` sahəsini səssizcə udur | WSHOP-02 | **yeni** |
| M-9 | Söhbətdəki «Qoş» kartı qeyri-sabitdir (3 cəhddən 1-i) | WORKER-05 | **yeni** |
| M-10 | «Qoşulmalar» paneli ümumiyyətlə yoxdur | WORKER-06 | **yeni** |
| M-11 | Balans çatmayanda 402 yox, kart checkout-u açılır | WSHOP-04 | sənədlə fərq |

### 5.3 🟡 Aşağı (12)
L-1 ingiliscə API mesajları · L-2 yetim təsdiq (cədvəl silinəndə) · L-3 rədd səbəbinin server limiti yoxdur · L-4 `usd_azn_rate=1.8` (sənəddə 1.70) · L-5 en-US rəqəm formatı · L-6 profil sahələri təmizlənə bilmir · L-7 `no_saved_card` mesajı xam koddur · L-8 işçi ödənişlərində `masked_pan` yazılmır · L-9 `sales_contact_email` boş → «Satışla əlaqə» ana səhifəni açır · L-10 tarixçədə xam `DECLINED` kodu · L-11 təsdiq kartında kömək mətni yoxdur · L-12 «Hesab» sahəsi boş + tenant PATCH naməlum sahələri səssizcə udur

### 5.4 Bilinən problemlərin canlı təsdiqi (5)
K-1 planlayıcıda saat qurşağı yoxdur (+4 saat) · K-2 illik paket alına bilmir · K-3 kart saxlanmır (`payriff.card_save=false`) · K-4 `<html lang>` tətbiqdə sabit `en` · K-5 STT/TTS qiymətləri ekranda yoxdur (amma real xərc yaradır)

### 5.5 🟢 Müsbət nəticələr
**Düzəldiyi aşkarlanan köhnə problemlər (2):**
- OFFER-04 «Müddət» sütununun kəsilməsi — **artıq yoxdur** (DOM-da overflow aşkarlanmadı)
- WALLET-08 «Balans bitib» xəbərdarlığının heç vaxt çıxmaması — **artıq işləyir** (`balance_enforced` indi `true`)

**Təhlükəsizlik qapıları — hamısı keçdi:**
| Yoxlama | Nəticə |
|---|---|
| Özünü-təsdiq qadağası | ✅ Həll anında **403**; üstəlik konfiqurasiya anında da **422 «A user cannot approve their own operations»** |
| Təsdiqsiz icra olmaması | ✅ `auto_approve:false` cədvəl təsdiqsiz işləmir |
| Parol dəyişəndə token ləğvi | ✅ Köhnə token `401 «token has been invalidated»` |
| Təkrar e-poçt | ✅ `409 email_taken` |
| İstifadəçi öz təsdiqçisini dəyişə bilmir | ✅ `PATCH /auth/me` `tool_approvers`-i nəzərə almır |
| Saxta Payriff bildirişi | ✅ Səhv sirrli ünvan → **404** |
| RBAC (bütün `/super-admin/*`, `/approvals/stuck`, plagin/işçi yaratma) | ✅ üzv/admin → 403, tokensiz → 401, data sızmır |
| Uğursuz ödənişə kredit yazılmaması | ✅ `DECLINED` → balans dəyişmir |
| Dayandırılmış təşkilatda yazma | ✅ 403 `tenant_read_only` |

**Pul riyaziyyatı — hamısı düzgün:** paket qiymətləri · yer başına hesablama (19.99×4=79.96, 34×4=136) · endirim (10 ₼ → 9.00 tutulur / 10.00 yazılır) · AI qiymət nümunəsi ((3+15)/1000 × 1.8 × 1.25 = 0.0405 ₼) · balans proqnozu (226.25 ÷ (6.75/30) = 1005 gün, aşağı yuvarlaqlaşdırılıb) · pulqab 1:1

---

## 6. İcra oluna bilməyən 20 test — səbəblərlə

| Səbəb | Testlər | Say |
|---|---|---:|
| **Funksiya söndürülüb** — `payriff.card_save=false`, kart saxlanmır (CARD-01-də uğurlu ödənişdən sonra da `configured:false` ilə sübut olundu) | CARD-02…06 · WALLET-05/06 | 7 |
| **DB-yə yazmaq lazımdır** — `period_end` / `suspension_floor_date`; super-admin API bu sahələri nə oxuyur, nə yazır | SUSP-01/03/04/05 · OFFER-07 | 5 |
| **Paylaşılan produksiya konfiqini dəyişmək lazımdır** — bütün müştərilərə təsir edərdi | PAY-12 · WSHOP-07 (`payriff.enabled` söndürmək) · SPAY-03 (`payriff.mode=production` + test açarı) · SPAY-04 (canlı paket qiyməti) | 4 |
| **Backend sirri / infrastruktur girişi lazımdır** | PAY-06 (callback secret ilə təkrar bildiriş) · PAY-07 (bildiriş ünvanını kəsmək) | 2 |
| **Xarici xidmət qoşulması lazımdır** | REG-11 (sahibin canlı Gmail-ini ayırmaq olmaz — geri qoşulma CAPTCHA-ya düşərdi) | 1 |
| **Test obyekti bu quraşdırmada yoxdur** | REG-10 — 1C bağlayıcısı kataloqda yoxdur (25 bağlayıcı, `auth_kind` yalnız `bearer`/`mcp_oauth2`/`none`; `env_config` mövcud deyil) | 1 |

### Qalan 2 qismən test
| Test | Nə edildi | Nə qaldı |
|---|---|---|
| **REG-07** | 8 addımdan **6-sı**: «Bağlantı məlumatı» paneli (4 sahə) · popup `accounts.google.com`-da açılır · popup bloklandıqda mesaj **hərfi-hərfinə düzgündür** · popup yarımçıq bağlananda səhifə **ilişmir** (🔴 əlamət yoxdur) · qoşulu Gmail «Aktiv» · **bütün 🔴 yoxlamalar keçdi** | Addım 3-4: Google girişinin tamamlanması — **CAPTCHA** («Eşitdiyiniz və ya gördüyünüz mətni daxil edin»), **iki dəfə** sınandı |
| **SUSP-02** | `manual` ssenarisi **tam** icra olundu → M-6 | `unpaid` ssenarisi: üç yol sınandı — (1) `PATCH {"suspended_reason":"unpaid"}` həmişə `manual` yazır, (2) süpürgə ucu yoxdur (404), (3) `period_end` əlçatmazdır |

---

## 7. Canlı sistemə edilən təsirlər — tam audit izi

> Bu bölmə nəzərdən keçirən üçün ən vacibidir: **canlı produksiya sistemində nə dəyişdirildi və geri qaytarıldı.**

### 7.1 Yaradılan və silinən obyektlər
| Obyekt | Say | Vəziyyət |
|---|---|---|
| Cədvəllər (schedules) | 7 | **silindi** |
| Bacarıq (skill) | 1 | **silindi** |
| Plagin | 1 | **silindi** |
| Bağlayıcılar (connectors) | 2 | **silindi** |
| AI işçilər | 2 («QA Worker Chain», «QA Tier Test») | **silindi / arxivləndi** |
| Təkliflər (offers) | 9 | **hamısı `off`** (silmə ucu yoxdur — PATCH ilə söndürüldü) |
| QA istifadəçi hesabları | 4 | qaldı (zərərsiz test hesabları) |
| QA təşkilatları | 4 | qaldı; «QA Lock Test» **`active`** vəziyyətinə qaytarıldı |

### 7.2 Dəyişdirilən və geri qaytarılan ayarlar
| Dəyişiklik | Geri qaytarıldı? |
|---|---|
| Üzvün parolu (PROF-03 testi, 2 dəfə) | ✅ `1`-ə qaytarıldı, tokenlər yeniləndi |
| Üzvün profil sahəsi | ✅ bərpa edildi |
| Admin access-rule (gmail write=approval) | ✅ silindi |
| QA tenant-ın dayandırılması (manual + unpaid cəhdi) | ✅ `active`-ə qaytarıldı, yazmanın işlədiyi yoxlanıldı |
| Üzvə müvəqqəti AI işçi təyinatı | ✅ geri alındı |
| Platforma üzrə 1 %-lik təklif (OFFER-01) | ✅ **~70 saniyə sonra söndürüldü**, sızma olmadığı yoxlanıldı |

### 7.3 Səhvlər və onların düzəldilməsi (şəffaflıq üçün)
| Hadisə | Nə oldu | Düzəliş |
|---|---|---|
| Başqa istifadəçinin bağlayıcısı | Sniffing zamanı «adaasdas» adlı istifadəçinin `gcal` təyinatı `false`-a keçdi | **Dərhal `true`-ya qaytarıldı** |
| Sahibin real bacarığı | REG-16 təmizliyində «explain» (9 təşkilat / 14 nəfər istifadə edir) bacarığının dialoqu təkrar açılırdı | Skript **dialoq başlığını yoxlayacaq** şəkildə yenidən yazıldı; sahibin bacarığı **dəyişdirilmədi** |
| Sahibin AI işçisi | WSHOP-02-də səhvən «agent!» işçisinin redaktə forması açıldı | **Heç bir yazma göndərilmədi** (sniffer boş idi), forma bağlandı; sonra ad yoxlaması əlavə edildi |
| Təsadüfi 5 ₼ xərc | WSHOP-03 testində işçi abunəsi balansdan dərhal tutuldu | Abunə **ləğv edildi** (dövr sonuna qədər aktiv qalır — mid-period qaytarma yoxdur) |

### 7.4 Sandbox ödənişləri (real pul hərəkət etməyib)
| Məqsəd | Məbləğ |
|---|---|
| PAY-02 uçdan-uca paket ödənişi | 19.99 ₼ |
| H-1 sübutu (iki paralel topup) | 2 × 9.00 ₼ |
| WSHOP-05 kartla işçi alışı | 5.00 ₼ |
| SUSP-02 dayandırılmış təşkilatın ödənişi | 34.00 ₼ |
| PAY-03 rədd edilən ödəniş | 10.00 ₼ (rədd — tutulmadı) |

Ödənilməmiş bir neçə `pending` sifariş qaldı — M-7-yə görə TTL süpürgəsi onları gec bağlayır.

### 7.5 ⚠️ Təmizlənə bilməyən artefakt
APPR-04 testinin təsdiq addımında Gmail-də **`QA-TEST-SILIN`** adlı etiket **real olaraq yaradıldı**. Gmail bağlayıcısında etiket **silmək üçün alət yoxdur** (agent özü bunu bildirdi), ona görə silinə bilmədi.
**→ Sahib əl ilə silməlidir: Gmail → Parametrlər → Bütün parametrlərə bax → Etiketlər → `QA-TEST-SILIN` → Sil.**
(«QA-TEST-SILIN-2» yaradılmadı — o, rədd edildi.)

---

## 8. Qəsdən edilməyənlər (təhlükəsizlik qərarları)

Nəzərdən keçirən bunları «edilməyib» yox, «şüurlu şəkildə edilməyib» kimi oxumalıdır:

1. **Canlı paket qiymətinin dəyişdirilməsi (SPAY-04)** — imkan təsdiqləndi (`PATCH /super-admin/plans/{code}` əlçatandır), lakin real müştərilərin qiymətinə toxunmamaq üçün icra edilmədi
2. **`payriff.enabled` / `payriff.mode` söndürülməsi (PAY-12, WSHOP-07, SPAY-03)** — bütün müştərilərin ödənişini dayandırardı
3. **`payriff.order_ttl_minutes` dəyişdirilməsi (PAY-08)** — real müştərilərin açıq checkout-larını vaxtından əvvəl bağlaya bilərdi; əvəzinə canlı data üzərində ölçüldü və M-7 tapıldı
4. **Sahibin canlı Gmail bağlayıcısının ayrılması (REG-11)** — geri qoşulma CAPTCHA-ya düşərdi və sahibin işi pozulardı
5. **OFFER-02-də təhlükəli `PATCH`** — brauzerdə abort edildi; davranış müşahidə olundu, server dəyişmədi
6. **OFFER-01-in platforma üzrə aktivləşdirilməsi** — əvvəlcə tamamilə buraxıldı; son dalğada **riski sıfıra endirən quruluşla** edildi: 1 % endirim + `max_redemptions=1` + 3 dəqiqəlik özü-bağlanan pəncərə (maksimum mümkün itki ≈ 0.10 ₼), ~70 saniyədən sonra əl ilə söndürüldü

---

## 9. Bu testin öz zəif tərəfləri (self-critique)

Nəzərdən keçirən üçün dürüst siyahı:

1. **Qara-qutu məhdudiyyəti** — kök səbəblər koddan deyil, davranışdan çıxarılıb. «Kök səbəb» kimi yazılan hər şey **müşahidəyə əsaslanan fərziyyədir** (məs. H-1-də «qapı yalnız `redemption_count`-a baxır»)
2. **DB-yə baxış yoxdur** — `payments.offer_id`, `offer_redemptions` cədvəli kimi bəzi gözləntilər yalnız API cavabları ilə dolayı yoxlanıb
3. **AI axınlarının qeyri-determinizmi** — WORKER-05 (M-9) məhz buna görə 3 dəfə təkrarlandı; digər AI-asılı testlərdə də nəticə dəyişkən ola bilər
4. **Keçici toast-lar** — nümunələmə üsulu ilə tutulub; çox qısa görünən bir bildiriş buraxılmış ola bilər
5. **Yalnız Chromium** — Safari/Firefox yoxlanılmayıb
6. **Mobil yalnız 390px emulyasiya** — real cihaz, toxunuş jestləri, Android/iOS tətbiqi yoxlanılmayıb (Əlavə 4 əhatədən kənardır)
7. **Yük/performans testi yoxdur** — yalnız funksional
8. **Bir anın şəkli** — bəzi nəticələr (məs. M-7-dəki `pending` sifarişlərin yaşı, kataloqdakı `balance_enforced`) ölçmə anının vəziyyətidir; konfiqurasiya dəyişsə nəticə dəyişər
9. **Əvvəlki sayım səhvi** — ilk hesabatda «119/132 (90 %)» yazılmışdı; bu, sübut jurnalında **adı keçən** testləri sayırdı. Dalğa 7-də hər ID əl ilə yenidən təsnif edilib; indiki rəqəmlər (110/2/20) bu yenidən sayımın nəticəsidir

---

## 10. Buraxılış üçün tövsiyə olunan prioritet

| # | Məsələ | Niyə |
|---|---|---|
| 1 | **H-1** — limitli kampaniyanın limitdən çox istifadəsi | Yeganə Yüksək; birbaşa pul təsiri; düzəlişi lokaldır (checkout-da atomar rezervasiya: `remaining = max − redeemed − in_progress`) |
| 2 | **M-7** — sifariş TTL-i işləmir | H-1-in pəncərəsini saatlarla uzadır; süpürgənin tezliyi düzəldilməlidir |
| 3 | **M-2** — serverdə parol siyasəti yoxdur | Təhlükəsizlik; hesab ələ keçirmə riski |
| 4 | **K-1** — planlayıcıda saat qurşağı yoxdur | İstifadəçiyə real təsir (+4 saat) |
| 5 | **M-6** — dayandırılmış təşkilat ödəyir, amma açılmır | Müştəri pul verir, nəticə almır |
| 6 | **M-5** — `cancel`/`unknown` yanlış nikbin mesaj | Dəstək yükü və etimad |
| 7 | M-4, M-9, M-10, M-8, M-3, M-11 və L-* qeydləri | Növbəti sprint |

---

## 11. Təhvil verilən sənədlər

| Fayl | Məzmun |
|---|---|
| `QA-REPORT-post-voice.md` (~62 KB) | Əsas QA hesabatı: statistika, tapıntılar, bölmə-bölmə nəticələr, dalğa-dalğa icra qeydləri, metodologiya |
| `BUG-REPORTS-post-voice.md` (~39 KB) | **24 bug hesabatı** standart formatda (1 Yüksək + 11 Orta + 12 Aşağı) + 5 bilinən problemin təsdiqi |
| `qa-evidence/raw-findings-log.md` (~84 KB) | Xam, əlavə-yalnız sübut jurnalı — hər dalğanın bütün ölçmələri, API cavabları, ekran mətnləri |
| `QA-SESSION-REPORT.md` | **Bu sənəd** — sessiyanın tam icmalı |

**Branch:** `claude/qa-testing-handoff-voice-t1np98`

### Son commit-lər
```
f92381d QA: aşağı səviyyəli buglar (L-1…L-12) + bilinən problemlərin təsdiqi (K-1…K-5)
a665a91 QA: 12 bug hesabatı (1 Yüksək + 11 Orta) standart formatda
e277c60 QA: son 3 qismən test — OFFER-01 tam bitdi, REG-07 6/8, SUSP-02 əlçatmazlığı sübut olundu
b4f4c42 QA: qalan 8 qismən testdən 5-i tamamlandı — 4 yeni Medium tapıntı
c79f41b QA: 4 "manual" testdən 3-ü icra olundu — REG-03, REG-08, APPR-04 keçdi
2d144c8 QA: dürüst yekun sayım — 101 tam / 12 qismən / 19 mümkün deyil (+PAY-03/04/13)
6818941 QA: retry pass #2 tamamlandı — əhatə 90% (119/132), yeni M-6 və I18N-02 təsdiqi
f7d3899 QA: retry pass #2 — PAY-02/WALLET-08/OFFER bölmələri açıldı, 1 Yüksək tapıntı
0effbd4 QA: finish green group (WALLET-07, WSHOP-03/06, +WSHOP-04 finding)
8e38ae8 QA: re-check pass — I18N-01 + LAND-05 newly passed
```
*(Əvvəlki dalğaların commit-ləri branch tarixçəsindədir.)*

---

## 12. Yekun qiymətləndirmə

**Buraxıla bilər — amma iki şərtlə.**

Voice-dən sonrakı modullar memarlıq baxımından sağlamdır: icazə modeli ardıcıldır, təsdiq sistemi əsas vədini (özünü-təsdiq qadağası, təsdiqsiz icra olmaması) **iki səviyyədə** yerinə yetirir, paket bayraqları və rol qapıları düzgün tətbiq olunur, yoxlanılan **bütün pul hesablamaları dəqiqdir**, validasiya mesajları təmizdir (xam 500 yoxdur), UI tam lokallaşdırılıb, konsol və şəbəkə təmizdir. **Kritik qüsur tapılmadı.**

Buraxılışdan əvvəl bağlanmalı iki məqam:
1. **H-1** — `max_redemptions` limitli kampaniya limitdən çox istifadə oluna bilir (iki real sandbox ödənişi ilə sübut olunub; `topup`/`ai_worker` üçün 100 % endirim icazəli olduğundan itki potensialı limitsizdir)
2. **M-2** — parol siyasətinin serverdə yoxlanılmaması

Bu dalğada **iki köhnə problemin düzəldiyi** də aşkarlandı (OFFER-04 sütun kəsilməsi, WALLET-08 xəbərdarlığı) — bu, kataloq/konfiqurasiya dəyişikliklərindən sonra təkrar yoxlamanın nə qədər faydalı olduğunu göstərir.

---

*Hazırlayan: avtomatlaşdırılmış QA dalğası (Claude Code) · 2026-10-16 · bütün ölçmələr canlı sistemdə aparılıb və sübut jurnalında qeydə alınıb.*
