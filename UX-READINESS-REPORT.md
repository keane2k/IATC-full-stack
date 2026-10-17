# OrgSysAI — Adi istifadəçi gözü ilə hazırlıq qiymətləndirməsi

> **Metod:** Yeni alıcı kimi davranış — pro paketli hesabla daxil olub platformanı gəzmək, gündəlik aksiyaları icra etmək, «istehsala hazırdırmı?» sualına cavab vermək. Brauzer (Chromium), AZ dili. Təhlükəsizlik/API deyil — **real müştəri təcrübəsi**.
> **Tarix:** 2026-10-17 · **Hesab:** pro paket, balans 4.97 ₼ (yeni ödəmiş müştəri kimi)

## Yekun: 🟢 Əsas təcrübə istehsala hazırdır
Gəzdiyim hər ekran yükləndi, **0 konsol xətası**, boş-vəziyyətlər düzgün dizayn olunub, əsas iş axınları (söhbət, cədvəl qurma, balans, üzv əlavə) uçdan-uca işləyir. Tapılan məsələlər **cilalama/UX** səviyyəsindədir, bloklayıcı deyil.

## İcra edilən alıcı aksiyaları

| Aksiya | Nəticə |
|---|---|
| İlk söhbət — «Sən nə edə bilirsən?» | ✅ AI **4 saniyədə** azərbaycanca, səmimi, strukturlu cavab verdi |
| Bütün yan-menyu bölmələrini gəzmək (8 bölmə) | ✅ hamısı yükləndi, **0 konsol xətası** |
| Boş-vəziyyət ekranları | ✅ yaxşı: «Hələ heç bir cədvəl yoxdur. İlk avtomatik tapşırığınızı qurun» + CTA |
| **Cədvəl qurmaq (uçdan-uca UI)** | ✅ modal açıldı → dolduruldu → «Yarat» → siyahıda göründü, statistika «1 Aktiv» oldu |
| Balans / paket səhifəsi | ✅ aydın: balans, sərf proqnozu, artırma paneli, avtomatik doldurma izahı |
| Bağlayıcılar mağazası | ✅ görünür (Gmail, Təqvim, Notion və s.) |
| Audit jurnalı (Pro-da yoxdur) | ✅ təmiz paket-qapısı: «Audit jurnalı Pro paketinizə daxil deyil. Paketi yüksəldin» |
| Komandaya üzv əlavə | ✅ «İşçi əlavə et» forması işlək (ad, e-poçt, rol İşçi/Admin, «Hər əlavə işçi ayda 19.99 ₼») |

## Tapılan məsələlər (cilalama — bloklayıcı deyil)

### UX-1 — E-poçt təsdiq banneri ödəmiş müştəriyə də göstərilir
**Ciddilik:** Aşağı (UX) · **Ekran:** hər yer (üst zolaq)
Pro paketə keçmiş, balansı **4.97 ₼** olan müştəriyə hələ də «**E-poçtunuzu təsdiqləyin — pulsuz 1 AZN balans təsdiqdən sonra veriləcək**» banneri göstərilir. Artıq ödəmiş müştəri üçün «pulsuz 1 AZN» vədi məntiqsiz görünür və hesabın yarımçıq olduğu təəssüratı yaradır.
**Tövsiyə:** Ödənişli paketdə və ya balans > 1 ₼ olduqda bu banneri gizlət (və ya yalnız «e-poçtu təsdiqlə» mesajını saxla, «pulsuz 1 AZN» hissəsini çıxar).

### UX-2 — «İşçi» termini iki ayrı şeyə işlənir (qarışıqlıq)
**Ciddilik:** Aşağı (terminologiya) · **Ekran:** Təşkilat + AI mütəxəssislər
«İşçilər» həm **insan komanda üzvləri** («İşçi əlavə et» = yeni istifadəçi), həm də **AI mütəxəssisləri/işçiləri** (AI worker) üçün işlənir. Alıcı «İşçi əlavə et» düyməsinin insan, yoxsa AI əlavə etdiyini qarışdıra bilər. (Sol menyuda «AI mütəxəssislər», təşkilatda isə «İşçilər» — ad uyğunsuzluğu da var.)
**Tövsiyə:** İnsan üzvlər üçün «Komanda üzvü / İstifadəçi», AI üçün «AI mütəxəssis» terminini ardıcıl işlət.

### UX-3 — Üzv əlavə «hesab yarat» üsuludur, «dəvət linki» deyil
**Ciddilik:** Aşağı (axın seçimi) · **Ekran:** Təşkilat → İşçilər
Admin üzvü **birbaşa hesab yaradaraq** əlavə edir (istifadəçi adı + e-poçt, «Parolun bərpası üçün işlənir»), e-poçt dəvət linki göndərməklə yox. İşləkdir, amma müasir SaaS gözləntisi (dəvət linki → üzv özü qeydiyyatdan keçir) ilə fərqlənir. Qeyd kimi saxlanılır.

## Qeyd: artıq sənədləşdirilmiş, alıcıya təsir edən məsələlər
Bu dalğadan kənar, əvvəlki passlarda tapılmış və **adi istifadəçiyə də təsir edən** məsələlər (tam təsvir `BUG-REPORTS-post-voice.md`-dədir):
- **K-1 (SCHED-02):** cədvəl saat qurşağı almır → alıcının qurduğu «hər səhər 9:00» tapşırığı Bakı vaxtı ilə +4 saat sürüşür. **Alıcı üçün real təsir.**
- **M-5 (PAY-05):** ödənişi ləğv edən müştəriyə «ödənişiniz yoxlanılır» deyilir.
- **L-5:** rəqəm formatı en-US (`19.99 ₼`, `4,474 gün`) — AZN üçün vergül gözlənilir.
- **M-9/M-10 (WORKER-05/06):** AI işçi ilə xidmət qoşma axını qeyri-sabit; «Qoşulmalar» paneli yoxdur.

## Nəticə
Platformanın **gündəlik istifadə təcrübəsi hamardır və istehsala hazırdır** — iş axınları işləyir, ekranlar təmizdir, dil tamdır, boş-vəziyyətlər düşünülüb. Buraxılışdan əvvəl ən çox alıcıya təsir edən iki məsələ: **cədvəl saat qurşağı (K-1)** və **e-poçt banneri (UX-1)**. Qalanlar tədricən cilalana bilər.
