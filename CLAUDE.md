# CLAUDE.md

> İnşaat Proje Finansal Yönetim Uygulaması — (çalışma adı; marka adı henüz seçilmedi, repo adınla değiştir)
> Founding tenant: **Çetin İnşaat** · İlk gerçek veri: **Pursaklar projesi**

Bu dosya Claude Code için projenin ana rehberidir. Her oturumda önce bunu oku. Mevcut, formülle birbirine bağlı 9–12 sayfalık Excel sistemini (sıfırdan ZIP/XML ile .xlsx üreten bir araçtı) gerçek bir web uygulamasına taşıyoruz. Excel'in sayfaları burada **tablo değil, varlık (entity)** olur; "Sapma" gibi şeyler saklanmaz, sorguyla anlık hesaplanır.

---

## Çalışma tarzı (önce bunu içselleştir)

- Dil **Türkçe**. Doğrudan, kısa, kararlı ol. Seçenek menüsü sunma — öner ve uygula.
- Küçük, atomik commit'ler. Her görev tek bir işe odaklı.
- Küçük, atomik commit'ler at; çalışan branch'e push edebilirsin (Burak onayladı).
- Lint/format **Biome** ile — ESLint/Prettier kurma.
- Yanlış bir hücre/formül mantığı = yanlış hakediş. Finansal hesaplarda spec'e birebir uy; emin değilsen sor.

---

## Stack

| Katman | Seçim |
|---|---|
| Framework | Next.js 15 (App Router) + TypeScript |
| ORM / DB | Prisma + PostgreSQL (TR hosting — KVKK; Vargonen/Turkcell) |
| UI | shadcn/ui + Tailwind |
| Data fetching | TanStack Query |
| Auth | NextAuth v5 + custom RBAC |
| Validation | Zod |
| Lint/format | Biome |
| Paket yöneticisi | pnpm |
| Tarih | `date-fns-tz`; uygulama TZ **Europe/Istanbul** (UTC+3, DST yok); DB **UTC** saklar |
| Para | Yalnızca **TL** |
| PDF | react-pdf / Puppeteer (hakediş çıktısı) |
| Excel export | SheetJS (kullanıcı veriyi yine Excel'e çekebilsin) |
| Gantt | `frappe-gantt` / `gantt-task-react` (sürükle-bırak) |
| PWA / offline | Faz 2 (şantiye metraj girişi için). MVP'de gerekmez. |

---

## Multi-tenant (1. günden — sonradan eklemek 3-4 kat pahalı)

- **Shared schema + row-level isolation.** Tenant'a ait her tablo `workspaceId` taşır.
- İzolasyon **Prisma Client Extension** ile zorlanır: hiçbir sorgu `workspaceId` olmadan veri dönmemeli. Bunu altyapı seviyesinde garanti et, her sorguda elle filtrelemeye güvenme.
- Çetin İnşaat = founding tenant / design partner. Pursaklar = ilk gerçek proje.
- Roller: `OWNER` (her şeyi görür/yönetir), `ACCOUNTING` (finansal), `SITE_MANAGER` (yalnız metraj/ilerleme girer).

---

## Finansal hesaplama kuralları (KRİTİK)

Tüm tutarlar **KDV dahil** mantıkla saklanır/hesaplanır. Aşağıdaki oranlar **sözleşme/kalem bazında ayarlanabilir** default'lardır — her işte hepsi olmayabilir (ör. tevkifat yalnız tevkifata tabi işlemlerde; stopaj yalnız yıllara sari inşaat işlerinde).

### İdare hakedişi → net tahsilat
```
base    = KDV'siz hakediş tutarı
KDV     = base × kdvRate            (default 0.20)
brut    = base + KDV
tevkifat = KDV  × tevkifatRate      (default 0.40 = 4/10 — idare KDV'nin %40'ını
                                      doğrudan vergi dairesine yatırır, sana ödemez)
stopaj   = base × stopajRate        (default 0.05 — yıllara sari işlerde; yoksa 0)
damga    = base × damgaRate         (default 0.00948 = binde 9,48)
net      = brut − tevkifat − stopaj − damga
```
Net tahsilat, Gelir → Sapma → İcmal zincirinin tamamına bu değerle akar.

### Vade kademeleri (hem gider hem gelir)
- 12 kademe: **Peşin, 30, 45, 60, 75, 90, 105, 120, 135, 150, 165, 180 gün.**
- Yarım kademeler (**45/75/105/135/165**) bir önceki tam kademeyle **aynı aya yuvarlanır**.
- Her ödeme satırı **Nakit** veya **Çek** olarak ayrışır (çek = vadeli).
- 180 güne kadar nakit akışında **~7 ay sarkma** olur; cash-flow projeksiyonu bunu kapsamalı.

### Sapma
Sapma bir tablo/sayfa **değildir**. Plan vs. gerçekleşen üzerinden **anlık hesaplanan görünüm**dür; saklanmaz, sorguyla üretilir. Kâr marjı eşik altına düşerse uyarı (default eşik %10).

### Metraj / fiyat ayrımı (Excel'den taşınan kritik kural)
- **Planlanan maliyet temeli (baseline) sabit kalmalı**: iş programı tarihleri revize edilince planlanan gider değişmemeli.
- Bunun için planlanan metraj/maliyet **yalnız plan tarihlerine** referans verir, asla revize tarihlere değil.
- Maliyet kalemi: işçilik B.F + malzeme B.F (+ idare), her biri için **ayrı revize tarih ve fiyat** (gün bazlı fiyat geçişi). İşçilik ve malzeme revize tarihleri **bağımsız** çalışır.

---

## Sözlük

- **Hakediş**: dönemsel iş ilerleme faturası
- **Metraj**: iş miktarı (m³, m² vb.)
- **Tevkifat**: KDV'nin idarece doğrudan vergi dairesine yatırılan kısmı
- **Stopaj**: gelir/kurumlar vergisi kesintisi (yıllara sari işler)
- **Damga vergisi**: yasal kesinti (binde 9,48)
- **Vade**: ödeme/tahsilat gün kademesi
- **Çek / Nakit**: ödeme tipi (çek = vadeli)
- **Taşeron**: alt yüklenici
- **İcmal**: özet · **Sapma**: plan vs gerçek farkı

---

## Modüller

1. **Projeler** — çok şantiye, her birinin kendi iş programı
2. **İş Kalemleri** — metraj, birim fiyat (işçilik/malzeme/idare), KDV, revize tarih + fiyat
3. **Gantt** — plan / revize başlangıç / revize bitiş renkli; sürükle-bırak tarih
4. **Hakediş** — aylık otomatik, manuel metraj düzeltme, taşeron/firma bazlı
5. **Nakit Akış** — yüzdelere göre Nakit/Çek dağılımı, ay bazında plan vs gerçek
6. **Çek Yönetimi** — hangi çek, kime, ne zaman, ne kadar — hatırlatma sistemi
7. **Sapma Analizi** — kâr marjı uyarıları
8. **Raporlama** — PDF hakediş, idareye resmi format, Excel/PDF export
9. **Çok kullanıcılı roller**

---

## Önerilen proje yapısı

```
src/
  app/                 # App Router sayfaları (route grupları: (auth), (app))
  components/ui/        # shadcn
  components/           # domain bileşenleri
  lib/
    db.ts               # Prisma client (tenant extension ile)
    auth.ts             # NextAuth v5
    finance/            # hesap motoru: net tahsilat, vade dağılımı, sapma
    tz.ts               # Europe/Istanbul yardımcıları
  server/               # server actions / route handlers
prisma/
  schema.prisma
  seed.ts
```

---

## Fazlar

- **Faz 0 (~1 hafta):** Prisma şeması, design system, proje iskeleti, Claude Code kurulumu (sub-agent + slash command + settings).
- **Faz 1 (3–6 hafta):** MVP — Projeler, İş Kalemleri + hesap motoru, Gantt, Hakediş, Nakit Akış, temel raporlama → **Pursaklar'da canlı kullanım**.
- **Faz 2:** Çek hatırlatma, Sapma uyarıları, PWA offline (şantiye metraj girişi), PDF hakediş çıktısı, 2. pilot müşteri.
- **Faz 3:** SaaS ürünleştirme — landing, demo video, ilk 5–10 müşteri (Ankara inşaat sektörü).
