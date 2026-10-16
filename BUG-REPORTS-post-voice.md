# OrgSysAI — Bug hesabatları (Yüksək + Orta)

**Mənbə:** Voice-dən sonrakı QA dalğası · 2026-10-16 · canlı sistem
**Test mühiti:** Chromium (Playwright, headless) · Linux x64 · az dili + REST API (curl)
**Hesablar:** super admin `dev@berc.az` · təşkilat admini `kanan.ismayilzada@berc.az` · üzv `yepyenitestuseri@gmail.com` · QA üçün yaradılmış test hesabları

---

## H-1 — Limitli kampaniya limitdən çox istifadə oluna bilir

**Komponent:** backend / billing (təkliflər · checkout)
**Mühit:** https://orgsysai.com/chat/plans → Payriff (sandbox) · REST API · az dili
**Rol/Hesab:** super admin (təklifi qurur) + tenant admin (alış edir)

**Təkrarlama addımları:**

1. Super admin: `POST /api/billing/offers/admin` ilə `topup` hədəfli, **`max_redemptions = 1`**, 10 % endirimli aktiv təklif yarat
2. Tenant admin: `POST /api/billing/wallet/topup {"amount_azn":"10"}` çağır — sifariş #1 yaranır (tutulan 9.00 ₼)
3. Ödənişi **tamamlama**, təklifin vəziyyətinə bax: `in_progress_count = 1`, `remaining_redemptions = 1`
4. Eyni tenant ilə **ikinci** `POST /api/billing/wallet/topup {"amount_azn":"10"}` çağır
5. Hər iki sifarişi Payriff-də ödə (3DS → `payment=ok`)
6. Balansı və təklifin `redemption_count` sayğacını yoxla

**Gözlənilən nəticə:** Təklif checkout başlayanda bronlanmalıdır (şlüz çağırılmazdan əvvəl) — birinci sifariş bronu tutur, ikinci sifariş **tam qiyməti** görür. `max_redemptions = 1` olan kampaniya yalnız **bir dəfə** istifadə oluna bilər.

**Faktiki nəticə:** İkinci sifariş də endirimli açıldı (9.00 ₼, eyni `offer_id`). Hər ikisi ödənildi: balans **0 → 20.00 ₼**, ödənilən cəmi **18.00 ₼** — kampaniya **iki dəfə** işlədildi. Təklifin sayğacı isə `redemption_count = 1`-də qaldı. Yalnız ödəniş **oturduqdan sonra** qapı işləyir (üçüncü checkout düzgün şəkildə tam qiymət — 10.00 ₼, `offer_id:""`).

**Təsir:** Limitli kampaniya istənilən sayda istifadə oluna bilər: N ədəd checkout aç, sonra hamısını ödə. `topup` və `ai_worker` üçün **100 % endirim icazəlidir** (OFFER-05-ə bax), ona görə `max_redemptions=1` + 100 % topup təklifi ilə itki **limitsizdir** (N dəfə pulsuz balans). Sayğac real istifadəni az göstərdiyi üçün problem hesabatlarda da görünmür. Formanın öz vədi pozulur: «Bir təşkilat onsuz da yalnız bir dəfə istifadə edə bilir».

**Kök səbəb (davranışdan):** Qapı yalnız `redemption_count`-a baxır; `in_progress_count` `remaining`-dən çıxılmır. Checkout yaradılması ↔ ödənişin oturması arasındakı pəncərə qorunmur. M-7 (TTL süpürgəsinin gecikməsi) bu pəncərəni saatlarla uzadır.

**Təklif:** Checkout yaradılarkən atomar rezervasiya — `remaining = max − redeemed − in_progress`; rezervasiyanın TTL ilə geri buraxılması.

**İstinad:** QA plan OFFER-06 (İki tabda eyni təklif — bron) — uğursuzluq əlaməti: «Hər iki tab endirimli sifariş açır → limitli kampaniya limitdən çox istifadə olunur»
**Severity: High · Prioritet: High · Status: Open**

---

## M-1 — Adi üzv xüsusi HTTP MCP bağlayıcısı yarada bilmir (sənədlə ziddiyyət)

**Komponent:** backend / registry (bağlayıcılar)
**Mühit:** https://orgsysai.com · REST API · az dili
**Rol/Hesab:** adi üzv · təşkilat admini · super admin

**Təkrarlama addımları:**

1. Üzv tokeni ilə `POST /api/registry/connectors` göndər, `transport: "mcp_http"`
2. Cavabı yaz
3. Eyni sorğunu təşkilat admini ilə təkrarla
4. `transport: "mcp_stdio"` ilə hər iki rolda təkrarla

**Gözlənilən nəticə:** REG-09-a görə HTTP serverini **adi üzv** yarada bilər; `mcp_stdio` isə admin səviyyəsindədir.

**Faktiki nəticə:** Üzv → **403 «Tenant admin access required»**. Admin → 201 (uğurlu). `mcp_stdio` isə **yalnız super admindədir** — admin də 403 alır. Təkrarlanma: 3/3.

**Təsir:** Təhlükə yoxdur — faktiki icazə modeli sənəddən **daha sərtdir**. Lakin sənəd və məhsul bir-birinə uyğun deyil: ya REG-09 yenilənməli, ya icazə qaydası sənədə uyğunlaşdırılmalıdır. Müştəri sənədə baxıb üzvlərin öz MCP serverlərini qoşa biləcəyini gözləyə bilər.

**İstinad:** QA plan REG-09 (MCP bağlayıcı icazələri)
**Severity: Medium · Prioritet: Low · Status: Open**

---

## M-2 — Parol qaydaları serverdə tətbiq olunmur

**Komponent:** backend / auth (parol dəyişmə · qeydiyyat)
**Mühit:** https://orgsysai.com · REST API · az dili
**Rol/Hesab:** adi üzv (istənilən rol)

**Təkrarlama addımları:**

1. `POST /api/auth/change-password` çağır, yeni parol kimi **3 simvol** («abc») göndər
2. Cavabı yaz
3. Yeni parol kimi **istifadəçi adının özünü** göndər
4. `confirm_password` sahəsini **uyğun gəlməyən** dəyərlə göndər
5. Platforma ayarlarında `auth.password_min_length` dəyərini oxu

**Gözlənilən nəticə:** Ön üzdəki qaydalar (≥10 simvol, istifadəçi adından fərqli, təsdiq uyğunluğu) serverdə də tətbiq olunmalıdır — zəif parol qəbul edilməməlidir.

**Faktiki nəticə:** Server **3 simvolluq parolu qəbul etdi**, **istifadəçi adı ilə eyni parolu qəbul etdi**, `confirm_password` uyğunsuzluğunu **nəzərə almadı**. Platforma ayarı: `auth.password_min_length = 1`. Bütün qaydalar yalnız ön üzdədir.

**Təsir:** Ön üzü keçərək (API, köhnə mobil tətbiq, skript) istənilən zəif parol qoyula bilər. Parol siyasəti faktiki olaraq mövcud deyil — hesab ələ keçirmə riski. Sənəddə «bilinən problem» kimi qeyd olunub, lakin təhlükəsizlik baxımından buraxılışdan əvvəl bağlanmalıdır.

**Qeyd:** Test zamanı üzvün parolu müvəqqəti dəyişdirilmiş və dərhal `1`-ə geri qaytarılmışdır.

**İstinad:** QA plan PROF-03 (Parol dəyişmə və sessiyaların ləğvi) · bilinən problem №5
**Severity: Medium (təhlükəsizlik baxımından High) · Prioritet: High · Status: Open**

---

## M-3 — Təklif rədd mesajı yalnız ingilis dilindədir

**Komponent:** backend / billing (təkliflər)
**Mühit:** https://orgsysai.com/super-admin/offers · REST API · az dili
**Rol/Hesab:** super admin

**Təkrarlama addımları:**

1. Dili `az` et
2. `plan` hədəfli, **faiz = 100** təklif yaratmağa çalış
3. Cavabdakı `detail` mətnini yaz
4. Eyni şeyi `free_period` və paketin tam qiymətinə bərabər sabit endirimlə təkrarla

**Gözlənilən nəticə:** Üç halın da rədd edilməsi (düzgündür) + rədd səbəbinin **seçilmiş dildə, insan dilində** verilməsi.

**Faktiki nəticə:** Məntiq tam düzgündür (hər üç hal 400 ilə rədd olunur), lakin səbəb **yalnız ingiliscədir**:
`«A plan offer cannot take the price to zero. Give that organisation a free period on its plan instead. Here, 100 % off leaves nothing to charge.»`
`«…a free period on a plan is a full discount, and nothing charges for the period it covers.»`
`«…19.99 ₼ off meets or exceeds the plan price (19.99 ₼), which leaves nothing to charge.»`

**Təsir:** Azərbaycan dilində işləyən super admin texniki ingiliscə mətn görür. Testin öz uğursuzluq əlaməti: «Rədd səbəbi xam ingiliscə və ya texniki mətndir».

**İstinad:** QA plan OFFER-05 (100 % / pulsuz dövr təklifi paketə yazıla bilmir) — uğursuzluq əlaməti: «Rədd səbəbi xam ingiliscədir»
**Severity: Medium · Prioritet: Low · Status: Open**

---

## M-4 — Aktiv təklifi redaktədə genişləndirmək təsdiqsiz keçir

**Komponent:** web / super-admin (təkliflər)
**Mühit:** https://orgsysai.com/super-admin/offers · Chromium · az dili
**Rol/Hesab:** super admin

**Təkrarlama addımları:**

1. Məhdudiyyətli təklif qur (yalnız `pro` paketi, bir təşkilata bağlı) və **aktivləşdir**
2. Aç/söndür açarı ilə aktivləşdirəndə təsdiq pəncərəsinin çıxdığını qeyd et
3. Həmin **aktiv** təklifin «Redaktə» formasını aç
4. Paket məhdudiyyətini (`Pro` qeyd qutusunu) **söndür** — yəni təklifi genişləndir
5. «Saxla» bas
6. Təsdiq pəncərəsinin çıxıb-çıxmadığını yoxla

**Gözlənilən nəticə:** Aktiv təklifi məhdudiyyətsiz formaya salmaq aktivləşdirmə qədər təhlükəlidir və **eyni təsdiqi** tələb etməlidir.

**Faktiki nəticə:** Aç/söndür açarı təsdiq istəyir («Təklif hamı üçün işə salınsın?»), lakin **redaktədə heç bir təsdiq çıxmır** — `PATCH /api/billing/offers/admin/{id}` dərhal göndərilir. Tutulan yük: `{"eligible_plan_codes":[], …, "active":true}`. Məhdudiyyət silindikdən sonra formadakı qırmızı xəbərdarlıq da yenilənmir.

**Təsir:** Artıq aktiv olan təklif redaktədə **səssizcə** platforma üzrə genişləndirilə bilər və dərhal qüvvəyə minir — heç kim xəbərdar edilmir. Təsdiq qoruması asimmetrikdir.

**Test təhlükəsizliyi:** Sorğu brauzerdə abort edildi — dəyişiklik serverə çatmadı, canlıda heç nə dəyişmədi (sonradan yoxlanıldı: təklif hələ də `eligible_plan_codes=['pro']`).

**İstinad:** QA plan OFFER-02 (Aktiv təklifi boş filtrə çevirmək) — bilinən boşluq, canlıda təsdiqləndi
**Severity: Medium · Prioritet: Medium · Status: Open**

---

## M-5 — `?payment=` dörd dəyərindən yalnız ikisi düzgün ayrılır

**Komponent:** web / chat (paketlər səhifəsi)
**Mühit:** https://orgsysai.com/chat/plans · Chromium · az dili
**Rol/Hesab:** tenant admin

**Təkrarlama addımları:**

1. `/chat/plans?payment=ok` aç, çıxan bildirişi oxu
2. `/chat/plans?payment=failed` aç, bildirişi oxu
3. `/chat/plans?payment=error` aç, bildirişi oxu
4. `/chat/plans?payment=cancel` aç, bildirişi oxu
5. `/chat/plans?payment=unknown` aç, bildirişi oxu
   *(bildiriş keçici toast-dır — 150 ms-lik nümunələmə ilə tutulur)*

**Gözlənilən nəticə:** Dörd dəyərin hər biri **öz mənasına uyğun** mesaj verməlidir; ləğv edilmiş ödəniş «gözləyin» deməməlidir.

**Faktiki nəticə:**
- `ok` → «Ödəniş alındı. Paketiniz yeniləndi.» ✅
- `failed` → «Ödəniş alınmadı.» ✅
- `error` → «Ödəniş yoxlanılır. Bank cavab verən kimi paketiniz yenilənəcək.» ❌
- `cancel` → **eyni** «Ödəniş yoxlanılır…» mesajı ❌
- `unknown` → **eyni** «Ödəniş yoxlanılır…» mesajı ❌

**Təsir:** Yalnız `failed` ayrıca tutulur; qalan hər şey nikbin «gözləyin» qoluna düşür. Ödənişi **özü ləğv edən** istifadəçiyə «ödənişiniz yoxlanılır, paketiniz yenilənəcək» deyilir. `unknown` halında sifariş nömrəsi ümumiyyətlə gəlməyib — heç nə yoxlanmır, amma istifadəçiyə gözləmək tövsiyə edilir. Dəstəyə şikayət mənbəyidir. (Müsbət: xam kod/i18n açarı sızmır.)

**İstinad:** QA plan PAY-05 (`?payment=` parametrinin dörd dəyəri) — bilinən davranış, canlıda təsdiqləndi
**Severity: Medium · Prioritet: Medium · Status: Open**

---

## M-6 — Əl ilə dayandırılmış təşkilat ödəyə bilir, amma dayandırılmış qalır

**Komponent:** backend / billing + tenant lifecycle
**Mühit:** https://orgsysai.com/chat/plans → Payriff (sandbox) · REST API · az dili
**Rol/Hesab:** super admin (dayandırır) + həmin təşkilatın admini (ödəyir)

**Təkrarlama addımları:**

1. Super admin: `PATCH /api/super-admin/tenants/{id} {"status":"suspended"}` — təşkilat `suspended_reason = manual` ilə dayandırılır
2. Həmin təşkilatın admini ilə gir, söhbətdə mesaj göndər — nəticəni yaz
3. Balans doldurmağa çalış (`POST /api/billing/wallet/topup`) — nəticəni yaz
4. Paket almağa çalış (`POST /api/payments/checkout {"plan_code":"plus"}`) — nəticəni yaz
5. Ödənişi **tamamla** (kart + 3DS)
6. Təşkilatın `status` və `suspended_reason` dəyərlərinə, abunəyə və yazma əməliyyatına bax

**Gözlənilən nəticə:** Ya ödəniş bloklanmalıdır (balans doldurma kimi), ya da ödənişdən əvvəl istifadəçi xəbərdar edilməlidir ki, bu ödəniş təşkilatı açmayacaq.

**Faktiki nəticə:**
- Söhbət → **403 `tenant_read_only`** (düzgün)
- Balans doldurma → **403 `tenant_read_only`** (bloklanır)
- Paket checkout-u → **200** (bloklanmır, ödəniş linki verilir)
- Ödəniş tamamlandı: **34.00 ₼ tutuldu**, abunə **pro → plus, active** oldu
- **Lakin** təşkilat hələ də `status=suspended`, `suspended_reason=manual`; yazma yenə **403 `tenant_read_only`**

**Təsir:** Müştəri 34 ₼ ödəyir, paketi yüksəlir, amma təşkilatı **istifadə edə bilmir** və ödənişdən əvvəl **heç bir xəbərdarlıq almır**. Balans doldurma bloklanır, paket ödənişi isə bloklanmır — davranış ziddiyyətlidir. (Ödənişin yalnız `unpaid` səbəbini qaldırması dizayn qərarıdır; boşluq xəbərdarlığın və checkout blokunun olmamasıdır.)

**Təklif:** `suspended_reason = 'manual'` halında ya checkout-u da blokla (topup kimi), ya da ödəniş səhifəsindən əvvəl «Bu təşkilat sahib tərəfindən dayandırılıb — ödəniş onu açmayacaq» xəbərdarlığı göstər.

**Test təhlükəsizliyi:** QA üçün yaradılmış öz test təşkilatımda aparıldı; sonda `active` vəziyyətinə qaytarıldı və yazmanın bərpası yoxlanıldı.

**İstinad:** QA plan SUSP-02 (Ödəniş edəndə dayandırmanın avtomatik qalxması) — addım 7
**Severity: Medium (müştəri təsiri baxımından High) · Prioritet: High · Status: Open**

---

## M-7 — Sifariş TTL-i praktikada tətbiq olunmur

**Komponent:** backend / billing (uzlaşdırma süpürgəsi)
**Mühit:** https://orgsysai.com/super-admin/payments · REST API · az dili
**Rol/Hesab:** super admin

**Təkrarlama addımları:**

1. `GET /api/super-admin/platform-settings` → `payriff.order_ttl_minutes` dəyərini yaz
2. `GET /api/super-admin/payments` → bütün `pending` sətirlərin `created_at` tarixini oxu
3. Hər sətrin yaşını (indiki vaxt − `created_at`) hesabla və TTL ilə müqayisə et
4. `expired` sətirlərin yaşına da bax

**Gözlənilən nəticə:** TTL (60 dəqiqə) keçən sifarişlər süpürgə tərəfindən `expired` edilməlidir.

**Faktiki nəticə:** Ayar **`payriff.order_ttl_minutes = 60`**, lakin canlıda **8 `pending` sifariş** var və yaşları: **31 · 60 · 85 · 253 · 255 · 256 · 257 · 257 dəqiqə** — dördü TTL-dən **4 dəfə çox** yaşayıb və hələ də `pending`-dir. Bütün **19 `expired`** sətir isə **dünənkidir** (yaş ~1760–1830 dəq ≈ 29–30 saat). Yəni süpürgə işləyir, amma 60 dəqiqəlik TTL-ə görə yox — görünür gündəlik (və ya çox seyrək) qrafiklə.

**Təsir:**
1. **H-1-i gücləndirir:** təklif bronu (`in_progress_count`) və `pending` işçi abunəsi məhz bu süpürgə ilə buraxılmalıdır — süpürgə vaxtında işləmirsə, bron saatlarla tutulu qalır
2. Müştərinin ödəniş tarixçəsində «Gözləyir» sətri saatlarla asılı qalır
3. Konfiqurasiya dəyəri yanıltıcıdır — sahib 60 dəqiqə gözlədiyini düşünür

**Qeyd:** Testin öz addımı (`order_ttl_minutes`-i 5-ə endirmək) paylaşılan produksiya konfiqi olduğu üçün icra edilmədi; nəticə canlı data üzərində ölçüldü.

**İstinad:** QA plan PAY-08 (Sifarişin vaxtının bitməsi — TTL)
**Severity: Medium · Prioritet: Medium · Status: Open**

---

## M-8 — Həcm zolağı (tier) ümumiyyətlə yaradıla bilmir; API sahəni səssizcə udur

**Komponent:** web / super-admin (AI mütəxəssislər) + backend / billing
**Mühit:** https://orgsysai.com/super-admin/ai-workers · Chromium · REST API · az dili
**Rol/Hesab:** super admin

**Təkrarlama addımları:**

1. Yeni AI mütəxəssis yarat, «Redaktə et» aç, qiymət təyin et və saxla
2. Formadakı **«Həcm pillələri»** bölməsinə bax — pillə əlavə etmək üçün element axtar
3. API ilə sına: `PUT /api/super-admin/ai-workers/{id}/price` gövdəsinə `tiers:[{"min_seats":1,"max_seats":4,"unit_price_azn":"10"},{"min_seats":5,"max_seats":10,"unit_price_azn":"8"}]` əlavə et
4. Cavabdakı `tiers` sahəsini yoxla
5. Ayrıca uclar sına: `/tiers`, `/volume-tiers`, `/price/tiers`

**Gözlənilən nəticə:** Super admin həcm zolağı qura bilməlidir (məs. 1–4 yer = 10 ₼, 5–10 yer = 8 ₼); `quotes` cavabında `tier_label` dolmalı, kartda «zolaq tətbiq olundu» sətri görünməlidir.

**Faktiki nəticə:**
- UI-da «Həcm pillələri» **bölməsi var**, amma yalnız mətn göstərir: «Həcm pilləsi yoxdur — hər adam sayında siyahı qiyməti tətbiq olunur.» **Pillə əlavə etmək üçün heç bir düymə/sahə yoxdur** (bölmə ətrafındakı yeganə düymə «Qiyməti yadda saxla»-dır)
- API: `tiers` sahəsi ilə sorğu **200 qaytarır, amma cavabda `tiers: []`** — sahə **səssizcə atılır** (nə `extra_forbidden`, nə xəbərdarlıq)
- Ayrıca uclar → **404**

**Təsir:**
1. Reklam olunan funksiya (həcm endirimi) işləmir — data modelində `tier_count`, `tier_label`, `anomaly` sahələri var, mexanizm yoxdur
2. API naməlum sahəni səssizcə udur — inteqrasiya yazan tərəf zolaqların saxlandığını düşünə bilər
3. **Müsbət tərəf:** WSHOP-02-nin 🔴 ssenariləri (üst-üstə düşən zolaqda ucuzun seçilməsi, boşluqda qiymətin 0 olması) bu buraxılışda **baş verə bilməz**

**Test təhlükəsizliyi:** Öz test işçim («QA Tier Test») üzərində aparıldı və sonda silindi; sahibin real işçisinə heç nə yazılmadı.

**İstinad:** QA plan WSHOP-02 (Həcm zolağı (tier) tətbiqi)
**Severity: Medium · Prioritet: Low · Status: Open**

---

## M-9 — Söhbətdəki «Qoş» kartı qeyri-sabitdir

**Komponent:** web / chat (AI mütəxəssis sessiyaları)
**Mühit:** https://orgsysai.com/chat · Chromium · az dili
**Rol/Hesab:** adi üzv (həmin xidməti **qoşmamış**), işçi sessiyası

**Təkrarlama addımları:**

1. Gmail-i qoşmamış üzv hesabı ilə gir
2. Həmin üzvə Gmail bağlayıcısı olan AI mütəxəssis təyin et
3. «AI mütəxəssislər» → mütəxəssis → «Yeni sessiya» aç
4. «Oxunmamış məktublarımı göstər.» yaz və göndər
5. Nəticəni yaz
6. **Addım 3-5-i 3 dəfə təkrarla** (test `🔁3` işarəlidir)

**Gözlənilən nəticə:** Hər dəfə söhbətdə «Gmail xidmətini qoşun» kartı çıxmalıdır: «Bu işi görmək üçün AI işçiyə Gmail xidmətinə sizin öz girişiniz lazımdır. Bir dəfə qoşun — AI işçi işinə davam edəcək.» + «Qoş» düyməsi.

**Faktiki nəticə:**
- **1/3 ✅** kart çıxdı — mətn demək olar eynidir: «**Gmail xidmətini qoşun**» · «Bu işi görmək üçün **AI mütəxəssisə** Gmail xidmətinə sizin öz girişiniz lazımdır. Bir dəfə qoşun — AI mütəxəssis işinə davam edəcək.» + «**Qoş**». Kart **yalnız bir xidməti** adlandırır (düzgün, tərkib açılmır)
- **2/3 ❌** agent əvvəlcə `list_connectors` çağırdı, bağlantı olmadığını gördü və **kart çıxarmadan** mətnlə cavab verdi:
  «Gmail-i Registry səhifəsindən qoşun, sonra son məktubunuzu göstərə bilərəm.»
  «E-poçt bağlayıcısını Registry səhifəsindən qoşduqdan sonra yenidən istəyin.»

**Təsir:** Vəd edilən «bir kliklə qoş və agent davam etsin» axını **üç haldan yalnız birində** işləyir. Qalan hallarda istifadəçi başqa səhifəyə yönləndirilir, söhbət kəsilir və geri qayıdıb sualı yenidən yazmalı olur. Davranış agentin hansı aləti əvvəl çağırmasından asılıdır — qeyri-deterministikdir.

**Qeyd:** Sənəddəki «AI işçiyə» ifadəsi məhsulda «AI mütəxəssisə»-dir — adlandırma dəyişikliyidir, qüsur deyil.

**İstinad:** QA plan WORKER-05 (Söhbətdəki «Qoş» kartı 🔁3)
**Severity: Medium · Prioritet: Medium · Status: Open**

---

## M-10 — «Qoşulmalar» paneli ümumiyyətlə yoxdur

**Komponent:** web / chat (AI mütəxəssis sessiyaları)
**Mühit:** https://orgsysai.com/chat · Chromium · az dili
**Rol/Hesab:** adi üzv **və** tenant admin (hər ikisi yoxlanıldı)

**Təkrarlama addımları:**

1. AI mütəxəssis sessiyası aç və bir neçə mesaj yaz
2. Söhbət başlığındakı **«⋮»** («Sessiya əməliyyatları») menyusunu aç
3. **«Qoşulmalar»** bəndini axtar
4. Müqayisə üçün: adi (işçi olmayan) söhbətdə də eyni menyunu aç
5. Testi həm heç nə qoşmamış üzvlə, həm də Gmail-i qoşulu olan adminlə təkrarla

**Gözlənilən nəticə:** «⋮» menyusunda **yalnız işçi sessiyasında** «Qoşulmalar» bəndi olmalıdır; panel «Bu AI işçi üçün qoşduğunuz xidmətlər» başlığı, boş vəziyyət mətni və sətir içi «Ayır» təsdiqi ilə açılmalıdır.

**Faktiki nəticə:** İşçi sessiyasında «⋮» menyusundakı bəndlər: «Layihəyə köçür», «Adını dəyiş», «Sessiyanı sil». **«Qoşulmalar» bəndi yoxdur.** Adi söhbətdə də menyu eynidir. **Hər iki rolda** yoxlandı (üzv: heç bir xidmət qoşmayıb · tenant admin: Gmail qoşulub) — ikisində də yoxdur, yəni «qoşulma varsa görünür» şərtinə görə gizlənmir.

**Təsir:** İstifadəçi AI mütəxəssis üçün hansı xidmətləri qoşduğunu görə və onları ayıra bilmir. WORKER-06-nın addımlarının **heç biri** icra edilə bilmir. Qoşulmanı geri almaq üçün yeganə yol ümumi Registry səhifəsidir ki, o da işçi kontekstini göstərmir.

**İstinad:** QA plan WORKER-06 («Qoşulmalar» paneli)
**Severity: Medium · Prioritet: Medium · Status: Open**

---

## M-11 — Balans çatmayanda 402 yox, kart checkout-u açılır

**Komponent:** backend / billing (AI işçi abunəsi)
**Mühit:** https://orgsysai.com/chat/workers · REST API · az dili
**Rol/Hesab:** tenant admin

**Təkrarlama addımları:**

1. Balansı məlum olan təşkilatla gir (məs. 226 ₼)
2. `GET /api/billing/workers/quotes?seats=50&cadence=monthly` ilə qiyməti yaz (50 yer = 250 ₼ — balansdan çox)
3. `POST /api/billing/workers/{id}/subscribe {"seats":50,"cadence":"monthly"}` çağır
4. HTTP kodunu və cavabı yaz
5. `pay_from_balance` sahəsi ilə təkrarla

**Gözlənilən nəticə:** WSHOP-04-ə görə balans çatmayanda **402** qaytarılmalı və üç rəqəm göstərilməlidir (lazım olan · mövcud · çatışmayan).

**Faktiki nəticə:** **402 qaytarılmır.** Əvəzinə `pending` abunə + şlüz ödəniş sifarişi yaradılır (`settled:false`, `payment_id` dolu, `paid_from:"card"`) — yəni sistem səssizcə kartla ödəniş yoluna keçir. `pay_from_balance` sahəsi qəbul olunmur (`extra_forbidden`).

**Təsir:** Sənəddəki davranışla fərqlənir. Müştəriyə «balansınız çatmır» mesajı əvəzinə birbaşa kart ödənişi təklif olunur — balansdan ödəmək istəyən müştəri bunu fərq etməyə bilər. Həmçinin yaranan `pending` abunə M-7 səbəbindən saatlarla asılı qalır.

**İstinad:** QA plan WSHOP-04 (Balans çatmayanda işçi alışı)
**Severity: Medium · Prioritet: Low · Status: Open**

---

*Aşağı səviyyəli (L-1 … L-12) qeydlər ayrıca `QA-REPORT-post-voice.md` sənədinin §2.4 bölməsindədir.*
