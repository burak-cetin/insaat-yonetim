# Faz-0 — Proje İskeleti & Temel Kurulum

**Hedef (kabul kriteri):** `pnpm dev` ile uygulama açılıyor · `.env`'deki OWNER ile giriş yapılıp boş dashboard görülüyor · tenant izolasyonu DB katmanında tek noktadan garanti · `pnpm biome check` ve `pnpm test` geçiyor · migration temiz.

**Kullanım:** Aşağıdaki görevleri **sırayla** Claude Code'a ver. Her görevden sonra çalıştığını doğrula, sonra **commit at — push etme** (push senin sorumluluğun). Önce `CLAUDE.md`'yi okuttur.

**Not:** Komutlar/sürümler referanstır. Güncel stable sürüm veya değişmiş CLI sözdizimi varsa Claude Code uyarlasın.

---

## Görev 0.1 — Proje iskeleti + Biome + klasör yapısı

```
CLAUDE.md'yi oku. Bu repoda Next.js 15 projesini kur: TypeScript, Tailwind,
App Router, src/ dizini, import alias @/*, paket yöneticisi pnpm.
ESLint EKLEME — Biome kullanacağız.

Sonra:
- Biome'u kur ve yapılandır (lint + format). package.json'a script'ler:
  "lint": biome check, "format": biome format --write.
- .env.example oluştur: DATABASE_URL, AUTH_SECRET, SEED_OWNER_EMAIL,
  SEED_OWNER_PASSWORD, SEED_WORKSPACE_NAME.
- CLAUDE.md'deki klasör yapısını kur: src/app içinde (auth) ve (app) route
  grupları, src/lib, src/lib/finance, src/components, src/components/ui,
  src/server, prisma/.
- pnpm dev ile boot ettiğini doğrula.
Sürüm/CLI değişmişse güncel stable'a uyarla.
```

---

## Görev 0.2 — Prisma client + multi-tenant izolasyon extension

```
prisma/schema.prisma zaten yerinde ve commit edildi. Prisma'yı kur
(@prisma/client + prisma dev dep).

src/lib/db.ts:
- Temel PrismaClient (global/auth sorguları için export: `db`).
- Bir tenant-scoped Prisma Client Extension yaz ve `forWorkspace(workspaceId)`
  factory'si ile export et. Extension, workspaceId taşıyan TÜM tenant
  modellerine (Membership, Project, WorkItem, PriceRevision, PaymentTerm,
  Subcontractor, Contract, Deduction, Payment, Hakedis, Collection, Cheque)
  yapılan sorgularda izolasyonu otomatik uygulasın:
    * read (findFirst/findMany/count/aggregate/groupBy) + update/delete/
      updateMany/deleteMany/upsert → where'e workspaceId enjekte et,
    * create/createMany → data'ya workspaceId set et,
    * upsert → create tarafına da workspaceId.
  Modül kodu workspaceId'yi ASLA elle filtrelemek zorunda kalmasın.
- User/Account/Session/VerificationToken/Workspace tenant-scoped DEĞİL —
  bunlara dokunma.

DİKKAT: findUnique/findUniqueOrThrow, Prisma'da where olarak yalnız unique
alan kabul eder; bu yüzden workspaceId enjekte edilemez. Bu çağrıları scoped
client'ta findFirst semantiğine çevir (veya scoped client'ta findUnique'i
yasakla). İzolasyonun her operasyonda tutarlı çalıştığını gösteren küçük bir
test yaz.

DATABASE_URL'i .env'e gir (lokal Postgres, depyon yerine yeni bir db adı),
`pnpm prisma migrate dev --name init` ve `pnpm prisma generate` çalıştır.
Migration'ın temiz uygulandığını doğrula.
```

---

## Görev 0.3 — Auth (NextAuth v5 / Auth.js, Credentials + JWT)

```
NextAuth v5 (Auth.js) kur. Credentials provider + JWT session stratejisi
(adapter tablolarına gerek yok; Session/Account kullanmıyoruz).

src/lib/auth.ts:
- email + password ile giriş; passwordHash'i bcrypt veya argon2 ile doğrula.
- JWT'ye userId koy.
- Aktif session'dan kullanıcıyı + membership'ini (workspaceId, role) çözen
  bir helper (`getSessionUser()`).

UI/koruma:
- (auth)/login sayfası: shadcn form, Zod doğrulama, server action ile giriş.
- (app) route grubu korunaklı olsun (middleware veya layout guard); giriş
  yoksa /login'e yönlendir.
- Şimdilik tek-workspace: kullanıcının ilk membership'i aktif workspace.

Paket adı/sürüm (next-auth@beta vs stable) için güncel olanı kullan.
```

---

## Görev 0.4 — Tenant context + scoped client bağlama

```
src/lib/tenant.ts:
- Aktif workspaceId'yi session→membership üzerinden çözen ve
  forWorkspace(workspaceId) döndüren request-scoped bir helper:
  `getTenantDb()`.
- Basit yetki guard'ı: `requireRole(...roles)` (OWNER/ACCOUNTING/SITE_MANAGER);
  yetersizse 403.

Kural: (app) altındaki tüm server action / route handler veriye getTenantDb()
ile erişir. Global `db` yalnız auth + workspace işleri için kullanılır.
Bunu CLAUDE.md'ye kısa bir "veri erişim kuralı" notu olarak da ekle.
```

---

## Görev 0.5 — Seed (env-driven, minimal)

```
prisma/seed.ts: SADECE .env'den okuyarak idempotent bir minimal seed:
- 1 Workspace (SEED_WORKSPACE_NAME'den; slug üret),
- 1 OWNER User (SEED_OWNER_EMAIL + SEED_OWNER_PASSWORD → hash),
- ikisi arası OWNER Membership.
Upsert kullan; tekrar çalışınca çoğaltmasın.
HİÇBİR proje / iş kalemi / finansal veri EKLEME — her şey boş başlasın.

package.json'a prisma seed config + "db:seed" script'i ekle.
Çalıştır; OWNER kullanıcı + workspace + membership oluştuğunu doğrula.
```

---

## Görev 0.6 — Design system + uygulama kabuğu

```
shadcn/ui'yi kur (güncel CLI). Sade, profesyonel, nötr (koyu moda uyumlu) bir
tema; Türkçe içerik ve TL biçimlendirme için uygun. Jenerik/şablon görünmesin.

(app) kabuğu:
- Sol sidebar navigasyonu, CLAUDE.md'deki modüllerle: Projeler, İş Kalemleri,
  Gantt, Hakediş, Nakit Akış, Çek Yönetimi, Sapma, Raporlama, Ayarlar.
- Her modül için şimdilik boş placeholder sayfa (başlık + "yakında").
- Üst bar: aktif workspace adı + kullanıcı menüsü (çıkış).
- TanStack Query provider'ı kur (app kabuğunu sar).
Giriş sonrası boş bir dashboard'a düşülsün.
```

---

## Görev 0.7 — Finans katmanı (motor + testler)

```
src/lib/finance/ içinde, CLAUDE.md'deki formüllere BİREBİR uyan saf (pure),
tipli fonksiyonlar + Vitest birim testleri yaz. Para hesapları Decimal-güvenli
olsun (decimal.js veya Prisma.Decimal; FLOAT KULLANMA).

Fonksiyonlar:
- idareNet({ base, vatRate, tevkifatRate, stopajRate, damgaRate }):
    gross = base*(1+vatRate)
    net   = gross − base*vatRate*tevkifatRate − base*stopajRate − base*damgaRate
- workItemFinancials(workItem, priceRevisions, asOfDate?):
    revizyonları effectiveDate'e göre uygulayıp costTotalInclVat, idareGross,
    idareNet, profit (=idareNet−costTotalInclVat), margin (=profit/idareNet) döndür.
- distributePayments({ amount, terms }):
    vade kademelerine (0/30/45/.../180) göre ay-ay Nakit/Çek dağılımı;
    yarım kademeleri (45/75/105/135/165) bir önceki TAM kademenin ayına yuvarla;
    180 güne kadar sarkmayı destekle.
- contractRemaining({ baseAmount, vatRate, payments }):
    gross − Σ(payments).  (bilgi amaçlı deductions'tan BAĞIMSIZ)

Testler: en az şu örneği doğrula —
base=1.000.000, vatRate=0.20, tevkifatRate=0.40, stopajRate=0.05,
damgaRate=0.00948 → KDV=200.000, gross=1.200.000,
tevkifat=80.000, stopaj=50.000, damga=9.480, net=1.060.520.
Ayrıca yarım-kademe yuvarlaması için bir distributePayments testi.
Vitest kur; `pnpm test` geçsin.
```

---

## Görev 0.8 — Doğrulama / kabul

```
Tümünü doğrula:
- pnpm dev → uygulama açılıyor,
- .env'deki OWNER ile giriş → boş dashboard,
- pnpm biome check temiz, pnpm test geçiyor,
- migration uygulanmış.
Sonra commit at (push etme).
```
