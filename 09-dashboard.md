# Modül 8 — Dashboard (LINQ ile İstatistik)

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Ana panele gerçek istatistikler koyacağız. **Yeni kavramlar:**

1. `GroupBy` — gruplayarak sayma
2. Projeksiyon — sadece gereken sütunları çekme
3. `CountAsync` ile koşullu sayma

---

## 📖 Kavram: Projeksiyon (Select)

Bu, EF'in performans açısından en önemli özelliği.

```csharp
// ❌ TÜM sütunları çeker — adres, telefon, TC... hepsi
var ogrenciler = await _db.Ogrenciler.ToListAsync();
int sayi = ogrenciler.Count;

// ✅ Sadece sayıyı çeker
int sayi = await _db.Ogrenciler.CountAsync();
```

Üretilen SQL'ler:

```sql
-- Birincisi
SELECT [o].[OgrenciId], [o].[OgrenciAd], [o].[OgrenciAdres], ... (13 sütun)
FROM [Ogrenciler] AS [o] WHERE ...

-- İkincisi
SELECT COUNT(*) FROM [Ogrenciler] AS [o] WHERE ...
```

> **Sınıfa sor:** *"10.000 öğrenci varsa, birinci kod ne kadar veri taşır?"*
> Yaklaşık 10.000 × 500 bayt = 5 MB. İkincisi 4 bayt.

### Aynı fikir listeler için

```csharp
// ❌ Bölümün tüm alanlarını çekiyor
var bolumler = await _db.Bolumler.Include(b => b.Fakulte).ToListAsync();

// ✅ Sadece ekranda göstereceğimiz üç değer
var bolumler = await _db.Bolumler
    .Select(b => new BolumDagilim
    {
        BolumAdi = b.BolumAdi,
        FakulteAd = b.Fakulte!.FakulteAd,
        OgrenciSayisi = b.Ogrenciler.Count()
    })
    .ToListAsync();
```

> ⭐ **`Select` içinde `Include` gerekmez.** `b.Fakulte!.FakulteAd` yazdığınızda EF gerekli JOIN'i zaten üretir. `Include` yalnızca tam nesneyi istediğinizde gerekir.

---

## ⌨️ Adım 1: ViewModel

`Models/DashboardViewModel.cs`:

```csharp
namespace UBYS.Models;

/// <summary>
/// Dashboard ekranının taşıyıcısı.
///
/// ⭐ Model ile ViewModel farkı:
///    Model     = bir TABLONUN karşılığı  (Fakulte, Ogrenci)
///    ViewModel = bir EKRANIN ihtiyacı    (bu sınıf)
///
/// Bu sınıfın DbSet'i YOK, tablosu YOK. Sadece veri taşır.
/// </summary>
public class DashboardViewModel
{
    // Üst kartlar
    public int FakulteSayisi { get; set; }
    public int BolumSayisi { get; set; }
    public int OgrenciSayisi { get; set; }
    public int AkademisyenSayisi { get; set; }

    // = new()  →  boş başlasın, null olmasın (view'da çökmesin)
    public List<BolumDagilim> BolumDagilimlari { get; set; } = new();
    public List<SinifDagilim> SinifDagilimlari { get; set; } = new();
    public List<CinsiyetDagilim> CinsiyetDagilimlari { get; set; } = new();
    public List<SonOgrenci> SonEklenenOgrenciler { get; set; } = new();
}

public class BolumDagilim
{
    public string BolumAdi { get; set; } = "";
    public string FakulteAd { get; set; } = "";
    public int OgrenciSayisi { get; set; }
    public int AkademisyenSayisi { get; set; }
}

public class SinifDagilim
{
    public int Sinif { get; set; }
    public int OgrenciSayisi { get; set; }
}

public class CinsiyetDagilim
{
    public string Cinsiyet { get; set; } = "";
    public int OgrenciSayisi { get; set; }
}

public class SonOgrenci
{
    public long OgrenciId { get; set; }
    public string TamAd { get; set; } = "";
    public string BolumAdi { get; set; } = "";
    public int Sinif { get; set; }
    public DateTime CreatedDate { get; set; }
}
```

---

## ⌨️ Adım 2: Controller

`Controllers/HomeController.cs`:

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class HomeController : Controller
{
    private readonly UbysDbContext _db;

    public HomeController(UbysDbContext db)
    {
        _db = db;
    }

    public async Task<IActionResult> Index()
    {
        var model = new DashboardViewModel();

        // ══════════════════════════════════════════════════
        //  1) SAYILAR
        //
        //  CountAsync → SQL'de COUNT(*) olur, veri taşınmaz.
        //  Query filter sayesinde WHERE AktifMi = 1 otomatik eklenir.
        // ══════════════════════════════════════════════════
        model.FakulteSayisi     = await _db.Fakulteler.CountAsync();
        model.BolumSayisi       = await _db.Bolumler.CountAsync();
        model.OgrenciSayisi     = await _db.Ogrenciler.CountAsync();
        model.AkademisyenSayisi = await _db.Akademisyenler.CountAsync();

        // ══════════════════════════════════════════════════
        //  2) BÖLÜM DAĞILIMI
        //
        //  ⭐ Select içinde alt sorgu:
        //     b.Ogrenciler.Count() → SQL'de scalar subquery olur.
        //     Include'a GEREK YOK — EF gereken JOIN'i kendisi üretir.
        // ══════════════════════════════════════════════════
        model.BolumDagilimlari = await _db.Bolumler
            .Select(b => new BolumDagilim
            {
                BolumAdi          = b.BolumAdi,
                FakulteAd         = b.Fakulte!.FakulteAd,
                OgrenciSayisi     = b.Ogrenciler.Count(),
                AkademisyenSayisi = b.Akademisyenler.Count()
            })
            .OrderByDescending(x => x.OgrenciSayisi)
            .ThenBy(x => x.BolumAdi)
            .ToListAsync();

        // ══════════════════════════════════════════════════
        //  3) SINIF DAĞILIMI — GroupBy
        //
        //  SQL karşılığı:
        //     SELECT ogrenci_sinif, COUNT(*)
        //     FROM Ogrenciler GROUP BY ogrenci_sinif
        // ══════════════════════════════════════════════════
        model.SinifDagilimlari = await _db.Ogrenciler
            .GroupBy(o => o.OgrenciSinif)        // neye göre grupla
            .Select(g => new SinifDagilim
            {
                Sinif         = g.Key,           // ⭐ g.Key = gruplanan değer
                OgrenciSayisi = g.Count()        // o gruptaki satır sayısı
            })
            .OrderBy(x => x.Sinif)
            .ToListAsync();

        // ══════════════════════════════════════════════════
        //  4) CİNSİYET DAĞILIMI — yine GroupBy
        // ══════════════════════════════════════════════════
        model.CinsiyetDagilimlari = await _db.Ogrenciler
            .GroupBy(o => o.OgrenciCinsiyet)
            .Select(g => new CinsiyetDagilim
            {
                Cinsiyet      = g.Key,
                OgrenciSayisi = g.Count()
            })
            .OrderByDescending(x => x.OgrenciSayisi)
            .ToListAsync();

        // ══════════════════════════════════════════════════
        //  5) SON EKLENEN ÖĞRENCİLER
        //
        //  Take(5) → SQL'de TOP(5) olur.
        //  ⚠️ SIRA ÖNEMLİ: OrderBy önce, Take sonra.
        //     Ters yazsaydık rastgele 5 kayıt alıp onları sıralardık.
        // ══════════════════════════════════════════════════
        model.SonEklenenOgrenciler = await _db.Ogrenciler
            .OrderByDescending(o => o.CreatedDate)
            .Take(5)
            .Select(o => new SonOgrenci
            {
                OgrenciId   = o.OgrenciId,
                TamAd       = o.OgrenciAd + " " + o.OgrenciSoyad,
                BolumAdi    = o.Bolum!.BolumAdi,
                Sinif       = o.OgrenciSinif,
                CreatedDate = o.CreatedDate
            })
            .ToListAsync();

        return View(model);
    }

    [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
    public IActionResult Error()
    {
        ViewBag.HataKodu = Activity.Current?.Id ?? HttpContext.TraceIdentifier;
        return View();
    }
}
```

### ⚠️ `TamAd` neden `Select` içinde elle birleştirildi?

`Ogrenci` modelinde `TamAd` hesaplanan bir özellik:

```csharp
[NotMapped]
public string TamAd => OgrenciAd + " " + OgrenciSoyad;
```

Ama `Select` içinde `o.TamAd` **yazamayız.** Neden?

EF, `Select` içindeki ifadeyi **SQL'e çevirmek** zorunda. `TamAd` bir C# özelliği; SQL'de karşılığı yok. Hata:

```
The LINQ expression could not be translated.
```

**Çözüm:** SQL'e çevrilebilecek şekilde yazmak:
```csharp
TamAd = o.OgrenciAd + " " + o.OgrenciSoyad
```

EF bunu `[OgrenciAd] + N' ' + [OgrenciSoyad]` olarak çevirir.

> ⭐ **Genel kural:** `Where`, `Select`, `OrderBy` içinde yazdığınız her şey SQL'e çevrilebilmeli. Kendi yazdığınız metotlar, `[NotMapped]` özellikler, karmaşık C# kodu çevrilemez.
>
> Modül 11'de bu hatanın çözüm yollarını göreceğiz.

### 📖 `GroupBy` — `g.Key` nedir?

```csharp
.GroupBy(o => o.OgrenciSinif)     // 1. sınıflar, 2. sınıflar, 3. sınıflar...
.Select(g => new {
    g.Key,                        // ⭐ o grubun değeri: 1, 2, 3...
    Adet = g.Count()              // o gruptaki öğrenci sayısı
})
```

Tahtaya çiz:

```
ÖNCESİ                      GRUPLAMA SONRASI
────────                    ─────────────────
Ayşe    3. sınıf            Key=1 → [Emre]                  Count=1
Mehmet  2. sınıf            Key=2 → [Mehmet]                Count=1
Zeynep  4. sınıf            Key=3 → [Ayşe, Gülşah]          Count=2
Emre    1. sınıf            Key=4 → [Zeynep]                Count=1
Gülşah  3. sınıf
```

---

## ⌨️ Adım 3: View

`Views/Home/Index.cshtml`:

```html
@model UBYS.Models.DashboardViewModel
@{
    ViewData["Title"] = "Panel";
}

@* ═══════════ ÜST KARTLAR ═══════════ *@
<div class="row g-3 mb-4">

    <div class="col-md-3">
        <a asp-controller="Fakulte" asp-action="Index" class="text-decoration-none">
            <div class="card border-0 shadow-sm h-100">
                <div class="card-body d-flex justify-content-between align-items-center">
                    <div>
                        <div class="text-muted small">Fakülte</div>
                        <div class="fs-3 fw-bold text-dark">@Model.FakulteSayisi</div>
                    </div>
                    <i class="bi bi-building fs-1 text-primary opacity-25"></i>
                </div>
            </div>
        </a>
    </div>

    <div class="col-md-3">
        <a asp-controller="Bolum" asp-action="Index" class="text-decoration-none">
            <div class="card border-0 shadow-sm h-100">
                <div class="card-body d-flex justify-content-between align-items-center">
                    <div>
                        <div class="text-muted small">Bölüm</div>
                        <div class="fs-3 fw-bold text-dark">@Model.BolumSayisi</div>
                    </div>
                    <i class="bi bi-diagram-3 fs-1 text-success opacity-25"></i>
                </div>
            </div>
        </a>
    </div>

    <div class="col-md-3">
        <a asp-controller="Ogrenci" asp-action="Index" class="text-decoration-none">
            <div class="card border-0 shadow-sm h-100">
                <div class="card-body d-flex justify-content-between align-items-center">
                    <div>
                        <div class="text-muted small">Öğrenci</div>
                        <div class="fs-3 fw-bold text-dark">@Model.OgrenciSayisi</div>
                    </div>
                    <i class="bi bi-people fs-1 text-warning opacity-25"></i>
                </div>
            </div>
        </a>
    </div>

    <div class="col-md-3">
        <a asp-controller="Akademisyen" asp-action="Index" class="text-decoration-none">
            <div class="card border-0 shadow-sm h-100">
                <div class="card-body d-flex justify-content-between align-items-center">
                    <div>
                        <div class="text-muted small">Akademisyen</div>
                        <div class="fs-3 fw-bold text-dark">@Model.AkademisyenSayisi</div>
                    </div>
                    <i class="bi bi-person-badge fs-1 text-info opacity-25"></i>
                </div>
            </div>
        </a>
    </div>

</div>


<div class="row g-3 mb-3">

    @* ═══════════ BÖLÜM DAĞILIMI ═══════════ *@
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm h-100">
            <div class="card-header bg-white fw-semibold">Bölümlere göre dağılım</div>
            <div class="card-body p-0">

                @if (Model.BolumDagilimlari.Count == 0)
                {
                    <p class="text-muted small p-3 mb-0">Henüz bölüm kaydı yok.</p>
                }
                else
                {
                    @{
                        // Çubukları en kalabalık bölüme göre ölçekleyeceğiz.
                        // ⚠️ Sıfıra bölme koruması: hiç öğrenci yoksa 0 olur.
                        int enYuksek = Model.BolumDagilimlari.Max(b => b.OgrenciSayisi);
                        if (enYuksek == 0) enYuksek = 1;
                    }

                    <table class="table table-sm mb-0 align-middle">
                        <thead class="table-light">
                            <tr>
                                <th>Bölüm</th>
                                <th>Fakülte</th>
                                <th class="text-center">Öğrenci</th>
                                <th class="text-center">Akademisyen</th>
                                <th style="width: 28%;">Doluluk</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach (var b in Model.BolumDagilimlari)
                            {
                                int yuzde = (b.OgrenciSayisi * 100) / enYuksek;
                                <tr>
                                    <td class="fw-semibold">@b.BolumAdi</td>
                                    <td class="text-muted small">@b.FakulteAd</td>
                                    <td class="text-center">@b.OgrenciSayisi</td>
                                    <td class="text-center">@b.AkademisyenSayisi</td>
                                    <td>
                                        @* Çubuk grafik — sadece CSS, kütüphane yok *@
                                        <div class="progress" style="height: 8px;">
                                            <div class="progress-bar bg-primary"
                                                 style="width: @yuzde%;"></div>
                                        </div>
                                    </td>
                                </tr>
                            }
                        </tbody>
                    </table>
                }

            </div>
        </div>
    </div>

    @* ═══════════ SINIF DAĞILIMI ═══════════ *@
    <div class="col-lg-4">
        <div class="card border-0 shadow-sm h-100">
            <div class="card-header bg-white fw-semibold">Sınıflara göre öğrenci</div>
            <div class="card-body">

                @if (Model.SinifDagilimlari.Count == 0)
                {
                    <p class="text-muted small mb-0">Henüz öğrenci kaydı yok.</p>
                }
                else
                {
                    @foreach (var s in Model.SinifDagilimlari)
                    {
                        int yuzde = Model.OgrenciSayisi > 0
                            ? (s.OgrenciSayisi * 100) / Model.OgrenciSayisi
                            : 0;

                        <div class="mb-3">
                            <div class="d-flex justify-content-between small mb-1">
                                <span>@s.Sinif. sınıf</span>
                                <span class="text-muted">@s.OgrenciSayisi kişi (%@yuzde)</span>
                            </div>
                            <div class="progress" style="height: 6px;">
                                <div class="progress-bar bg-warning" style="width: @yuzde%;"></div>
                            </div>
                        </div>
                    }
                }

            </div>
        </div>
    </div>

</div>


<div class="row g-3">

    @* ═══════════ CİNSİYET DAĞILIMI ═══════════ *@
    <div class="col-lg-4">
        <div class="card border-0 shadow-sm h-100">
            <div class="card-header bg-white fw-semibold">Cinsiyet dağılımı</div>
            <div class="card-body">

                @if (Model.CinsiyetDagilimlari.Count == 0)
                {
                    <p class="text-muted small mb-0">Henüz öğrenci kaydı yok.</p>
                }
                else
                {
                    @* Tek çubukta iki renk: aynı .progress içine iki .progress-bar *@
                    <div class="progress mb-3" style="height: 20px;">
                        @foreach (var c in Model.CinsiyetDagilimlari)
                        {
                            int yuzde = Model.OgrenciSayisi > 0
                                ? (c.OgrenciSayisi * 100) / Model.OgrenciSayisi
                                : 0;
                            string renk = c.Cinsiyet == "Kadın" ? "bg-danger" : "bg-primary";

                            <div class="progress-bar @renk" style="width: @yuzde%;">%@yuzde</div>
                        }
                    </div>

                    @foreach (var c in Model.CinsiyetDagilimlari)
                    {
                        string nokta = c.Cinsiyet == "Kadın" ? "text-danger" : "text-primary";
                        <div class="d-flex justify-content-between small mb-1">
                            <span>
                                <i class="bi bi-circle-fill @nokta" style="font-size: .5rem;"></i>
                                @c.Cinsiyet
                            </span>
                            <span class="text-muted">@c.OgrenciSayisi</span>
                        </div>
                    }
                }

            </div>
        </div>
    </div>

    @* ═══════════ SON EKLENENLER ═══════════ *@
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm h-100">
            <div class="card-header bg-white fw-semibold">Son eklenen öğrenciler</div>
            <div class="card-body p-0">

                @if (Model.SonEklenenOgrenciler.Count == 0)
                {
                    <p class="text-muted small p-3 mb-0">Henüz öğrenci kaydı yok.</p>
                }
                else
                {
                    <ul class="list-group list-group-flush">
                        @foreach (var o in Model.SonEklenenOgrenciler)
                        {
                            <li class="list-group-item d-flex justify-content-between align-items-center">
                                <div>
                                    <a asp-controller="Ogrenci" asp-action="Details"
                                       asp-route-id="@o.OgrenciId"
                                       class="text-decoration-none fw-semibold small">
                                        @o.TamAd
                                    </a>
                                    <div class="text-muted" style="font-size: .75rem;">
                                        @o.BolumAdi &middot; @o.Sinif. sınıf
                                    </div>
                                </div>
                                <span class="text-muted" style="font-size: .75rem;">
                                    @o.CreatedDate.ToString("dd.MM.yyyy")
                                </span>
                            </li>
                        }
                    </ul>
                }

            </div>
        </div>
    </div>

</div>
```

---

## ▶️ Üretilen SQL'i incele

Dashboard'ı aç ve konsola bak. **Beş ayrı sorgu** göreceksin. En ilginç ikisi:

### Bölüm dağılımı — alt sorgu

```sql
SELECT [b].[BolumAdi],
       [f].[FakulteAd],
       (SELECT COUNT(*) FROM [Ogrenciler] AS [o]
        WHERE [o].[AktifMi] = CAST(1 AS bit) AND [b].[BolumId] = [o].[BolumId]) AS [OgrenciSayisi],
       (SELECT COUNT(*) FROM [Akademisyenler] AS [a]
        WHERE [a].[AktifMi] = CAST(1 AS bit) AND [b].[BolumId] = [a].[BolumId]) AS [AkademisyenSayisi]
FROM [Bolumler] AS [b]
INNER JOIN [Fakulteler] AS [f] ON [b].[FakulteId] = [f].[FakulteId]
WHERE [b].[AktifMi] = CAST(1 AS bit) AND [f].[AktifMi] = CAST(1 AS bit)
ORDER BY ...
```

> **Sınıfa göster:** ADO.NET sürümünde bu alt sorguları **elle yazmıştık.** EF, `b.Ogrenciler.Count()` yazdığımız için aynısını üretti. Query filter'ı alt sorgulara da uygulamayı unutmadı.

### Sınıf dağılımı — GroupBy

```sql
SELECT [o].[OgrenciSinif] AS [Sinif], COUNT(*) AS [OgrenciSayisi]
FROM [Ogrenciler] AS [o]
WHERE [o].[AktifMi] = CAST(1 AS bit)
GROUP BY [o].[OgrenciSinif]
ORDER BY [o].[OgrenciSinif]
```

Tam beklediğimiz SQL.

---

## ⚠️ GroupBy tuzağı — bilinmesi gereken

Bazı `GroupBy` kullanımları SQL'e çevrilemez:

```csharp
// ❌ Çevrilemez — grup içindeki NESNELERİ istiyoruz
var gruplar = await _db.Ogrenciler
    .GroupBy(o => o.BolumId)
    .Select(g => new { g.Key, Ogrenciler = g.ToList() })
    .ToListAsync();
```

SQL `GROUP BY` yalnızca **özet değer** döndürebilir (`COUNT`, `SUM`, `MAX`...), grup içindeki satırları değil.

**Çözüm:** Veriyi çekip belleğe aldıktan sonra grupla:

```csharp
// ✅ Önce çek, sonra bellekte grupla
var ogrenciler = await _db.Ogrenciler.ToListAsync();
var gruplar = ogrenciler.GroupBy(o => o.BolumId).ToList();
```

> ⚠️ Ama dikkat: bu yöntem **tüm satırları belleğe alır.** 10.000 öğrenci varsa hepsi gelir. Sadece küçük veri kümelerinde kullanın.

---

## ▶️ Test senaryosu

| Test | Beklenen |
|---|---|
| Dashboard aç | Gerçek sayılar |
| SSMS'ten doğrula | `SELECT COUNT(*) FROM Ogrenciler WHERE AktifMi=1` |
| Yeni öğrenci ekle | Sayı arttı, "son eklenenler"de en üstte |
| Bir öğrenciyi sil | Sayı azaldı |
| Kartlara tıkla | İlgili listeye gidiyor |
| Boş veritabanı | **Çökmüyor**, sıfırlar görünüyor |
| Konsoldaki SQL | Beş sorgu, `COUNT` ve `GROUP BY` var |

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `The LINQ expression could not be translated` | `Select` içinde `[NotMapped]` özellik | Alanları elle birleştir |
| `Attempted to divide by zero` | Kayıt yokken yüzde hesabı | `> 0` kontrolü |
| `Object reference not set` | ViewModel listesi null | `= new()` ile başlat |
| `Sequence contains no elements` (`Max`) | Boş listede `Max()` | Önce `Count == 0` kontrol et |
| Sayılar yanlış | Query filter beklenmedik davranıyor | Konsoldaki SQL'e bak |
| Son eklenenler rastgele | `Take` önce, `OrderBy` sonra yazılmış | Sırayı düzelt |
| Dashboard çok yavaş | Gereksiz `Include` var | `Select` ile projeksiyon yap |

---

## ✏️ Öğrenci alıştırması

1. Dashboard'ı tamamla, boş veritabanıyla da test et.
2. Konsoldaki beş sorguyu incele. Hangisi en karmaşık? Neden?
3. "Fakülte özeti" kartı ekle: her fakülte için bölüm, öğrenci ve akademisyen sayısı.
   İpucu:
   ```csharp
   .Select(f => new {
       f.FakulteAd,
       BolumSayisi = f.Bolumler.Count(),
       OgrenciSayisi = f.Bolumler.SelectMany(b => b.Ogrenciler).Count()
   })
   ```
4. Ortalama öğrenci yaşını hesapla ve göster.
5. **Düşünme soruları:**
   - Dashboard beş sorgu çalıştırıyor. Tek sorguya indirmek mümkün mü? Değer mi?
   - `Select` ile projeksiyon yapmasaydık ne kadar fazla veri taşınırdı?
   - `b.Ogrenciler.Count()` yazdık ama `Include(b => b.Ogrenciler)` yazmadık. Neden çalıştı?

---

👉 Sonraki: [`10-arama-filtreleme-sayfalama.md`](10-arama-filtreleme-sayfalama.md)
