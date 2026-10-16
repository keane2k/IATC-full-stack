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
| Brauzer UI testi (desktop 1905px + mobil 390px + tünd tema) | **15+ ekran/axın** — hamısı §0.4 məcburi yoxlamalarından keçdi |
| **Yeni Kritik qüsur** | **0** |
| **Yeni Yüksək qüsur** | **0** |
| Orta səviyyəli qeyd (sənədlə uyğunsuzluq / zəiflik) | 2 |
| Aşağı səviyyəli qeyd | 6 |
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
- **L-5 (format, PLAN-09/LAND-01):** həm landing-də, həm tətbiqdə onluq ayırıcı **nöqtədir** (`19.99 ₼`, `34.00 ₼`, `231.39 ₼`), amma sənəd §0.2 AZN üçün **vergül** gözləyir (`19,99 ₼`). Ardıcıl i18n/format uyğunsuzluğu.
- **L-6 (profil, PROF-01):** profil mətn sahələri (məs. “Vəzifə”) `PATCH /auth/me` ilə **təmizlənə bilmir** — boş sətir, `null` və boşluq hamısı köhnə dəyəri saxlayır (dəyişiklik baş vermir), həm də cavabdakı dəyər saxlanan dəyərlə ziddiyyətlidir.

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

## 3.5 Brauzer UI nəticələri (Playwright — desktop 1905px · mobil 390px · tünd tema)

Parol qapısı (`bercberc`) və giriş işlədi. Bütün yoxlanılan ekranlarda **§0.4 məcburi yoxlamalar keçdi**: üfüqi sürüşmə yoxdur · xam i18n açarı yoxdur · xam JSON/`[object Object]` yoxdur · **konsol xətası sıfır** · gözlənilməz 4xx/5xx yoxdur.

- **LAND-01 🔴 PASS:** landing (az + en) qiymət bölməsi backend ilə **uyğun** (Free · Pro 19.99 ₼ · Plus 34.00 ₼ · Enterprise/On-premise “Satışla əlaqə”; guest/platform gizli). Səhifə tam (hero, imkan kartları, bağlayıcı loqoları, FAQ, hüquqi footer).
- **REG-01/02 UI PASS:** scope tabları (Hamısı/Təşkilat/Şəxsi) + sayğac; “Yeni bacarıq yarat” modalı (ad/izah/parametrlər).
- **SCHED-01 UI PASS:** yaratma modalı tam — Başlıq · Vəziyyət · Cədvəl (Gündəlik) **+ saat (08:00)** · “Bitmə tarixi təyin et” · Tapşırıq mətni · “Təsdiqi ötür” açarı · **“Kim təsdiqləyir”** təsdiqçi seçimi + izah mətni (APPR-05 delegasiyasını vizual təsdiqləyir). Xam cron göstərilmir.
- **PLAN-02/03 + WALLET-01/02 UI PASS:** `/chat/plans` “Paketlər və balans” — balans 231.39 ₼, 30 günlük sərf + proqnoz, Plus=“Cari paket”, “Paketi yalnız admin dəyişə bilər” qeydi.
- **APPR-08 UI PASS:** görünüşlər (Mənim / Mənə təyin edilib) + üç sekme (Gözləyən/Təsdiqlənmiş/Rədd).
- **Mobil 390px (REG-18 · APPR-12 · SCHED-10 · PROF-08 · RESP-01) PASS:** heç bir ekranda üfüqi sürüşmə yoxdur; cədvəl yaratma modalı ekrana sığır, “Yarat” düyməsi görünür.
- **Tünd tema (PROF-06 · RESP-03) PASS:** fon qaranlıq, mətn oxunur, **forma qutuları fondan fərqlənir** (əvvəlki reqressiya təkrarlanmır); açarlar və “Yadda saxla” görünür.

## 3.6 REG bölməsinin tam icrası (full-access pass — admin + super admin)

Sahibdən tam icazə (admin + super admin + real Google hesabı) alındıqdan sonra REG brauzerlə genişləndirildi. **Yekun: 11 PASS + 1 fərq + 5 qismən + 1 xarici-bloklu.**

- **REG-04 ✅** fayl yükləmə: `test.pdf` qəbul · `qadagan.exe` rədd («Unsupported file type '.exe'») · 25 MB+ rədd («Maksimum ölçü: 25 MB»).
- **REG-06 ✅** admin 3 sekme (Mağaza/Mənim əlavələrim/Qoşulanlar) + kateqoriya süzgəcləri + loqolu kartlar (Gmail qoşulu ✓) + boş axtarış mesajı.
- **REG-13 ✅** komanda modalı (Bacarıq seçimi + ad + arqumentlər).
- **REG-15 ✅ + YENİ TAPINTI:** sahiblik keçidi **asimmetrikdir** — **«Təşkilata yüksəlt» (promote) təsdiq SORUŞMUR** (sənəddəki UX riski təsdiqləndi), amma **«Sahiblik statusuna endir» (demote) TƏSDİQ soruşur** («…şəxsi edilsin? Bu əməl geri qaytarılmır»). Sənəd «ikisi də soruşmur» deyirdi → demote yaxşılaşıb.
- **REG-16 ✅** super admin: **«Bütün təşkilatlara aç»** (tenant admindəki «Təşkilata yüksəlt» ilə qarışmır — rol etiketləri düzgün); platforma yüksəltmə/endirmə işləyir.
- **REG-03/17 🟡** offline: təhlükəli boş-forma / səssiz-boş siyahı **müşahidə olunmadı** (yaxşı), amma açıq xəta/qoruma mesajı avtomatlaşdırma ilə təsdiqlənmədi — manual lazımdır.
- **REG-08/10 🟡** token qoşma dialoqu / 1C formu skriptlə etibarlı açıla bilmədi → token maskalanması (🔴) manual yoxlanmalıdır.
- **REG-12 ✅** alət təsdiq səviyyələri: Google Təqvim detalında hər alətin açılan menyusu **üç səviyyəni** göstərir (Avtomatik icazə / Təsdiq tələb edir / Bloklanıb). Həmçinin REG-06 detal görünüşü təsdiqləndi (geri düyməsi, «Bağlantı məlumatı» paneli, Nəqliyyat=mcp_http·OAuth 2.0).
- **REG-07 🟡** (real Google hesabı ilə) **tətbiq tərəfi düzgün**: «Qoşul» düyməsi düzgün Google icazə pəncərəsini açır — «Sign in with Google — to continue to **orgsysai.com**» + Privacy/Terms linkləri (OAuth tətbiqi düzgün qurulub). Uçdan-uca giriş **avtomatlaşdırma ilə tamamlana bilmir**: Google avtomatlaşdırılmış brauzeri aşkar edib **CAPTCHA** («Type the text you hear or see») göstərir. İnsan tester bunu keçir; skript keçə bilmir — **bu OrgSysAI qüsuru deyil**.
- **REG-11 ❌** qoşulu xidmətin ayrılması: yeni qoşula bilən xidmət lazımdır; sahibin canlı Gmail qoşulması pozulmasın deyə toxunulmadı (yenidən qoşma eyni CAPTCHA-ya düşür).

> ⚠️ **Təhlükəsizlik qeydi:** REG-16 təmizləməsində sahibin real platforma bacarığı «explain» (9 təşkilat / 14 nəfər istifadə edir) səhvən hədəflənə bilərdi; təsdiqdən əvvəl dialoq başlığı yoxlanıb dayandırıldı — **toxunulmadı**. Yalnız `qa-*` test obyektləri dəyişdirildi və hamısı silindi.

## 3.7 APPR bölməsinin tam icrası (full-access pass)

**Yekun: 11 PASS + 1 qismən.** Təsdiq MÜHƏRRİKİ + paket-gating tam yoxlanıldı.
- **APPR-06 ✅** «Parametrlərə bax»: Giriş parametrləri + Sistem məlumatı cədvəlləri, kopyala ikonları, xam obyekt yoxdur.
- **APPR-07 ✅** «Həvalə parametrləri» delegasiya paneli + «Planlaşdırılmış alətlər» seçicisi.
- **APPR-08 ✅** sayğaclar · axtarış · görünüşlər (Mənim / Mənə təyin edilib).
- **APPR-11 ✅** ilk qərar qalib gəlir; həll olunmuşa ikinci müraciət → **409 «approval already resolved»** (təmiz).
- **APPR-01 ✅** (real Gmail yazma ilə): admin access-rule (gmail write_mode=approval, delegat=üzv). Söhbətdə Gmail YAZMA (create_draft) → **təsdiq kartı çıxdı**: «Agent gmail bağlayıcısında create_draft alətini icra etmək istəyir» + Bağlayıcı/Alət + JSON parametrlər + «Təsdiq gözlənilir». Sorğu /approvals-da da yarandı.
- **🔒 Təhlükəsizlik (yaxşı dizayn):** write_mode=approval yalnız YAZMA-nı tutur — Gmail OXUMA (son məktubun mövzusu) avtomatik icra olundu, YAZMA (create_draft) təsdiq tələb etdi. Oxu/yaz fərqi düzgündür.
- **APPR-03 ✅** (real yazma ilə): create_draft təsdiqi RƏDD edildi → status «rejected», qaralama YARADILMADI (qapı yazmanı dayandırdı).
- **APPR-10 ✅** (yeni `free` hesab): free paketdə approvals=False; GET /approvals → **403 struktur `plan_required`** (feature/plan/mesaj) — xam 500/ağ ekran deyil; frontend bunu kilid paneli kimi göstərir. (Mesaj ingiliscədir → lokalizasiya boşluğu.)
- **APPR-04 🟡** söhbətdən özü «approve»: mexanizm mövcuddur (söhbət «Agentin sualına cavab verin…» rejiminə keçir), amma özünü-təsdiq variantı alət-səviyyəli öz konfiqı tələb edir (access-rule məcburi təsdiq üçün delegat tələb edir). Delegat axını tam işləyir.

Əvvəl API ilə sübut olunmuşdu: APPR-02 (təsdiq), **APPR-05 (🔴 özünü-təsdiq qadağası)**, **APPR-09 (🔴 stuck RBAC)**, APPR-12 (mobil).

## 4. İcra edilə bilməyən testlər və səbəbi

**A. Canlı sistemdə qadağan olanlar (§0.4):** SUSP-01…05 (hamısı `period_end`/`suspension_floor_date` baza yazması tələb edir) · OFFER-01/02 (boş filtrli təklif = bütün platformaya endirim) · PAY-03/07/08 · CARD-05 · OFFER-06 · SPAY-03. **Səbəb:** canlı bazaya yazma / real ödənişlərin sındırılması / 24 saat gözləmə.

**B. CARD-01…06 və avtomatik yenilənmə:** Payriff `autoPay` söndürülüb (bilinən). Aparıla bilmir.

**C. İnteraktiv/qeyri-deterministik (AI söhbət axını):** APPR-01/02/03/04 (🔁3), WORKER-03/04/05, CHAT-dan irəli gələn təsdiq kartları — canlı AI alət-çağırışı tələb edir; avtomatlaşdırılmış təkrarlana bilən qara-qutu üçün uyğun deyil. **Təhlükəsizlik qapısı** cədvəl-mənşəli təsdiqlə dolayı yolla sübut edildi (SCHED-04 + APPR-05).

**D. Qalan sırf-interaktiv UI (brauzer passından sonra qalanlar):** drag-drop fayl yükləmə və tip/ölçü limitləri (REG-04), real OAuth popup axını (REG-07), token sahəsinin maskalanması (REG-08), 1C forму (REG-10), işçi söhbət sessiyası və “Qoş” kartı (WORKER-04/05/06), LAND-05 (backend söndükdə fallback). **Qeyd:** 390px mobil (REG-18/APPR-12/SCHED-10/PROF-08/RESP-01), tünd tema (PROF-06/RESP-03), yaratma modalları (REG-02/SCHED-01) və landing qiyməti (LAND-01) **artıq brauzerlə icra olundu** (§3.5). Qalanlar üçün real xarici xidmət / fayl / provayder qoşulması lazımdır.

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
