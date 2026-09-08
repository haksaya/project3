# Modül 9 — Arama, Filtreleme, Sayfalama

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Öğrenci listesine arama, filtre ve sayfalama ekleyeceğiz.

**Ana kavram:** `IQueryable` ile sorguyu **parça parça inşa etmek.**

---

## 📖 Kavram: IQueryable ile sorgu inşası

### ADO.NET'te ne yapıyorduk?

```csharp
string kosullar = " WHERE o.is_active = '1' ";

if (!string.IsNullOrWhiteSpace(arama))
    kosullar += " AND (o.ogrenci_ad LIKE @arama OR ...) ";

if (bolumId.HasValue)
    kosullar += " AND o.bolum_id = @bolumId ";

string sql = "SELECT ... FROM ogrenci o ..." + kosullar + " ORDER BY ...";
```

**Metin birleştiriyorduk.** Yazım hatası ancak çalışma zamanında ortaya çıkıyordu.

### EF'te

```csharp
IQueryable<Ogrenci> sorgu = _db.Ogrenciler;

if (!string.IsNullOrWhiteSpace(arama))
    sorgu = sorgu.Where(o => o.OgrenciAd.Contains(arama));

if (bolumId.HasValue)
    sorgu = sorgu.Where(o => o.BolumId == bolumId.Value);

var liste = await sorgu.ToListAsync();
```

**Kod birleştiriyoruz.** Yazım hatası **derleme zamanında** yakalanıyor.

> ⭐ **Kritik nokta:** Bu `if` bloklarının hiçbiri veritabanına gitmez. Sadece son satırdaki `ToListAsync()` gider ve **tüm koşulları içeren tek bir SQL** çalıştırır.

Modül 3'teki ertelenmiş çalıştırma (deferred execution) kavramının pratik karşılığı budur.

---

## 📖 LIKE karşılığı

| LINQ | SQL | Bulur |
|---|---|---|
| `.Contains("ays")` | `LIKE N'%ays%'` | İçeren |
| `.StartsWith("ays")` | `LIKE N'ays%'` | Başlayan |
| `.EndsWith("ays")` | `LIKE N'%ays'` | Biten |

```csharp
sorgu = sorgu.Where(o => o.OgrenciAd.Contains(arama));
```

EF bunu otomatik olarak parametreli `LIKE`'a çevirir:
```sql
WHERE [o].[OgrenciAd] LIKE N'%' + @__arama_0 + N'%'
```

> ⭐ **SQL injection yok.** Değer parametre olarak gidiyor. ADO.NET'te bunu elle sağlamak zorundaydık.

### ⚠️ Büyük/küçük harf duyarlılığı

SQL Server'ın varsayılan harmanlaması (collation) büyük/küçük harf **duyarsızdır**. Yani `Contains("ayse")` → "Ayşe"yi de bulur.

`ToLower()` yazmaya **gerek yok** ve yazmak zararlı:
```csharp
// ❌ Gereksiz — üstelik indeks kullanımını engeller
.Where(o => o.OgrenciAd.ToLower().Contains(arama.ToLower()))

// ✅
.Where(o => o.OgrenciAd.Contains(arama))
```

---

## 📖 Sayfalama: Skip ve Take

```csharp
sorgu.Skip(20).Take(10)
```

SQL karşılığı:
```sql
ORDER BY ... OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY
```

> ADO.NET'te bunu elle yazıyorduk. Aynı SQL, tek satır LINQ.

| Sayfa | `Skip` | `Take` |
|---|---|---|
| 1 | 0 | 10 |
| 2 | 10 | 10 |
| 3 | 20 | 10 |

Formül: `Skip((sayfa - 1) * sayfaBoyutu)`

⚠️ **`Skip` kullanmak için `OrderBy` zorunludur.** Sıralama olmadan "ilk 20"nin bir anlamı yok; EF de SQL Server da buna izin vermez.

---

## ⌨️ Adım 1: Controller

`OgrenciController.Index` metodunu değiştir:

```csharp
    // ══════════════════════════════════════════════════════
    //  LİSTELEME + ARAMA + FİLTRE + SAYFALAMA
    //  GET: /Ogrenci?arama=ayse&bolumId=1&sinif=3&sayfa=2
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Index(
        string? arama,
        long? bolumId,
        int? sinif,
        int sayfa = 1)
    {
        const int sayfaBoyutu = 10;

        // Kullanıcı adres çubuğuna sayfa=0 veya sayfa=-5 yazabilir
        if (sayfa < 1) sayfa = 1;

        // ══════════════════════════════════════════════════
        //  1) SORGUYU PARÇA PARÇA KUR
        //
        //  ⭐ Bu satırların HİÇBİRİ veritabanına gitmez.
        //     Sadece "ne isteyeceğimizin tarifi" hazırlanır.
        // ══════════════════════════════════════════════════
        IQueryable<Ogrenci> sorgu = _db.Ogrenciler
            .Include(o => o.Bolum)
                .ThenInclude(b => b!.Fakulte);

        if (!string.IsNullOrWhiteSpace(arama))
        {
            string temizArama = arama.Trim();

            // Ad, soyad, e-posta veya TC içinde ara
            sorgu = sorgu.Where(o =>
                o.OgrenciAd.Contains(temizArama) ||
                o.OgrenciSoyad.Contains(temizArama) ||
                o.OgrenciEposta.Contains(temizArama) ||
                o.OgrenciTc.Contains(temizArama));
        }

        if (bolumId.HasValue && bolumId.Value > 0)
            sorgu = sorgu.Where(o => o.BolumId == bolumId.Value);

        if (sinif.HasValue && sinif.Value > 0)
            sorgu = sorgu.Where(o => o.OgrenciSinif == sinif.Value);

        // ══════════════════════════════════════════════════
        //  2) TOPLAM KAYIT SAYISI
        //
        //  ⚠️ Bu satır veritabanına GİDER (COUNT sorgusu).
        //     Sayfa sayısını hesaplamak için gerekli — sayfalanmış
        //     sonuçtan toplam sayıyı çıkaramayız.
        //
        //  Not: Include'lar COUNT sorgusuna dâhil edilmez,
        //       EF gereksiz JOIN'leri kendisi atar.
        // ══════════════════════════════════════════════════
        int toplamKayit = await sorgu.CountAsync();

        // ══════════════════════════════════════════════════
        //  3) SAYFAYI GETİR
        //
        //  ⚠️ SIRA ÖNEMLİ: OrderBy → Skip → Take
        // ══════════════════════════════════════════════════
        var liste = await sorgu
            .OrderBy(o => o.OgrenciAd)
            .ThenBy(o => o.OgrenciSoyad)
            .Skip((sayfa - 1) * sayfaBoyutu)
            .Take(sayfaBoyutu)
            .ToListAsync();

        // ══════════════════════════════════════════════════
        //  4) SAYFALAMA BİLGİLERİ
        //
        //  (double) dönüşümü ŞART:
        //  47 / 10 = 4  (tam sayı bölmesi)  → 5. sayfa kaybolur
        //  47.0 / 10 = 4.7 → Ceiling → 5    ✅
        // ══════════════════════════════════════════════════
        ViewBag.Sayfa       = sayfa;
        ViewBag.ToplamSayfa = (int)Math.Ceiling((double)toplamKayit / sayfaBoyutu);
        ViewBag.ToplamKayit = toplamKayit;

        // Filtre değerlerini geri gönder — form dolu kalsın
        ViewBag.Arama         = arama;
        ViewBag.SeciliBolum   = bolumId;
        ViewBag.SeciliSinif   = sinif;

        await BolumListesiniHazirlaAsync(bolumId);

        return View(liste);
    }
```

> **Filtre değerlerini geri göndermezsen** kullanıcı arama yapar, sonucu görür ama arama kutusu boşalmış olur. "Ne aramıştım ben?"

---

## ⌨️ Adım 2: Arama formu

`Views/Ogrenci/Index.cshtml`'in en üstüne:

```html
<div class="card border-0 shadow-sm mb-3">
    <div class="card-body">

        @* ⭐ method="get" — POST DEĞİL!
           Sebep: arama sonuçları paylaşılabilir ve yer imine eklenebilir olmalı.
           Adres çubuğunda /Ogrenci?arama=ayse&sinif=3 görünür.
           Genel kural: arama = GET, kaydetme = POST *@
        <form method="get" asp-action="Index" class="row g-2 align-items-end">

            <div class="col-md-4">
                <label class="form-label small text-muted">Ara</label>
                <input type="text" name="arama" value="@ViewBag.Arama"
                       class="form-control form-control-sm"
                       placeholder="Ad, soyad, e-posta veya TC" />
            </div>

            <div class="col-md-3">
                <label class="form-label small text-muted">Bölüm</label>
                <select name="bolumId" asp-items="ViewBag.Bolumler"
                        class="form-select form-select-sm">
                    <option value="">Tüm bölümler</option>
                </select>
            </div>

            <div class="col-md-2">
                <label class="form-label small text-muted">Sınıf</label>
                <select name="sinif" class="form-select form-select-sm">
                    <option value="">Tümü</option>
                    @for (int i = 1; i <= 6; i++)
                    {
                        <option value="@i" selected="@(ViewBag.SeciliSinif as int? == i)">
                            @i. sınıf
                        </option>
                    }
                </select>
            </div>

            <div class="col-md-3">
                <button type="submit" class="btn btn-primary btn-sm">
                    <i class="bi bi-search"></i> Ara
                </button>
                <a asp-action="Index" class="btn btn-outline-secondary btn-sm">Temizle</a>
            </div>

        </form>
    </div>
</div>
```

---

## ⌨️ Adım 3: Sayfalama bağlantıları

Tablonun **altına**:

```html
<div class="d-flex justify-content-between align-items-center mt-3">

    <div class="text-muted small">
        Toplam <strong>@ViewBag.ToplamKayit</strong> kayıt
        @if (ViewBag.ToplamSayfa > 0)
        {
            <span>| Sayfa @ViewBag.Sayfa / @ViewBag.ToplamSayfa</span>
        }
    </div>

    @if (ViewBag.ToplamSayfa > 1)
    {
        <nav>
            <ul class="pagination pagination-sm mb-0">

                @* ── Önceki ── *@
                <li class="page-item @(ViewBag.Sayfa == 1 ? "disabled" : "")">
                    <a class="page-link"
                       asp-action="Index"
                       asp-route-arama="@ViewBag.Arama"
                       asp-route-bolumId="@ViewBag.SeciliBolum"
                       asp-route-sinif="@ViewBag.SeciliSinif"
                       asp-route-sayfa="@(ViewBag.Sayfa - 1)">‹</a>
                </li>

                @* ── Sayfa numaraları ── *@
                @for (int i = 1; i <= ViewBag.ToplamSayfa; i++)
                {
                    <li class="page-item @(i == ViewBag.Sayfa ? "active" : "")">
                        <a class="page-link"
                           asp-action="Index"
                           asp-route-arama="@ViewBag.Arama"
                           asp-route-bolumId="@ViewBag.SeciliBolum"
                           asp-route-sinif="@ViewBag.SeciliSinif"
                           asp-route-sayfa="@i">@i</a>
                    </li>
                }

                @* ── Sonraki ── *@
                <li class="page-item @(ViewBag.Sayfa == ViewBag.ToplamSayfa ? "disabled" : "")">
                    <a class="page-link"
                       asp-action="Index"
                       asp-route-arama="@ViewBag.Arama"
                       asp-route-bolumId="@ViewBag.SeciliBolum"
                       asp-route-sinif="@ViewBag.SeciliSinif"
                       asp-route-sayfa="@(ViewBag.Sayfa + 1)">›</a>
                </li>

            </ul>
        </nav>
    }
</div>
```

### ⚠️ En kritik nokta

Her sayfalama bağlantısında **arama ve filtre değerlerini taşımak zorundayız:**

```html
asp-route-arama="@ViewBag.Arama"
asp-route-bolumId="@ViewBag.SeciliBolum"
asp-route-sinif="@ViewBag.SeciliSinif"
```

Bunları koymazsan: kullanıcı "Ayşe" arar, 2. sayfaya geçer, **arama sıfırlanır ve tüm öğrenciler gelir.**

> **Deney:** Bu satırları sil, arama yapıp 2. sayfaya geçmelerini iste. Sorunu kendileri yaşasın. Sonra geri ekle.

---

## ⌨️ Adım 4: Boş sonuç mesajı

Tabloyu saran `if` bloğunu güncelle:

```html
@if (Model.Count == 0)
{
    @{
        bool filtreVar = !string.IsNullOrEmpty(ViewBag.Arama as string)
                      || ViewBag.SeciliBolum != null
                      || ViewBag.SeciliSinif != null;
    }

    <div class="text-center text-muted py-5">
        @if (filtreVar)
        {
            <i class="bi bi-search fs-1 d-block mb-2"></i>
            <p class="mb-3">Aramanıza uyan öğrenci bulunamadı.</p>
            <a asp-action="Index" class="btn btn-sm btn-outline-primary">
                Filtreleri temizle
            </a>
        }
        else
        {
            <i class="bi bi-inbox fs-1 d-block mb-2"></i>
            <p class="mb-3">Henüz öğrenci yok.</p>
            <a asp-action="Create" class="btn btn-sm btn-primary">İlk öğrenciyi ekle</a>
        }
    </div>
}
```

> **Arayüz yazısı dersi:** "Sonuç bulunamadı" iki farklı duruma aynı cevabı verir. Ama kullanıcının yapması gereken şey farklı: birinde filtreyi temizlemeli, diğerinde kayıt eklemeli. **İyi arayüz, kullanıcıya bir sonraki adımı söyler.**

---

## ▶️ Üretilen SQL'i incele

Arama yap (`?arama=ay&sinif=3&sayfa=2`) ve konsola bak. **İki sorgu** var:

### 1) Sayım sorgusu

```sql
SELECT COUNT(*)
FROM [Ogrenciler] AS [o]
INNER JOIN [Bolumler] AS [b] ON [o].[BolumId] = [b].[BolumId]
WHERE [o].[AktifMi] = CAST(1 AS bit) AND [b].[AktifMi] = CAST(1 AS bit)
  AND ([o].[OgrenciAd] LIKE N'%' + @__temizArama_0 + N'%'
       OR [o].[OgrenciSoyad] LIKE N'%' + @__temizArama_0 + N'%'
       OR ...)
  AND [o].[OgrenciSinif] = @__sinif_1
```

### 2) Veri sorgusu

```sql
SELECT [o].[OgrenciId], ..., [b].[BolumAdi], ..., [f].[FakulteAd], ...
FROM [Ogrenciler] AS [o]
INNER JOIN [Bolumler] AS [b] ON ...
INNER JOIN [Fakulteler] AS [f] ON ...
WHERE ... (aynı koşullar)
ORDER BY [o].[OgrenciAd], [o].[OgrenciSoyad]
OFFSET @__p_2 ROWS FETCH NEXT @__p_3 ROWS ONLY
```

### Öğrenciyle beraber inceleyin

1. **"Değerler nerede?"** → `@__temizArama_0` gibi **parametreler**. Metin içine gömülmemiş — SQL injection yok.
2. **"Neden iki sorgu?"** → Biri toplam sayı için, biri o sayfadaki kayıtlar için.
3. **"COUNT sorgusunda kaç JOIN var?"** → Sadece bir tane (Bolumler). EF, `Fakulteler` JOIN'ini gereksiz olduğu için **atmış**. Sayım için fakülte bilgisine gerek yok.
4. **"`OFFSET/FETCH` tanıdık mı?"** → ADO.NET sürümünde elle yazmıştık.

> **Ders çıkarımı:** *"EF sadece SQL yazmıyor, gereksiz kısımları da ayıklıyor. Ama bunu her zaman doğru yapacağını varsaymayın — konsola bakın."*

---

## ▶️ Test senaryosu

| Test | Beklenen |
|---|---|
| Arama kutusuna isim yaz | Sadece o kayıtlar |
| Bölüm seç | Sadece o bölüm |
| Arama + bölüm + sınıf | Üçü de uygulanmış |
| Aramadan sonra 2. sayfa | **Arama korunuyor** |
| Olmayan bir şey ara | "Filtreleri temizle" butonu |
| Adres çubuğuna `?sinif=3` | Doğrudan filtreli liste |
| Adres çubuğuna `?sayfa=999` | Boş liste, **çökme yok** |
| Adres çubuğuna `?sayfa=0` | 1. sayfa gösteriliyor |
| Temizle | Tüm filtreler sıfır |
| Konsoldaki SQL | İki sorgu, parametreli, `OFFSET/FETCH` |

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `Skip` hata veriyor | `OrderBy` yok | Önce sırala |
| Sayfa değişince arama sıfırlanıyor | `asp-route-*` eksik | Tüm filtreleri her bağlantıya ekle |
| Sayfa sayısı hep 1 | Tam sayı bölmesi | `(double)` dönüşümü ekle |
| 2. sayfada aynı kayıtlar | `Skip` hesabı yanlış | `(sayfa - 1) * sayfaBoyutu` |
| `sayfa=0` → boş sayfa | Kontrol yok | `if (sayfa < 1) sayfa = 1;` |
| Filtre seçili kalmıyor | ViewBag'e geri gönderilmemiş | Controller'da ata |
| `The LINQ expression could not be translated` | `Contains` içinde karmaşık ifade | Değişkene al, sonra kullan |
| Arama çok yavaş | `LIKE '%...%'` indeks kullanamaz | Aşağıya bak |

---

## 💬 Tartışma: `%...%` neden yavaş?

**Sınıfa sor:** *"10.000 öğrenci varsa `Contains("ay")` araması ne kadar sürer?"*

```sql
WHERE OgrenciAd LIKE N'%ay%'
```

Baştaki `%` yüzünden veritabanı **indeksi kullanamaz.** Her satıra tek tek bakmak zorunda kalır (tam tablo taraması).

```sql
WHERE OgrenciAd LIKE N'ay%'    -- ✅ indeks kullanılabilir
WHERE OgrenciAd LIKE N'%ay%'   -- ❌ kullanılamaz
```

Çözüm yolları:
- `StartsWith` kullanmak (kullanıcı deneyimi düşer)
- Full-text search kullanmak (SQL Server özelliği)
- Elasticsearch gibi arama motoru (büyük projeler)

> Ders projesinde sorun değil ama öğrenci bunun bir maliyeti olduğunu **bilmeli.**

---

## ✏️ Öğrenci alıştırması

1. Arama + filtre + sayfalamayı tamamla, on test senaryosunu geçir.
2. Aynı yapıyı `Akademisyen` listesine ekle.
3. Sayfa boyutunu kullanıcı seçebilsin (10 / 25 / 50).
4. Tablo başlıklarına tıklanınca sıralansın. İkinci tıkta ters sırala.
   İpucu:
   ```csharp
   sorgu = siralama switch
   {
       "ad_desc"  => sorgu.OrderByDescending(o => o.OgrenciAd),
       "sinif"    => sorgu.OrderBy(o => o.OgrenciSinif),
       "tarih"    => sorgu.OrderByDescending(o => o.CreatedDate),
       _          => sorgu.OrderBy(o => o.OgrenciAd)
   };
   ```
   > ⚠️ Burada da **beyaz liste** kullanıyoruz — kullanıcının yazdığı metni doğrudan kullanmıyoruz. ADO.NET'te bu SQL injection'a karşı zorunluydu; EF'te teknik olarak zorunlu değil ama yine de doğru tasarım.
5. **Düşünme soruları:**
   - Neden iki sorgu (sayım + veri) çalıştırıyoruz? Tek sorguda yapılabilir mi?
   - 100 sayfa varsa 100 numara göstermek mantıklı mı? Nasıl iyileştirirsin?
   - Filtreleme neden GET, kaydetme neden POST?

---

👉 Sonraki: [`11-login.md`](11-login.md)
