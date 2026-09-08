# Modül 5 — Bölüm CRUD (İlişkiler)

**Süre:** 1-2 ders saati

---

## 🎯 Bu derste ne yapacağız

Aynı CRUD kalıbı, bu sefer **ilişkili tablo** ile. Yeni kavramlar:

1. `Include()` — ilişkili veriyi getirmek (eager loading)
2. N+1 problemi ve nasıl kaçınılacağı
3. Navigasyon özelliğinin doğrulama tuzağı

---

## 📖 Kavram: Include — ilişkili veriyi getirmek

### Sorun

Bölüm listesinde fakülte adını göstermek istiyoruz.

```csharp
var bolumler = await _db.Bolumler.ToListAsync();
```

View'da:
```html
@bolum.Fakulte.FakulteAd     <!-- ❌ NullReferenceException! -->
```

**Neden çöktü?** Çünkü EF `Fakulte` nesnesini **getirmedi.** Navigasyon özelliği tanımlı olması, verinin otomatik geleceği anlamına gelmez.

### Çözüm: `Include`

```csharp
var bolumler = await _db.Bolumler
    .Include(b => b.Fakulte)          // ⭐ fakülteyi de getir
    .ToListAsync();
```

Konsola bak — EF `JOIN` üretti:

```sql
SELECT [b].[BolumId], [b].[BolumAdi], ..., [f].[FakulteId], [f].[FakulteAd], ...
FROM [Bolumler] AS [b]
INNER JOIN [Fakulteler] AS [f] ON [b].[FakulteId] = [f].[FakulteId]
WHERE [b].[AktifMi] = CAST(1 AS bit) AND [f].[AktifMi] = CAST(1 AS bit)
ORDER BY [b].[BolumAdi]
```

> ⭐ **ADO.NET'te bu JOIN'i elle yazıyorduk.** Şimdi `.Include(b => b.Fakulte)` diyoruz, EF `INNER JOIN`'i kendisi yazıyor.
>
> Ayrıca dikkat: `WHERE` içinde **iki** filtre var. Query filter hem bölüme hem fakülteye uygulandı — bunu biz istemedik, EF ilişkili tabloya da kendi filtresini ekledi.

### Yükleme (loading) türleri

| Tür | Nasıl | Ne zaman |
|---|---|---|
| **Eager loading** | `.Include(...)` | ⭐ Varsayılan tercih. Baştan biliyorsanız. |
| **Explicit loading** | `_db.Entry(b).Reference(x => x.Fakulte).LoadAsync()` | Sonradan gerekirse |
| **Lazy loading** | Otomatik, erişince yüklenir | ⚠️ Kapalı gelir, açmayın (aşağıda) |

### ⚠️ Lazy loading neden kapalı?

Lazy loading açık olsaydı, `bolum.Fakulte.FakulteAd` yazdığınız anda EF arka planda **gizlice** bir sorgu çalıştırırdı. 50 bölümlük bir listede 50 gizli sorgu.

Buna **N+1 problemi** denir ve EF projelerinde en yaygın performans hatasıdır.

```
❌ Lazy loading ile:
   1 sorgu  → 50 bölümü getir
   50 sorgu → her birinin fakültesini getir
   ─────────
   51 sorgu

✅ Include ile:
   1 sorgu  → bölümleri VE fakülteleri JOIN ile getir
```

> **Öğrenciye:** *"ADO.NET'te bu hatayı yapamazdınız çünkü her sorguyu elle yazıyordunuz. EF'te kolayca yapabilirsiniz — bu yüzden konsoldaki SQL'e bakmak zorundasınız."*

Modül 11'de N+1'i canlı göstereceğiz.

### İç içe Include

Öğrencinin bölümünü ve o bölümün fakültesini birlikte istersek:

```csharp
var ogrenciler = await _db.Ogrenciler
    .Include(o => o.Bolum)                  // önce bölüm
        .ThenInclude(b => b!.Fakulte)       // sonra bölümün fakültesi
    .ToListAsync();
```

Artık `ogrenci.Bolum.Fakulte.FakulteAd` çalışır.

---

## ⌨️ Adım 1: Controller

`Controllers/BolumController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class BolumController : Controller
{
    // ⭐ ADO.NET'te İKİ repository enjekte ediyorduk
    //    (BolumRepository + FakulteRepository).
    //    EF'te tek DbContext yeter — içinde tüm tablolar var.
    private readonly UbysDbContext _db;

    public BolumController(UbysDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: fakülte açılır listesi
    // ══════════════════════════════════════════════════════
    private async Task FakulteListesiniHazirlaAsync(long? secili = null)
    {
        var fakulteler = await _db.Fakulteler
            .OrderBy(f => f.FakulteAd)
            .ToListAsync();

        ViewBag.Fakulteler = new SelectList(fakulteler, "FakulteId", "FakulteAd", secili);
    }

    // ══════════════════════════════════════════════════════
    //  1) LİSTELEME
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Index()
    {
        var bolumler = await _db.Bolumler
            .Include(b => b.Fakulte)              // ⭐ JOIN
            .OrderBy(b => b.Fakulte!.FakulteAd)   // fakülte adına göre
            .ThenBy(b => b.BolumAdi)              // sonra bölüm adına göre
            .ToListAsync();

        return View(bolumler);
    }

    // ══════════════════════════════════════════════════════
    //  2) YENİ KAYIT FORMU
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Create()
    {
        await FakulteListesiniHazirlaAsync();
        return View();
    }

    // ══════════════════════════════════════════════════════
    //  3) YENİ KAYDI KAYDET
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(Bolum bolum)
    {
        if (!ModelState.IsValid)
        {
            // ⭐ EN ÇOK UNUTULAN SATIR
            // ViewBag sadece o istek boyunca yaşar. POST yeni bir istektir.
            // Doldurmazsak açılır liste boş gelir ve sayfa çöker.
            await FakulteListesiniHazirlaAsync(bolum.FakulteId);
            return View(bolum);
        }

        bolum.CreatedDate = DateTime.Now;
        bolum.UpdatedDate = null;
        bolum.AktifMi = true;

        // ⚠️ ÖNEMLİ: bolum.Fakulte navigasyon özelliği NULL.
        //    Bu SORUN DEĞİL — EF, FakulteId alanına bakar.
        //    Eğer Fakulte nesnesini de doldursaydık EF onu
        //    YENİ BİR FAKÜLTE sanıp INSERT etmeye çalışabilirdi.
        _db.Bolumler.Add(bolum);
        await _db.SaveChangesAsync();

        TempData["Basarili"] = $"\"{bolum.BolumAdi}\" kaydedildi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  4) DÜZENLEME FORMU
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Edit(long id)
    {
        var bolum = await _db.Bolumler.FindAsync(id);

        if (bolum == null)
            return NotFound();

        await FakulteListesiniHazirlaAsync(bolum.FakulteId);
        return View(bolum);
    }

    // ══════════════════════════════════════════════════════
    //  5) DÜZENLEMEYİ KAYDET
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(Bolum bolum)
    {
        if (!ModelState.IsValid)
        {
            await FakulteListesiniHazirlaAsync(bolum.FakulteId);
            return View(bolum);
        }

        var mevcut = await _db.Bolumler.FindAsync(bolum.BolumId);

        if (mevcut == null)
            return NotFound();

        mevcut.FakulteId    = bolum.FakulteId;      // bölüm başka fakülteye taşınabilir
        mevcut.BolumAdi     = bolum.BolumAdi;
        mevcut.BolumAdres   = bolum.BolumAdres;
        mevcut.BolumTelefon = bolum.BolumTelefon;
        mevcut.BolumEposta  = bolum.BolumEposta;
        mevcut.UpdatedDate  = DateTime.Now;

        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Bölüm güncellendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  6) SİLME ONAY SAYFASI
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Delete(long id)
    {
        var bolum = await _db.Bolumler
            .Include(b => b.Fakulte)
            .FirstOrDefaultAsync(b => b.BolumId == id);

        if (bolum == null)
            return NotFound();

        // Bağlı kayıt sayıları — kullanıcıya önceden göster
        ViewBag.OgrenciSayisi = await _db.Ogrenciler.CountAsync(o => o.BolumId == id);
        ViewBag.AkademisyenSayisi = await _db.Akademisyenler.CountAsync(a => a.BolumId == id);

        return View(bolum);
    }

    // ══════════════════════════════════════════════════════
    //  7) SİLMEYİ ONAYLA
    // ══════════════════════════════════════════════════════
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(long id)
    {
        var bolum = await _db.Bolumler.FindAsync(id);

        if (bolum == null)
            return NotFound();

        int ogrenciSayisi = await _db.Ogrenciler.CountAsync(o => o.BolumId == id);
        int akademisyenSayisi = await _db.Akademisyenler.CountAsync(a => a.BolumId == id);

        if (ogrenciSayisi > 0 || akademisyenSayisi > 0)
        {
            TempData["Uyari"] = $"Bu bölümde {ogrenciSayisi} öğrenci ve " +
                                $"{akademisyenSayisi} akademisyen var. Önce onları taşıyın.";
            return RedirectToAction(nameof(Index));
        }

        bolum.AktifMi = false;
        bolum.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Bölüm silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  8) DETAY
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Details(long id)
    {
        // ⭐ Üç seviyeli yükleme: bölüm + fakültesi + öğrencileri + akademisyenleri
        var bolum = await _db.Bolumler
            .Include(b => b.Fakulte)
            .Include(b => b.Ogrenciler)
            .Include(b => b.Akademisyenler)
            .FirstOrDefaultAsync(b => b.BolumId == id);

        if (bolum == null)
            return NotFound();

        return View(bolum);
    }
}
```

### ⚠️ `OrderBy(b => b.Fakulte!.FakulteAd)` — ünlem ne?

```csharp
.OrderBy(b => b.Fakulte!.FakulteAd)
                       └─ "null olmadığını biliyorum"
```

`Fakulte` özelliği `Fakulte?` tipinde olduğu için derleyici "null olabilir" diye uyarır. `!` işareti uyarıyı susturur.

Burada güvenli çünkü:
- `Include` ile fakülteyi getirdik
- `FakulteId` zorunlu (`NOT NULL`), her bölümün fakültesi var

> ⚠️ `!` işaretini gelişigüzel kullanmayın. Gerçekten null olabilecek bir yerde kullanırsanız uygulama çalışma zamanında çöker.

---

## ⚠️ Navigasyon özelliği tuzağı — mutlaka deney yaptır

`Bolum` modelindeki bu satıra dön:

```csharp
public Fakulte? Fakulte { get; set; }
```

**Deney:** Soru işaretini sil, `public Fakulte Fakulte { get; set; }` yap. Bölüm eklemeyi dene.

**Sonuç:** Form her seferinde geri geliyor, kaydetmiyor. Neden?

Formdan gelen veri:
```
FakulteId=1&BolumAdi=Yazılım&BolumAdres=...
```

`Fakulte` nesnesi gelmiyor — sadece id geliyor. `?` olmadığında ASP.NET Core bunu "zorunlu alan boş" sayıyor ve `ModelState.IsValid` **false** oluyor.

**Hatayı görmek için** view'a geçici olarak ekle:

```html
<div asp-validation-summary="All" class="alert alert-danger"></div>
```

`ModelOnly` yerine `All` yazınca tüm alan hataları görünür. Listede "Fakulte alanı gereklidir" yazdığını göreceksiniz.

> Sonra `?` işaretini geri koy ve `ModelOnly`'ye dön.

**İkinci çözüm** (bilinmesi iyi): Controller'da o alanı doğrulamadan çıkarmak.

```csharp
ModelState.Remove(nameof(Bolum.Fakulte));

if (!ModelState.IsValid) { ... }
```

Ama `?` kullanmak daha temiz — modelin kendisi doğru bilgiyi taşımış oluyor.

---

## ⌨️ Adım 2: View'lar

### Index.cshtml

```html
@model List<UBYS.Models.Bolum>
@{
    ViewData["Title"] = "Bölümler";
}

@if (TempData["Basarili"] != null)
{
    <div class="alert alert-success alert-dismissible fade show">
        <i class="bi bi-check-circle"></i> @TempData["Basarili"]
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
}

@if (TempData["Uyari"] != null)
{
    <div class="alert alert-warning alert-dismissible fade show">
        <i class="bi bi-exclamation-triangle"></i> @TempData["Uyari"]
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
}

<div class="card border-0 shadow-sm">
    <div class="card-header bg-white d-flex justify-content-between align-items-center">
        <span class="fw-semibold">Kayıtlı bölümler (@Model.Count)</span>
        <a asp-action="Create" class="btn btn-primary btn-sm">
            <i class="bi bi-plus-lg"></i> Yeni bölüm
        </a>
    </div>

    <div class="card-body p-0">
        @if (Model.Count == 0)
        {
            <div class="text-center text-muted py-5">
                <i class="bi bi-inbox fs-1 d-block mb-2"></i>
                <p class="mb-3">Henüz bölüm yok. Önce en az bir fakülte oluşturun.</p>
                <a asp-action="Create" class="btn btn-sm btn-primary">İlk bölümü ekle</a>
            </div>
        }
        else
        {
            <table class="table table-hover align-middle mb-0">
                <thead class="table-light">
                    <tr>
                        <th>Bölüm adı</th>
                        <th>Fakülte</th>
                        <th>Telefon</th>
                        <th>E-posta</th>
                        <th class="text-end">İşlemler</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var bolum in Model)
                    {
                        <tr>
                            <td>
                                <a asp-action="Details" asp-route-id="@bolum.BolumId"
                                   class="fw-semibold text-decoration-none">
                                    @bolum.BolumAdi
                                </a>
                            </td>
                            <td>
                                @* ⭐ Include sayesinde nokta ile gidiyoruz.
                                     ?. kullanıyoruz: Include unutulursa çökmesin, boş görünsün. *@
                                <span class="badge bg-secondary">@bolum.Fakulte?.FakulteAd</span>
                            </td>
                            <td>@bolum.BolumTelefon</td>
                            <td>@bolum.BolumEposta</td>
                            <td class="text-end text-nowrap">
                                <a asp-action="Edit" asp-route-id="@bolum.BolumId"
                                   class="btn btn-sm btn-outline-primary">
                                    <i class="bi bi-pencil"></i>
                                </a>
                                <a asp-action="Delete" asp-route-id="@bolum.BolumId"
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
@model UBYS.Models.Bolum
@{
    ViewData["Title"] = "Yeni bölüm";
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Create" method="post">

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <div class="mb-3">
                        <label asp-for="FakulteId" class="form-label"></label>
                        <select asp-for="FakulteId" asp-items="ViewBag.Fakulteler"
                                class="form-select">
                            <option value="">-- Fakülte seçin --</option>
                        </select>
                        <span asp-validation-for="FakulteId" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="BolumAdi" class="form-label"></label>
                        <input asp-for="BolumAdi" class="form-control" />
                        <span asp-validation-for="BolumAdi" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="BolumAdres" class="form-label"></label>
                        <textarea asp-for="BolumAdres" class="form-control" rows="3"></textarea>
                        <span asp-validation-for="BolumAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="BolumTelefon" class="form-label"></label>
                            <input asp-for="BolumTelefon" class="form-control" />
                            <span asp-validation-for="BolumTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="BolumEposta" class="form-label"></label>
                            <input asp-for="BolumEposta" class="form-control"
                                   placeholder="bolum@@okul.edu.tr" />
                            <span asp-validation-for="BolumEposta" class="text-danger small"></span>
                        </div>
                    </div>

                    <hr />
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-lg"></i> Bölümü kaydet
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

```html
@model UBYS.Models.Bolum
@{
    ViewData["Title"] = "Bölümü düzenle";
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Edit" method="post">

                    @* Sadece id — diğer sistem alanlarını controller okuyor *@
                    <input type="hidden" asp-for="BolumId" />

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <div class="mb-3">
                        <label asp-for="FakulteId" class="form-label"></label>
                        @* Controller'da FakulteListesiniHazirlaAsync(bolum.FakulteId)
                           çağrıldığı için mevcut fakülte seçili gelir *@
                        <select asp-for="FakulteId" asp-items="ViewBag.Fakulteler"
                                class="form-select">
                            <option value="">-- Fakülte seçin --</option>
                        </select>
                        <span asp-validation-for="FakulteId" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="BolumAdi" class="form-label"></label>
                        <input asp-for="BolumAdi" class="form-control" />
                        <span asp-validation-for="BolumAdi" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="BolumAdres" class="form-label"></label>
                        <textarea asp-for="BolumAdres" class="form-control" rows="3"></textarea>
                        <span asp-validation-for="BolumAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="BolumTelefon" class="form-label"></label>
                            <input asp-for="BolumTelefon" class="form-control" />
                            <span asp-validation-for="BolumTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="BolumEposta" class="form-label"></label>
                            <input asp-for="BolumEposta" class="form-control" />
                            <span asp-validation-for="BolumEposta" class="text-danger small"></span>
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
@model UBYS.Models.Bolum
@{
    ViewData["Title"] = "Bölümü sil";
    int ogrenciSayisi = ViewBag.OgrenciSayisi ?? 0;
    int akademisyenSayisi = ViewBag.AkademisyenSayisi ?? 0;
    bool silinebilir = (ogrenciSayisi == 0 && akademisyenSayisi == 0);
}

<div class="row">
    <div class="col-lg-7">
        <div class="card border-danger shadow-sm">
            <div class="card-header bg-danger text-white">
                <i class="bi bi-exclamation-triangle"></i> Silme onayı
            </div>
            <div class="card-body">

                <dl class="row mb-4">
                    <dt class="col-sm-4">Bölüm adı</dt>
                    <dd class="col-sm-8 fw-semibold">@Model.BolumAdi</dd>
                    <dt class="col-sm-4">Fakülte</dt>
                    <dd class="col-sm-8">@Model.Fakulte?.FakulteAd</dd>
                    <dt class="col-sm-4">Telefon</dt>
                    <dd class="col-sm-8">@Model.BolumTelefon</dd>
                    <dt class="col-sm-4">E-posta</dt>
                    <dd class="col-sm-8">@Model.BolumEposta</dd>
                </dl>

                @if (!silinebilir)
                {
                    <div class="alert alert-warning">
                        <i class="bi bi-x-circle"></i>
                        Bu bölümde <strong>@ogrenciSayisi öğrenci</strong> ve
                        <strong>@akademisyenSayisi akademisyen</strong> kayıtlı.
                        Silmeden önce onları başka bölüme taşımalısınız.
                    </div>
                    <a asp-action="Index" class="btn btn-outline-secondary">Listeye dön</a>
                }
                else
                {
                    <div class="alert alert-info small">
                        Kayıt tamamen silinmez, pasif duruma alınır.
                    </div>

                    <form asp-action="Delete" method="post">
                        <input type="hidden" asp-for="BolumId" name="id" />
                        <button type="submit" class="btn btn-danger">
                            <i class="bi bi-trash"></i> Evet, sil
                        </button>
                        <a asp-action="Index" class="btn btn-outline-secondary">Vazgeç</a>
                    </form>
                }

            </div>
        </div>
    </div>
</div>
```

### Details.cshtml

```html
@model UBYS.Models.Bolum
@{
    ViewData["Title"] = "Bölüm detayı";
}

<div class="row">
    <div class="col-lg-9">
        <div class="card border-0 shadow-sm">

            <div class="card-body border-bottom">
                <h5 class="mb-1">@Model.BolumAdi</h5>
                <span class="badge bg-secondary">@Model.Fakulte?.FakulteAd</span>
            </div>

            <div class="card-body border-bottom">
                <dl class="row mb-0">
                    <dt class="col-sm-3 fw-normal text-muted">Adres</dt>
                    <dd class="col-sm-9">@Model.BolumAdres</dd>
                    <dt class="col-sm-3 fw-normal text-muted">Telefon</dt>
                    <dd class="col-sm-9">@Model.BolumTelefon</dd>
                    <dt class="col-sm-3 fw-normal text-muted">E-posta</dt>
                    <dd class="col-sm-9">@Model.BolumEposta</dd>
                </dl>
            </div>

            <div class="row g-0">

                @* ⭐ Include ile gelen öğrenciler *@
                <div class="col-md-6 border-end">
                    <div class="card-body">
                        <h6 class="text-muted mb-3">
                            Öğrenciler (@Model.Ogrenciler.Count)
                        </h6>

                        @if (Model.Ogrenciler.Count == 0)
                        {
                            <p class="text-muted small mb-0">Kayıtlı öğrenci yok.</p>
                        }
                        else
                        {
                            <ul class="list-group list-group-flush">
                                @foreach (var o in Model.Ogrenciler.OrderBy(x => x.OgrenciAd))
                                {
                                    <li class="list-group-item px-0 d-flex justify-content-between">
                                        <span class="small">@o.TamAd</span>
                                        <span class="badge bg-info">@o.OgrenciSinif. sınıf</span>
                                    </li>
                                }
                            </ul>
                        }
                    </div>
                </div>

                @* ⭐ Include ile gelen akademisyenler *@
                <div class="col-md-6">
                    <div class="card-body">
                        <h6 class="text-muted mb-3">
                            Akademisyenler (@Model.Akademisyenler.Count)
                        </h6>

                        @if (Model.Akademisyenler.Count == 0)
                        {
                            <p class="text-muted small mb-0">Kayıtlı akademisyen yok.</p>
                        }
                        else
                        {
                            <ul class="list-group list-group-flush">
                                @foreach (var a in Model.Akademisyenler.OrderBy(x => x.AkademisyenAd))
                                {
                                    <li class="list-group-item px-0">
                                        <span class="small">@a.TamAd</span>
                                    </li>
                                }
                            </ul>
                        }
                    </div>
                </div>

            </div>

            <div class="card-footer bg-white">
                <a asp-action="Edit" asp-route-id="@Model.BolumId" class="btn btn-primary btn-sm">
                    <i class="bi bi-pencil"></i> Düzenle
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

## ▶️ Deney: Include'u sil, ne oluyor gör

Bu deneyi **mutlaka** yaptır — `Include`'un ne işe yaradığı ancak böyle anlaşılır.

**1.** `Index` metodundaki `.Include(b => b.Fakulte)` satırını sil.
**2.** Sayfayı aç.

Sonuç: Fakülte sütunu **boş.** Çökme yok çünkü view'da `?.` kullandık. `?.` yerine `.` yazsaydık `NullReferenceException` alırdık.

**3.** Konsoldaki SQL'e bak — `JOIN` yok:
```sql
SELECT [b].[BolumId], [b].[BolumAdi], ... FROM [Bolumler] AS [b]
```

**4.** `Include`'u geri ekle, SQL'e tekrar bak — `INNER JOIN` geldi.

> **Ders çıkarımı:** *"Navigasyon özelliği tanımlamak, verinin geleceği anlamına gelmez. İstemek gerekir."*

---

## ▶️ Test senaryosu

| Test | Beklenen |
|---|---|
| Bölüm listesi | Fakülte adları rozette görünüyor |
| Konsoldaki SQL | `INNER JOIN Fakulteler` var |
| Yeni bölüm | Açılır listede fakülteler dolu |
| Fakülte seçmeden kaydet | "Fakülte seçmelisiniz" hatası |
| Hatalı gönderimden sonra | **Açılır liste hâlâ dolu** |
| Düzenle | Mevcut fakülte seçili geliyor |
| Bölümü başka fakülteye taşı | Listede yeni fakülte görünüyor |
| Detay sayfası | Öğrenci ve akademisyen listeleri |
| Öğrencisi olan bölümü sil | Uyarı, silinmiyor |
| Boş bölümü sil | Siliniyor (`AktifMi = 0`) |

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `NullReferenceException` — `bolum.Fakulte.FakulteAd` | `Include` yok | `.Include(b => b.Fakulte)` ekle |
| Fakülte sütunu boş görünüyor | Aynı sebep | Aynı çözüm |
| Form hep geçersiz, hata belli değil | Navigasyon özelliğinde `?` yok | `Fakulte?` yap, ya da `ModelState.Remove` |
| Açılır liste boş → çökme | POST'ta liste hazırlanmamış | `ModelState` bloğuna ekle |
| `The entity is already being tracked` | Aynı kayıt iki kez çekilmiş | Tek seferde çek |
| Bölüm eklerken FK hatası | Olmayan `FakulteId` | Açılır listeyi kontrol et |
| `Include` sonrası liste yavaş | Gereksiz veri çekiliyor | Sadece gerekeni `Select` ile al (Modül 8) |
| Silinen fakültenin bölümü listede | Query filter ilişkiye de uygulanıyor | Normal davranış; `IgnoreQueryFilters` ile görülebilir |

---

## ✏️ Öğrenci alıştırması

1. Bölüm CRUD'unu tamamla, test senaryolarını geçir.
2. `Include` deneyini yap, iki SQL'i yan yana koy, farkı yaz.
3. Fakülte listesine "Bölüm sayısı" sütunu ekle.
   İpucu: `_db.Fakulteler.Include(f => f.Bolumler)` ve `fakulte.Bolumler.Count`
4. `Details` sayfasında öğrencileri sınıfa göre grupla.
5. **Düşünme soruları:**
   - `Include` ile `ThenInclude` arasındaki fark nedir? Öğrenci listesinde fakülte adını göstermek isteseydik hangisi gerekirdi?
   - Detay sayfasında üç `Include` var. Bu tek bir SQL mi üretiyor, üç mü? Konsola bak.
   - Bir bölümün 5000 öğrencisi olsaydı, `Include(b => b.Ogrenciler)` iyi bir fikir olur muydu?

---

👉 Sonraki: [`07-ogrenci-crud.md`](07-ogrenci-crud.md)
