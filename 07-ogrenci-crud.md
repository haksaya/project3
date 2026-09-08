# Modül 6 — Öğrenci CRUD (Doğrulama ve Hata Yakalama)

**Süre:** 1-2 ders saati

---

## 🎯 Bu derste ne yapacağız

Kalıp aynı. **Yeni kavramlar:**

1. `DbUpdateException` — benzersizlik hatasını yakalamak
2. `ThenInclude` — iki seviye derin yükleme
3. `AnyAsync` ile önden kontrol

---

## 📖 Kavram: DbUpdateException

### Sorunu göster

Öğrencilere yaptır:
1. Bir öğrenci ekle, e-postası `test@okul.edu.tr`
2. Bir öğrenci daha ekle, **aynı e-postayla**

Sonuç: **sarı hata sayfası, uygulama çöktü.**

```
DbUpdateException: An error occurred while saving the entity changes.
 ---> SqlException: Cannot insert duplicate key row in object 'dbo.Ogrenciler'
      with unique index 'IX_Ogrenciler_OgrenciEposta'.
```

### ADO.NET'ten farkı

| | ADO.NET | EF Core |
|---|---|---|
| Fırlatılan hata | `SqlException` | `DbUpdateException` |
| Gerçek hata nerede | Doğrudan | `.InnerException` içinde |
| Hata kodu | `ex.Number` | `((SqlException)ex.InnerException).Number` |

EF, veritabanı hatasını kendi hatasının **içine sarar.** Bu yüzden bir katman derine inmemiz gerekiyor.

### Çözüm

```csharp
try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateException ex)
{
    // Gerçek SQL hatası içeride
    if (ex.InnerException is SqlException sqlEx &&
        (sqlEx.Number == 2601 || sqlEx.Number == 2627))
    {
        // benzersizlik ihlali
    }
}
```

`is SqlException sqlEx` → "eğer SqlException tipindeyse, onu `sqlEx` adıyla kullan" demek. Tek satırda hem tip kontrolü hem dönüştürme.

Gerekli using:
```csharp
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
```

---

## 📖 İki yaklaşım: önden kontrol mü, hata yakalama mı?

Öğrenciye ikisini de göster, farkı tartıştır.

```csharp
// ── YOL A: Önden kontrol ────────────────────────────
bool varMi = await _db.Ogrenciler
    .AnyAsync(o => o.OgrenciEposta == ogrenci.OgrenciEposta);

if (varMi)
{
    ModelState.AddModelError("OgrenciEposta", "Bu e-posta zaten kayıtlı.");
    return View(ogrenci);
}

// ── YOL B: Hata yakalama ────────────────────────────
try { await _db.SaveChangesAsync(); }
catch (DbUpdateException ex) { /* ... */ }
```

| | Yol A (önden kontrol) | Yol B (hata yakalama) |
|---|---|---|
| Mesaj kalitesi | ⭐ Hangi alan çakıştı net | Hata metninden çıkarmak gerekir |
| Ekstra sorgu | Var | Yok |
| **Yarış koşulu** | ❌ Açık | ✅ Güvenli |

### ⚠️ Yarış koşulu (race condition) nedir?

```
Kullanıcı A                    Kullanıcı B
───────────                    ───────────
"ayse@x.com var mı?" → Hayır
                               "ayse@x.com var mı?" → Hayır
INSERT → başarılı
                               INSERT → 💥 ÇÖKER
```

İkisi arasında milisaniyeler var ama yeterli. **Tek başına önden kontrol yetmez.**

> **Doğru çözüm: ikisini birden kullanmak.** Yol A güzel mesaj verir, Yol B nadir durumda çökmeyi engeller. Aşağıdaki kodda ikisi de var.

---

## ⌨️ Adım 1: Controller

`Controllers/OgrenciController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class OgrenciController : Controller
{
    private readonly UbysDbContext _db;

    public OgrenciController(UbysDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: bölüm açılır listesi
    //
    //  ⭐ Select ile SADECE ihtiyacımız olan iki alanı çekiyoruz.
    //     Tüm Bolum nesnesini çekmeye gerek yok — açılır listede
    //     yalnızca id ve ad kullanılacak.
    // ══════════════════════════════════════════════════════
    private async Task BolumListesiniHazirlaAsync(long? secili = null)
    {
        var bolumler = await _db.Bolumler
            .Include(b => b.Fakulte)
            .OrderBy(b => b.Fakulte!.FakulteAd)
            .ThenBy(b => b.BolumAdi)
            .Select(b => new
            {
                b.BolumId,
                // Aynı adlı bölüm farklı fakültelerde olabilir — karışmasın
                Ad = b.BolumAdi + " (" + b.Fakulte!.FakulteAd + ")"
            })
            .ToListAsync();

        ViewBag.Bolumler = new SelectList(bolumler, "BolumId", "Ad", secili);
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: benzersizlik ön kontrolü
    //
    //  guncellenenId: düzenleme yaparken kaydın KENDİSİNİ
    //  çakışma sayma. Yoksa kendi e-postasıyla çakışır.
    // ══════════════════════════════════════════════════════
    private async Task BenzersizlikKontrolAsync(Ogrenci ogrenci, long guncellenenId = 0)
    {
        bool epostaVar = await _db.Ogrenciler
            .AnyAsync(o => o.OgrenciEposta == ogrenci.OgrenciEposta
                        && o.OgrenciId != guncellenenId);

        if (epostaVar)
            ModelState.AddModelError(nameof(Ogrenci.OgrenciEposta),
                "Bu e-posta adresi başka bir öğrenciye kayıtlı.");

        bool tcVar = await _db.Ogrenciler
            .AnyAsync(o => o.OgrenciTc == ogrenci.OgrenciTc
                        && o.OgrenciId != guncellenenId);

        if (tcVar)
            ModelState.AddModelError(nameof(Ogrenci.OgrenciTc),
                "Bu TC kimlik numarası başka bir öğrenciye kayıtlı.");
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: veritabanı hatasını kullanıcı diline çevir
    //
    //  ⚠️ ex.Message'ı OLDUĞU GİBİ GÖSTERME!
    //     Tablo ve indeks adlarını ifşa eder.
    // ══════════════════════════════════════════════════════
    private void VeritabaniHatasiniIsle(DbUpdateException ex)
    {
        if (ex.InnerException is SqlException sqlEx &&
            (sqlEx.Number == 2601 || sqlEx.Number == 2627))
        {
            string mesaj = sqlEx.Message;

            if (mesaj.Contains("Eposta"))
                ModelState.AddModelError(nameof(Ogrenci.OgrenciEposta),
                    "Bu e-posta adresi başka bir öğrenciye kayıtlı.");
            else if (mesaj.Contains("Tc"))
                ModelState.AddModelError(nameof(Ogrenci.OgrenciTc),
                    "Bu TC kimlik numarası başka bir öğrenciye kayıtlı.");
            else
                ModelState.AddModelError("", "Bu kayıt zaten mevcut.");
        }
        else
        {
            ModelState.AddModelError("",
                "Kayıt sırasında bir sorun oluştu. Lütfen tekrar deneyin.");
        }
    }

    // ══════════════════════════════════════════════════════
    //  1) LİSTELEME
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Index()
    {
        var ogrenciler = await _db.Ogrenciler
            .Include(o => o.Bolum)                    // bölüm
                .ThenInclude(b => b!.Fakulte)         // ⭐ bölümün fakültesi
            .OrderBy(o => o.OgrenciAd)
            .ThenBy(o => o.OgrenciSoyad)
            .ToListAsync();

        return View(ogrenciler);
    }

    // ══════════════════════════════════════════════════════
    //  2) YENİ KAYIT FORMU
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Create()
    {
        await BolumListesiniHazirlaAsync();
        return View();
    }

    // ══════════════════════════════════════════════════════
    //  3) YENİ KAYDI KAYDET
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(Ogrenci ogrenci)
    {
        // Önden kontrol — güzel hata mesajı için
        await BenzersizlikKontrolAsync(ogrenci);

        if (!ModelState.IsValid)
        {
            await BolumListesiniHazirlaAsync(ogrenci.BolumId);
            return View(ogrenci);
        }

        ogrenci.CreatedDate = DateTime.Now;
        ogrenci.UpdatedDate = null;
        ogrenci.AktifMi = true;

        try
        {
            _db.Ogrenciler.Add(ogrenci);
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateException ex)
        {
            // Yarış koşulu ağa takıldı — nadir ama mümkün
            VeritabaniHatasiniIsle(ex);
            await BolumListesiniHazirlaAsync(ogrenci.BolumId);
            return View(ogrenci);
        }

        TempData["Basarili"] = $"{ogrenci.TamAd} kaydedildi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  4) DÜZENLEME FORMU
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Edit(long id)
    {
        var ogrenci = await _db.Ogrenciler.FindAsync(id);

        if (ogrenci == null)
            return NotFound();

        await BolumListesiniHazirlaAsync(ogrenci.BolumId);
        return View(ogrenci);
    }

    // ══════════════════════════════════════════════════════
    //  5) DÜZENLEMEYİ KAYDET
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(Ogrenci ogrenci)
    {
        // ⭐ Kendi kaydını çakışma sayma
        await BenzersizlikKontrolAsync(ogrenci, ogrenci.OgrenciId);

        if (!ModelState.IsValid)
        {
            await BolumListesiniHazirlaAsync(ogrenci.BolumId);
            return View(ogrenci);
        }

        var mevcut = await _db.Ogrenciler.FindAsync(ogrenci.OgrenciId);

        if (mevcut == null)
            return NotFound();

        mevcut.BolumId            = ogrenci.BolumId;
        mevcut.OgrenciAd          = ogrenci.OgrenciAd;
        mevcut.OgrenciSoyad       = ogrenci.OgrenciSoyad;
        mevcut.OgrenciSinif       = ogrenci.OgrenciSinif;
        mevcut.OgrenciDogumTarihi = ogrenci.OgrenciDogumTarihi;
        mevcut.OgrenciCinsiyet    = ogrenci.OgrenciCinsiyet;
        mevcut.OgrenciAdres       = ogrenci.OgrenciAdres;
        mevcut.OgrenciTelefon     = ogrenci.OgrenciTelefon;
        mevcut.OgrenciEposta      = ogrenci.OgrenciEposta;
        mevcut.OgrenciTc          = ogrenci.OgrenciTc;
        mevcut.UpdatedDate        = DateTime.Now;

        try
        {
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateException ex)
        {
            VeritabaniHatasiniIsle(ex);
            await BolumListesiniHazirlaAsync(ogrenci.BolumId);
            return View(ogrenci);
        }

        TempData["Basarili"] = $"{ogrenci.TamAd} güncellendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  6) SİLME ONAY SAYFASI
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Delete(long id)
    {
        var ogrenci = await _db.Ogrenciler
            .Include(o => o.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .FirstOrDefaultAsync(o => o.OgrenciId == id);

        if (ogrenci == null)
            return NotFound();

        return View(ogrenci);
    }

    // ══════════════════════════════════════════════════════
    //  7) SİLMEYİ ONAYLA
    // ══════════════════════════════════════════════════════
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(long id)
    {
        var ogrenci = await _db.Ogrenciler.FindAsync(id);

        if (ogrenci == null)
            return NotFound();

        ogrenci.AktifMi = false;
        ogrenci.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Öğrenci silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  8) DETAY
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Details(long id)
    {
        var ogrenci = await _db.Ogrenciler
            .Include(o => o.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .FirstOrDefaultAsync(o => o.OgrenciId == id);

        if (ogrenci == null)
            return NotFound();

        return View(ogrenci);
    }
}
```

### 📖 `ThenInclude` — iki seviye derin

```csharp
.Include(o => o.Bolum)                // öğrenci → bölüm
    .ThenInclude(b => b!.Fakulte)     // bölüm → fakülte
```

Ürettiği SQL:
```sql
FROM [Ogrenciler] AS [o]
INNER JOIN [Bolumler] AS [b] ON [o].[BolumId] = [b].[BolumId]
INNER JOIN [Fakulteler] AS [f] ON [b].[FakulteId] = [f].[FakulteId]
```

Artık view'da:
```html
@ogrenci.Bolum?.BolumAdi
@ogrenci.Bolum?.Fakulte?.FakulteAd
```

> **ADO.NET'te bu iki JOIN'i elle yazıyorduk.** Şimdi iki satır.

### 📖 `Select` ile projeksiyon

`BolumListesiniHazirlaAsync` içinde:

```csharp
.Select(b => new { b.BolumId, Ad = b.BolumAdi + " (" + b.Fakulte!.FakulteAd + ")" })
```

Bu, EF'e "bana tüm bölüm nesnesini değil, **sadece bu iki değeri** getir" der.

Üretilen SQL:
```sql
SELECT [b].[BolumId], [b].[BolumAdi] + N' (' + [f].[FakulteAd] + N')' AS [Ad]
FROM [Bolumler] AS [b]
INNER JOIN [Fakulteler] AS [f] ON ...
```

**Sadece iki sütun çekiliyor.** Adres, telefon, e-posta hiç gelmiyor. Bu, büyük tablolarda ciddi fark yaratır.

> Modül 8'de (dashboard) bunu daha çok kullanacağız.

---

## ⌨️ Adım 2: View'lar

### Index.cshtml

```html
@model List<UBYS.Models.Ogrenci>
@{
    ViewData["Title"] = "Öğrenciler";
}

@if (TempData["Basarili"] != null)
{
    <div class="alert alert-success alert-dismissible fade show">
        <i class="bi bi-check-circle"></i> @TempData["Basarili"]
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
}

<div class="card border-0 shadow-sm">
    <div class="card-header bg-white d-flex justify-content-between align-items-center">
        <span class="fw-semibold">Kayıtlı öğrenciler (@Model.Count)</span>
        <a asp-action="Create" class="btn btn-primary btn-sm">
            <i class="bi bi-plus-lg"></i> Yeni öğrenci
        </a>
    </div>

    <div class="card-body p-0">
        @if (Model.Count == 0)
        {
            <div class="text-center text-muted py-5">
                <i class="bi bi-inbox fs-1 d-block mb-2"></i>
                <p class="mb-3">Henüz öğrenci yok.</p>
                <a asp-action="Create" class="btn btn-sm btn-primary">İlk öğrenciyi ekle</a>
            </div>
        }
        else
        {
            <table class="table table-hover align-middle mb-0">
                <thead class="table-light">
                    <tr>
                        <th>Ad soyad</th>
                        <th>Bölüm</th>
                        <th>Fakülte</th>
                        <th class="text-center">Sınıf</th>
                        <th class="text-center">Yaş</th>
                        <th>TC</th>
                        <th>İletişim</th>
                        <th class="text-end">İşlemler</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var o in Model)
                    {
                        @{
                            // Yaş hesabı
                            int yas = DateTime.Today.Year - o.OgrenciDogumTarihi.Year;
                            if (o.OgrenciDogumTarihi.Date > DateTime.Today.AddYears(-yas))
                                yas--;

                            // TC maskeleme — kişisel veri, listede tam gösterilmez
                            string tcMaskeli = o.OgrenciTc.Length >= 3
                                ? o.OgrenciTc.Substring(0, 3) + "********"
                                : "***********";
                        }

                        <tr>
                            <td>
                                <a asp-action="Details" asp-route-id="@o.OgrenciId"
                                   class="fw-semibold text-decoration-none">
                                    @o.TamAd
                                </a>
                                <div class="text-muted small">@o.OgrenciCinsiyet</div>
                            </td>
                            <td>
                                @* Include ile geldi *@
                                <span class="badge bg-secondary">@o.Bolum?.BolumAdi</span>
                            </td>
                            <td class="small text-muted">
                                @* ThenInclude ile geldi *@
                                @o.Bolum?.Fakulte?.FakulteAd
                            </td>
                            <td class="text-center">
                                <span class="badge bg-info">@o.OgrenciSinif</span>
                            </td>
                            <td class="text-center">@yas</td>
                            <td><code class="small">@tcMaskeli</code></td>
                            <td class="small">
                                <div>@o.OgrenciTelefon</div>
                                <div class="text-muted">@o.OgrenciEposta</div>
                            </td>
                            <td class="text-end text-nowrap">
                                <a asp-action="Edit" asp-route-id="@o.OgrenciId"
                                   class="btn btn-sm btn-outline-primary">
                                    <i class="bi bi-pencil"></i>
                                </a>
                                <a asp-action="Delete" asp-route-id="@o.OgrenciId"
                                   class="btn btn-sm btn-outline-danger">
                                    <i class="bi bi-trash"></i>
                                </a>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        }
    </div>
</div>
```

### Create.cshtml

```html
@model UBYS.Models.Ogrenci
@{
    ViewData["Title"] = "Yeni öğrenci";
}

<div class="row">
    <div class="col-lg-10">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Create" method="post">

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <h6 class="text-muted mb-3">Kişisel bilgiler</h6>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciAd" class="form-label"></label>
                            <input asp-for="OgrenciAd" class="form-control" autofocus />
                            <span asp-validation-for="OgrenciAd" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciSoyad" class="form-label"></label>
                            <input asp-for="OgrenciSoyad" class="form-control" />
                            <span asp-validation-for="OgrenciSoyad" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-4 mb-3">
                            <label asp-for="OgrenciDogumTarihi" class="form-label"></label>
                            <input asp-for="OgrenciDogumTarihi" type="date" class="form-control" />
                            <span asp-validation-for="OgrenciDogumTarihi" class="text-danger small"></span>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label class="form-label">Cinsiyet</label>
                            <div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="OgrenciCinsiyet" value="Kadın" id="cinsiyetK" />
                                    <label class="form-check-label" for="cinsiyetK">Kadın</label>
                                </div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="OgrenciCinsiyet" value="Erkek" id="cinsiyetE" />
                                    <label class="form-check-label" for="cinsiyetE">Erkek</label>
                                </div>
                            </div>
                            <span asp-validation-for="OgrenciCinsiyet" class="text-danger small"></span>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label asp-for="OgrenciSinif" class="form-label"></label>
                            <input asp-for="OgrenciSinif" type="number" min="1" max="6"
                                   class="form-control" />
                            <span asp-validation-for="OgrenciSinif" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="OgrenciTc" class="form-label"></label>
                        <input asp-for="OgrenciTc" class="form-control" maxlength="11"
                               placeholder="11 haneli TC kimlik numarası" />
                        <span asp-validation-for="OgrenciTc" class="text-danger small"></span>
                    </div>

                    <hr class="my-4" />
                    <h6 class="text-muted mb-3">Kayıt ve iletişim bilgileri</h6>

                    <div class="mb-3">
                        <label asp-for="BolumId" class="form-label"></label>
                        <select asp-for="BolumId" asp-items="ViewBag.Bolumler" class="form-select">
                            <option value="">-- Bölüm seçin --</option>
                        </select>
                        <span asp-validation-for="BolumId" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="OgrenciAdres" class="form-label"></label>
                        <textarea asp-for="OgrenciAdres" class="form-control" rows="2"></textarea>
                        <span asp-validation-for="OgrenciAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciTelefon" class="form-label"></label>
                            <input asp-for="OgrenciTelefon" class="form-control"
                                   placeholder="05321234567" />
                            <span asp-validation-for="OgrenciTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciEposta" class="form-label"></label>
                            <input asp-for="OgrenciEposta" class="form-control"
                                   placeholder="ad.soyad@@ogr.okul.edu.tr" />
                            <span asp-validation-for="OgrenciEposta" class="text-danger small"></span>
                        </div>
                    </div>

                    <hr />
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-lg"></i> Öğrenciyi kaydet
                    </button>
                    <a asp-action="Index" class="btn btn-outline-secondary">Vazgeç</a>

                </form>
            </div>
        </div>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### Edit.cshtml

Create'in aynısı, tek fark gizli id alanı:

```html
@model UBYS.Models.Ogrenci
@{
    ViewData["Title"] = "Öğrenciyi düzenle";
}

<div class="row">
    <div class="col-lg-10">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Edit" method="post">

                    @* Sadece id. CreatedDate ve AktifMi'yi taşımıyoruz —
                       controller onları veritabanından okuyor. *@
                    <input type="hidden" asp-for="OgrenciId" />

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <h6 class="text-muted mb-3">Kişisel bilgiler</h6>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciAd" class="form-label"></label>
                            <input asp-for="OgrenciAd" class="form-control" />
                            <span asp-validation-for="OgrenciAd" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciSoyad" class="form-label"></label>
                            <input asp-for="OgrenciSoyad" class="form-control" />
                            <span asp-validation-for="OgrenciSoyad" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-4 mb-3">
                            <label asp-for="OgrenciDogumTarihi" class="form-label"></label>
                            <input asp-for="OgrenciDogumTarihi" type="date" class="form-control" />
                            <span asp-validation-for="OgrenciDogumTarihi" class="text-danger small"></span>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label class="form-label">Cinsiyet</label>
                            <div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="OgrenciCinsiyet" value="Kadın" id="cinsiyetK" />
                                    <label class="form-check-label" for="cinsiyetK">Kadın</label>
                                </div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="OgrenciCinsiyet" value="Erkek" id="cinsiyetE" />
                                    <label class="form-check-label" for="cinsiyetE">Erkek</label>
                                </div>
                            </div>
                            <span asp-validation-for="OgrenciCinsiyet" class="text-danger small"></span>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label asp-for="OgrenciSinif" class="form-label"></label>
                            <input asp-for="OgrenciSinif" type="number" min="1" max="6"
                                   class="form-control" />
                            <span asp-validation-for="OgrenciSinif" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="OgrenciTc" class="form-label"></label>
                        <input asp-for="OgrenciTc" class="form-control" maxlength="11" />
                        <span asp-validation-for="OgrenciTc" class="text-danger small"></span>
                    </div>

                    <hr class="my-4" />
                    <h6 class="text-muted mb-3">Kayıt ve iletişim bilgileri</h6>

                    <div class="mb-3">
                        <label asp-for="BolumId" class="form-label"></label>
                        <select asp-for="BolumId" asp-items="ViewBag.Bolumler" class="form-select">
                            <option value="">-- Bölüm seçin --</option>
                        </select>
                        <span asp-validation-for="BolumId" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="OgrenciAdres" class="form-label"></label>
                        <textarea asp-for="OgrenciAdres" class="form-control" rows="2"></textarea>
                        <span asp-validation-for="OgrenciAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciTelefon" class="form-label"></label>
                            <input asp-for="OgrenciTelefon" class="form-control" />
                            <span asp-validation-for="OgrenciTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="OgrenciEposta" class="form-label"></label>
                            <input asp-for="OgrenciEposta" class="form-control" />
                            <span asp-validation-for="OgrenciEposta" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="text-muted small mb-3">
                        <i class="bi bi-clock-history"></i>
                        Kayıt: @Model.CreatedDate.ToString("dd.MM.yyyy HH:mm")
                        @if (Model.UpdatedDate.HasValue)
                        {
                            <span> | Güncelleme: @Model.UpdatedDate.Value.ToString("dd.MM.yyyy HH:mm")</span>
                        }
                    </div>

                    <hr />
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-lg"></i> Değişiklikleri kaydet
                    </button>
                    <a asp-action="Index" class="btn btn-outline-secondary">Vazgeç</a>

                </form>
            </div>
        </div>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### Delete.cshtml

```html
@model UBYS.Models.Ogrenci
@{
    ViewData["Title"] = "Öğrenciyi sil";
}

<div class="row">
    <div class="col-lg-7">
        <div class="card border-danger shadow-sm">
            <div class="card-header bg-danger text-white">
                <i class="bi bi-exclamation-triangle"></i> Silme onayı
            </div>
            <div class="card-body">

                <dl class="row mb-4">
                    <dt class="col-sm-4">Ad soyad</dt>
                    <dd class="col-sm-8 fw-semibold">@Model.TamAd</dd>
                    <dt class="col-sm-4">Bölüm</dt>
                    <dd class="col-sm-8">@Model.Bolum?.BolumAdi</dd>
                    <dt class="col-sm-4">Fakülte</dt>
                    <dd class="col-sm-8">@Model.Bolum?.Fakulte?.FakulteAd</dd>
                    <dt class="col-sm-4">Sınıf</dt>
                    <dd class="col-sm-8">@Model.OgrenciSinif. sınıf</dd>
                    <dt class="col-sm-4">TC kimlik no</dt>
                    <dd class="col-sm-8">@Model.OgrenciTc</dd>
                    <dt class="col-sm-4">E-posta</dt>
                    <dd class="col-sm-8">@Model.OgrenciEposta</dd>
                </dl>

                <div class="alert alert-info small">
                    <i class="bi bi-info-circle"></i>
                    Kayıt tamamen silinmez, pasif duruma alınır.
                </div>

                <form asp-action="Delete" method="post">
                    <input type="hidden" asp-for="OgrenciId" name="id" />
                    <button type="submit" class="btn btn-danger">
                        <i class="bi bi-trash"></i> Evet, sil
                    </button>
                    <a asp-action="Index" class="btn btn-outline-secondary">Vazgeç</a>
                </form>

            </div>
        </div>
    </div>
</div>
```

### Details.cshtml

```html
@model UBYS.Models.Ogrenci
@{
    ViewData["Title"] = "Öğrenci detayı";

    int yas = DateTime.Today.Year - Model.OgrenciDogumTarihi.Year;
    if (Model.OgrenciDogumTarihi.Date > DateTime.Today.AddYears(-yas))
        yas--;

    string basHarfler = "";
    if (Model.OgrenciAd.Length > 0) basHarfler += Model.OgrenciAd[0];
    if (Model.OgrenciSoyad.Length > 0) basHarfler += Model.OgrenciSoyad[0];
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">

            <div class="card-body border-bottom">
                <div class="d-flex align-items-center gap-3">
                    <div class="rounded-circle bg-primary text-white d-flex
                                align-items-center justify-content-center fw-bold"
                         style="width: 56px; height: 56px; font-size: 1.25rem;">
                        @basHarfler.ToUpper()
                    </div>
                    <div>
                        <h5 class="mb-1">@Model.TamAd</h5>
                        <span class="badge bg-secondary">@Model.Bolum?.BolumAdi</span>
                        <span class="badge bg-info">@Model.OgrenciSinif. sınıf</span>
                        <div class="text-muted small mt-1">@Model.Bolum?.Fakulte?.FakulteAd</div>
                    </div>
                </div>
            </div>

            <div class="card-body">
                <h6 class="text-muted mb-3">Kişisel bilgiler</h6>
                <dl class="row mb-4">
                    <dt class="col-sm-4 fw-normal text-muted">TC kimlik no</dt>
                    <dd class="col-sm-8">@Model.OgrenciTc</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Doğum tarihi</dt>
                    <dd class="col-sm-8">
                        @Model.OgrenciDogumTarihi.ToString("dd MMMM yyyy")
                        <span class="text-muted">(@yas yaşında)</span>
                    </dd>
                    <dt class="col-sm-4 fw-normal text-muted">Cinsiyet</dt>
                    <dd class="col-sm-8">@Model.OgrenciCinsiyet</dd>
                </dl>

                <h6 class="text-muted mb-3">İletişim</h6>
                <dl class="row mb-4">
                    <dt class="col-sm-4 fw-normal text-muted">Telefon</dt>
                    <dd class="col-sm-8">@Model.OgrenciTelefon</dd>
                    <dt class="col-sm-4 fw-normal text-muted">E-posta</dt>
                    <dd class="col-sm-8">@Model.OgrenciEposta</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Adres</dt>
                    <dd class="col-sm-8">@Model.OgrenciAdres</dd>
                </dl>

                <h6 class="text-muted mb-3">Kayıt bilgileri</h6>
                <dl class="row mb-0">
                    <dt class="col-sm-4 fw-normal text-muted">Kayıt tarihi</dt>
                    <dd class="col-sm-8">@Model.CreatedDate.ToString("dd.MM.yyyy HH:mm")</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Son güncelleme</dt>
                    <dd class="col-sm-8">
                        @(Model.UpdatedDate?.ToString("dd.MM.yyyy HH:mm") ?? "Henüz güncellenmemiş")
                    </dd>
                </dl>
            </div>

            <div class="card-footer bg-white">
                <a asp-action="Edit" asp-route-id="@Model.OgrenciId" class="btn btn-primary btn-sm">
                    <i class="bi bi-pencil"></i> Düzenle
                </a>
                <a asp-action="Delete" asp-route-id="@Model.OgrenciId" class="btn btn-outline-danger btn-sm">
                    <i class="bi bi-trash"></i> Sil
                </a>
                <a asp-action="Index" class="btn btn-outline-secondary btn-sm">
                    <i class="bi bi-arrow-left"></i> Listeye dön
                </a>
            </div>

        </div>
    </div>
</div>
```

---

## ▶️ Test senaryosu

| # | Test | Beklenen |
|---|---|---|
| 1 | Boş form gönder | Tüm alanlarda hata |
| 2 | TC'ye `123` yaz | "11 haneli olmalı" |
| 3 | TC'ye `01234567890` | "0 ile başlamamalı" |
| 4 | Sınıfa `9` | "1-6 arasında olmalı" |
| 5 | **Aynı e-postayı ikinci kez** | Kırmızı uyarı, **çökme yok** |
| 6 | **Aynı TC'yi ikinci kez** | Kırmızı uyarı, **çökme yok** |
| 7 | Kendi e-postasıyla düzenle, kaydet | **Hata yok** (kendi kaydı) |
| 8 | Hatalı gönderim sonrası | Açılır liste dolu, yazılanlar duruyor |
| 9 | Listede fakülte sütunu | Dolu (ThenInclude çalışıyor) |
| 10 | Konsoldaki SQL | İki `INNER JOIN` var |

> **7. maddeyi mutlaka test ettir.** `BenzersizlikKontrolAsync` metodundaki `o.OgrenciId != guncellenenId` koşulu olmasaydı, öğrenci kendi e-postasıyla çakışır ve hiç kaydedilemezdi.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `DbUpdateException` sayfası çıkıyor | `try-catch` yok | Ekle |
| `SqlException` tanınmıyor | using eksik | `using Microsoft.Data.SqlClient;` |
| `ex.Number` bulunamıyor | `DbUpdateException`'da yok | `ex.InnerException` içine bak |
| Düzenlemede "e-posta kayıtlı" hatası | Kendi kaydı sayılıyor | `o.OgrenciId != guncellenenId` |
| Fakülte sütunu boş | `ThenInclude` yok | Ekle |
| `b!.Fakulte` derleme uyarısı | Null kontrolü | `!` ekle veya `?.` kullan |
| Açılır liste boş → çökme | POST'ta hazırlanmamış | `ModelState` bloğuna ekle |
| Cinsiyet seçili gelmiyor (Edit) | `asp-for` yerine elle `name` yazılmış | `asp-for` kullan |

---

## ✏️ Öğrenci alıştırması

1. Öğrenci CRUD'unu tamamla, on test senaryosunu geçir.
2. Konsolda `Index` sorgusunun SQL'ine bak. Kaç JOIN var? Neden?
3. `BenzersizlikKontrolAsync` çağrısını sil. Aynı e-postayı iki kez gir. Hangi hata mesajını alıyorsun? `try-catch` onu nasıl yakalıyor?
4. Telefon numarası için de benzersizlik indeksi ekle (Fluent API + migration) ve kontrolünü yaz.
5. **Düşünme soruları:**
   - Önden kontrol ve hata yakalamayı **ikisini birden** kullandık. Sadece biri yeterli olmaz mıydı?
   - `Select` ile projeksiyon yaptığımızda EF hangi sütunları çekiyor? Tümünü çekseydik ne fark ederdi?
   - `AnyAsync` yerine `CountAsync(...) > 0` yazsaydık ne değişirdi? Hangisi daha hızlı?

---

👉 Sonraki: [`08-akademisyen-crud.md`](08-akademisyen-crud.md)
