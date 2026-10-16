# OrgSysAI — QA Hesabatı (Voice bölməsindən sonrakı testlər)

**Tester:** Senior QA (avtomatlaşdırılmış qara-qutu dalğası)
**Tarix:** 2026-10-16
**Mühit:** Canlı sistem — `https://orgsysai.com` (ayrıca staging yoxdur)
**Əhatə:** Əlavə 2-nin **VOICE-dən sonrakı** bölmələri (REG · APPR · SCHED · WORKER · PROF) + bütün **Əlavə 3** (PLAN · PAY · CARD · WALLET · WSHOP · OFFER · SPAY · SUSP · LAND · RESP)
**Metod:** API səviyyəsində qara-qutu (3 rol: super admin `dev@berc.az`, təşkilat admini `kanan.ismayilzada@berc.az`, üzv `yepyenitestuseri@gmail.com`) + brauzer duman testi (Playwright/Chromium, 1905px, AZ)

---

## 0. Bir cümləlik nəticə (sahibə)

> **Buraxıla bilər — amma şərtlə.** Voice-dən sonrakı funksionallıqda (reyestr, təsdiqlər, cədvəllər, AI işçilər, profil, ödəniş/paket) **yeni Kritik və ya Yüksək qüsur tapılmadı**: təhlükəsizlik qapıları (özünü-təsdiq qadağası, RBAC, paket kilidləri, saxta ödəniş bildirişi, təkrar e-poçt, parol dəyişəndə token ləğvi) **işləyir**, və pul riyaziyyatı (qiymətlər, yer başına ödəniş, endirim, pulqab) **düzgündür**. Buraxılışdan əvvəl düzəldilməsi tövsiyə olunan yeganə ciddi məqam — **parol qaydalarının serverdə yoxlanılmaması** (artıq “bilinən” kimi qeyd olunub, amma təhlükəsizlik baxımından yüksəldilməlidir).

---

## 1. Statistika

| Göstərici | Say |
|---|---:|
| API səviyyəsində icra edilən / təsdiqlənən yoxlama | **29 PASS** + 12 bilinən davranışın canlı ölçülməsi |
| Brauzer duman testi (ekran) | 5 ekran (registry, approvals, schedules, profile, settings) — hamısı §0.4 məcburi yoxlamalarından keçdi |
| **Yeni Kritik qüsur** | **0** |
| **Yeni Yüksək qüsur** | **0** |
| Orta səviyyəli qeyd (sənədlə uyğunsuzluq / zəiflik) | 2 |
| Aşağı səviyyəli qeyd | 4 |
| Bilinən problemlərin canlı təsdiqi (yeni deyil) | 4 (TZ · illik paket · kart saxlama · zəif parol serveri) |
| İcra edilə bilməyən (səbəbi ilə, §2.3) | ~25 UI/mobil + 11 canlıda qadağan (§0.4) |

---

## 2. Tapıntılar

### 2.1 🔴 Təhlükəsizlik qapıları — HAMISI KEÇDİ (ən vacib nəticə)

| Test | Nəticə | Sübut (API) |
|---|---|---|
| **APPR-05** özünü-təsdiq qadağası | ✅ PASS | Təsdiqçisi `admin` olan sorğunu **sahib özü həll edə bilmir** → `403 "This approval must be resolved by your designated approver: …"`; delegat (admin) həll edir → `approved`. |
| **SCHED-04** təsdiq tələb edən cədvəl | ✅ PASS | `skip_approval:false` cədvəl işə salınanda `queued_for_approval` / `pending_approval` olur və **təsdiqsiz icra olunmur**. |
| **PROF-03** parol dəyişəndə digər sessiyalar | ✅ PASS | Uğurlu dəyişiklik tokeni fırladır: köhnə token → `401 "token has been invalidated"`, çağıran yeni token alır (cari cihaz qalır, digərləri çıxır). |
| **PROF-02** başqasının e-poçtu | ✅ PASS | `409 {code:"email_taken"}` — qəbul olunmur. |
| **PROF-07** istifadəçi öz təsdiqçisini dəyişir | ✅ PASS | `PATCH /auth/me` ilə `tool_approvers` **nəzərə alınmır** (boş qalır). |
| **PAY-10** saxta Payriff bildirişi | ✅ PASS | Səhv sirrli ünvan (`/payments/payriff/SEHV-SIRR`) → `404` (emal edilmir). |
| **RBAC** (post-voice uclar) | ✅ PASS | `/approvals/stuck`, `/super-admin/*`, `/billing/offers/admin`, plagin yaratma, işçi yaratma, işçi abunəsi — üzv/admin üçün düzgün `403`; tokensiz → `401`. |

### 2.2 💰 Pul riyaziyyatı — HAMISI DÜZGÜN

| Test | Nəticə | Ölçülən |
|---|---|---|
| **PLAN-01/03/05/09** qiymət kataloqu | ✅ PASS | `free 0` · `pro 19.99/199.90` · `plus 34.00/340.00` · `enterprise/onpremise`=satışla əlaqə · `guest/platform` **gizli**. Hamısı gözləntiyə uyğun. |
| **PLAN-06** yer (seat) başına qiymət | ✅ PASS | Pro aylıq (4 yer) = **79.96** (19.99×4); Plus aylıq = **136.00** (34×4). |
| **PAY-11** məbləğ manatla | ✅ PASS | Checkout cavabı `amount: 79.96 / 136.00 AZN` — qəpik deyil. |
| **OFFER-03** endirim riyaziyyatı | ✅ PASS | Aktiv 30 ₼ sabit endirim: 50 ₼ doldurma → **tutulan 20.00 ₼ / yazılan 50.00 ₼**. Endirim = banner dəyəri. Endirim müştəri tərəfindən enjekt edilə bilmir (`offer_id` → `extra_forbidden`). |
| **WALLET-04** minimum doldurma | ✅ PASS | 5 ₼ → `422 {below_minimum "The smallest top-up is 10 AZN."}`; 10 ₼ → tutulan 10 / yazılan 10 (1:1). |
| **SPAY-05** AI qiymət riyaziyyatı | ✅ Uyğun | `usd_azn_rate=1.8 · margin=25%` → input $3/Mtok = 6.75 ₼/Mtok (daxili ardıcıl). |

### 2.3 Orta səviyyəli qeydlər (yeni, amma təhlükəsizlik/pul deyil)

**Qeyd M-1 — `[registry] Adi üzv xüsusi HTTP MCP bağlayıcısı yarada bilmir (sənədlə ziddiyyət)`**
- **Ciddilik:** Orta (funksional məhdudiyyət, təhlükəsizlik deyil — əksinə daha sərtdir)
- **Test ID:** REG-09
- **Addımlar:** Üzv tokeni ilə `POST /api/registry/connectors` (`transport:"mcp_http"`).
- **Gözlədim:** Sənəd (REG-09) deyir ki, HTTP serveri **adi üzv** yarada bilər.
- **Gördüm:** Üzv → `403 "Tenant admin access required"`. Admin → `201`. Yəni xüsusi bağlayıcı (HTTP daxil) **yalnız admindir**. Ayrıca `mcp_stdio` yalnız **super admin**dədir (admin belə `403` alır) — sənəd admini gözləyirdi.
- **Təkrarlanır?** 3/3.
- **Tövsiyə:** Ya sənəd (REG-09), ya da icazə qaydası uyğunlaşdırılmalıdır. Daha sərt olduğu üçün təhlükə yoxdur; sadəcə gözləntidən fərqlidir.

**Qeyd M-2 — `[profile] Parol qaydaları serverdə tətbiq olunmur` (bilinən #5 — yüksəldilməsi tövsiyə olunur)**
- **Ciddilik:** Orta→Yüksək (təhlükəsizlik zəifliyi)
- **Test ID:** PROF-03
- **Gördüm:** Server **3 simvolluq parolu**, **istifadəçi adı ilə eyni parolu** qəbul etdi və `confirm_password` uyğunsuzluğunu nəzərə almadı. Bütün qaydalar (≥10 simvol, istifadəçi adından fərqli, təsdiq uyğunluğu) yalnız **ön üzdədir**.
- **Qeyd:** Sənəddə “bilinən problem” kimi var, ona görə **yeni qüsur kimi açılmır**, amma parol siyasətinin serverdə yoxlanılmaması real zəiflikdir və buraxılışdan əvvəl düzəldilməsi tövsiyə olunur.

### 2.4 Aşağı səviyyəli qeydlər

- **L-1 (localization):** API xəta mesajları ingiliscədir (məs. stdio bağlayıcı izahı, cədvəl validasiyası). Yoxladığım UI ekranları **tam azərbaycanca** idi, ona görə bu yalnız xam API səviyyəsindədir — amma UI bəzi mesajları birbaşa göstərirsə, `az/ru` istifadəçi üçün tərcümə boşluğudur.
- **L-2 (schedules):** Cədvəl silinəndə onun **gözləyən təsdiqi ləğv olunmur** (yetim təsdiq qalır). Kiçik.
- **L-3 (approvals):** Rədd səbəbi serverdə 2000 simvolla məhdudlaşmır (2001 simvol qəbul edildi) — limit yalnız UI-dədir.
- **L-4 (ai-prices):** `usd_azn_rate` canlıda **1.8**-dir (sənəddə standart 1.70 yazılıb) — konfiqurasiya dəyişib, qüsur deyil, sadəcə qeyd.

### 2.5 Bilinən problemlərin canlı təsdiqi (yeni qüsur kimi açılmır)

| # | Bilinən problem | Canlı ölçmə |
|---|---|---|
| TZ | Planlayıcı saat qurşağı almır (SCHED-02) | Cədvəl modelində **ümumiyyətlə timezone sahəsi yoxdur**; `cron "0 9 * * *"` server (UTC) vaxtı ilə → Bakı üçün **+4 saat** fərq. Təsdiqləndi. |
| İllik | İllik paket alına bilmir (PLAN-07) | `cadence:"yearly"` checkout **aylıq məbləği** qaytarır (pro illik = 79.96, aylıqla eyni). Təsdiqləndi. |
| Kart | Kart saxlanmır (CARD-*) | `/payments/card` → `configured:false`. CARD-01…06 aparıla bilmir. Təsdiqləndi. |
| Parol | Serverdə parol minimumu 1 | Bax M-2. |

---

## 3. Bölmə-bölmə PASS xülasəsi

- **REG (reyestr):** REG-02 (yaratma/təkrar 409/boş 422) ✅ · REG-05 (qlobal bacarığı silmə qadağası 403) ✅ · REG-09 (MCP icazələri — bax M-1) ⚠️ · REG-14 (plagin yalnız admin 403) ✅ · UI (tablar, sayğac, scope) brauzerdə təsdiqləndi ✅. REG-15/16 (sahiblik keçidi) — endpoint qara-qutu ilə tapılmadı, UI tələb edir.
- **APPR (təsdiqlər):** APPR-02 (təsdiq) ✅ · APPR-03 (rədd+səbəb) ✅ · APPR-05 (özünü-təsdiq qadağası 🔴) ✅ · APPR-09 (stuck RBAC + bilinən boşluq) ✅ · resolve 404/401 təmiz ✅. Canlı AI alət-çağırışı tələb edən tam axın (APPR-01/04/11) interaktiv/manual tələb edir.
- **SCHED (cədvəllər):** SCHED-01 (yaratma+validasiya) ✅ · SCHED-02 (TZ, ölçüldü) 🔴-bilinən · SCHED-03 (indi işə sal) ✅ · SCHED-04 (təsdiq qapısı 🔴) ✅ · SCHED-05 (söndürmə) ✅ · SCHED-06 (silmə 204) ✅.
- **WORKER (AI işçilər):** WORKER-01 (yaratma/boş 422/təkrar 409/admin 403) ✅ · WORKER-08 (istifadəsiz işçi həqiqətən silinir, cavab `archived:false`) ✅ · WORKER-09 (üzv abunə ola bilmir 403, qiymət AZN) ✅. Təklif/vermə/geri-alma asimmetriyası və işçi söhbət sessiyası (WORKER-03/05/06/07) çox-hesablı UI tələb edir.
- **PROF (profil/ayarlar):** PROF-01 (profil saxlanır, istifadəçi adı dəyişməz) ✅ · PROF-02 (e-poçt validasiyası+təkrar) ✅ · PROF-03 (sessiya ləğvi 🔴 + M-2 zəifliyi) ✅ · PROF-07 (təsdiqçi yalnız oxunur) ✅ · PROF-04/05/06 (/settings yalnız brauzerdə — dizayn) ✅ (UI-də tam azərbaycanca göründü).
- **Əlavə 3 (ödəniş/paket):** PLAN qiymətləri ✅ · yer başına ödəniş ✅ · OFFER-03 endirim ✅ · WALLET minimum/1:1 ✅ · PAY-10 saxta bildiriş ✅ · SPAY RBAC ✅.

---

## 4. İcra edilə bilməyən testlər və səbəbi

**A. Canlı sistemdə qadağan olanlar (§0.4):** SUSP-01…05 (hamısı `period_end`/`suspension_floor_date` baza yazması tələb edir) · OFFER-01/02 (boş filtrli təklif = bütün platformaya endirim) · PAY-03/07/08 · CARD-05 · OFFER-06 · SPAY-03. **Səbəb:** canlı bazaya yazma / real ödənişlərin sındırılması / 24 saat gözləmə.

**B. CARD-01…06 və avtomatik yenilənmə:** Payriff `autoPay` söndürülüb (bilinən). Aparıla bilmir.

**C. İnteraktiv/qeyri-deterministik (AI söhbət axını):** APPR-01/02/03/04 (🔁3), WORKER-03/04/05, CHAT-dan irəli gələn təsdiq kartları — canlı AI alət-çağırışı tələb edir; avtomatlaşdırılmış təkrarlana bilən qara-qutu üçün uyğun deyil. **Təhlükəsizlik qapısı** cədvəl-mənşəli təsdiqlə dolayı yolla sübut edildi (SCHED-04 + APPR-05).

**D. Sırf UI/vizual/mobil:** drag-drop fayl yükləmə (REG-04), OAuth popup (REG-07), token maskalanması (REG-08), 1C formu (REG-10), 390px mobil (hər bölmənin son testi), tünd tema (PROF-06/RESP-03), landing vizual qiymət bölməsi (LAND-01/05, SPA JS ilə render olunur). **Tövsiyə:** bunlar manual/interaktiv sessiyada tamamlanmalıdır.

---

## 5. Metodologiya və məhdudiyyətlər (şəffaflıq)

- Testlər **qara-qutu** API çağırışları (curl) + bir brauzer duman testi ilə aparılıb. Backend/frontend mənbə kodu əlçatan deyildi (verilən repozitoriya ayrı bir homework reposudur).
- Pul hərəkət etməyib: bütün ödəniş testləri **sandbox** (`payriff.mode=test`) rejimindədir; checkout/topup yalnız **başladılıb**, 3-D Secure tamamlanmayıb (canlı abunəni dəyişməmək üçün). Bir neçə `pending` Payriff sifarişi 60 dəqiqəlik TTL ilə öz-özünə bitəcək.
- **Yaradılan bütün test obyektləri təmizləndi:** 7 cədvəl, 1 bacarıq, 1 plagin, 2 bağlayıcı, 1 AI işçi silindi; gözləyən təsdiqlər rədd edildi; üzvün parolu (müvəqqəti PROF-03 testi üçün dəyişdirilmişdi) **`1`-ə geri qaytarıldı**; profil sahəsi bərpa edildi. Başqa təşkilatların datasına toxunulmadı.
- Təsdiqçi/təşkilat adminin verdiyi e-poçt siyahısında kiçik yazı səhvi var idi: admin `kanan.ismayilzad**e**@berc.az` deyil, **`kanan.ismayilzad**a**@berc.az`** (tapşırıq mətnindəki ilk yazılışla) düzgün işləyir.

---

## 6. Yekun qiymətləndirmə

Voice-dən sonrakı modullar **memarlıq baxımından sağlamdır**: icazə modeli ardıcıldır, təsdiq sistemi öz əsas vədini (özünü-təsdiq qadağası, təsdiqsiz icra olmaması) yerinə yetirir, paket bayraqları və rol qapıları düzgün tətbiq olunur, və bütün yoxlanılan pul hesablamaları (qiymət, yer başına, endirim, pulqab) **dəqiqdir**. Validasiya mesajları təmizdir (xam 500 yoxdur), UI tam lokallaşdırılıb və konsol/şəbəkə təmizdir.

Buraxılış üçün diqqət tələb edən üç məqam **artıq bilinən**dir və bu dalğada **canlı ölçülüb təsdiqləndi**: (1) planlayıcıda saat qurşağı yoxluğu (+4 saat), (2) illik paketin alına bilməməsi, (3) kart saxlamanın söndürülməsi. Bunlardan **birincisi (TZ)** istifadəçiyə real təsir edir və tez düzəldilməlidir. Ayrıca **serverdə parol siyasətinin olmaması (M-2)** — sənəddə “bilinən” olsa da — təhlükəsizlik baxımından buraxılışdan əvvəl bağlanması tövsiyə olunur.

**Qalan UI/mobil/interaktiv testlər** (§4-C və §4-D) avtomatlaşdırılmış qara-qutu ilə örtülmədi və manual sessiyada tamamlanmalıdır; bu hesabat onların yerini dəqiq göstərir.
