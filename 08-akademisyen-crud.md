# Modül 7 — Akademisyen CRUD

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Dördüncü ve son CRUD. **Yeni kavram yok** — kalıbın pekişmesi için.

Öğrenci CRUD'undan tek farkı: `sinif` alanı yok, `Unvan` alanı var (isteğe bağlı).

> **Ders formatı:** Kod tam olarak verilmiştir. Öğrenci CRUD'unu açıp yan yana koyun, farkları beraber bulun. Bu, "kalıbı gördüm" hissini pekiştirir.

---

## 🗣️ Derse giriş — 10 dakika

Tahtaya iki sınıfı yan yana yaz, farkları öğrencilere buldur:

```
     Ogrenci                        Akademisyen
─────────────────────         ─────────────────────
 OgrenciId                     AkademisyenId
 BolumId                       BolumId
 OgrenciAd                     AkademisyenAd
 OgrenciSoyad                  AkademisyenSoyad
 OgrenciSinif          ←──── ✂️ YOK
                       ────→  Unvan  ⭐ YENİ (nullable)
 OgrenciDogumTarihi            AkademisyenDogumTarihi
 OgrenciCinsiyet               AkademisyenCinsiyet
 ...                           ...
```

**Sor:** *"`Unvan` alanı `string?` tipinde. Bu ne demek ve veritabanında ne oluyor?"*
→ NULL olabilir, sütun `NULL` olarak oluşuyor, `[Required]` yok.

---

## ⌨️ Adım 1: Controller

`Controllers/AkademisyenController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.Data.SqlClient;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class AkademisyenController : Controller
{
    private readonly UbysDbContext _db;

    public AkademisyenController(UbysDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: bölüm açılır listesi
    //  (OgrenciController'daki ile birebir aynı)
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
                Ad = b.BolumAdi + " (" + b.Fakulte!.FakulteAd + ")"
            })
            .ToListAsync();

        ViewBag.Bolumler = new SelectList(bolumler, "BolumId", "Ad", secili);
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: benzersizlik ön kontrolü
    // ══════════════════════════════════════════════════════
    private async Task BenzersizlikKontrolAsync(Akademisyen akademisyen, long guncellenenId = 0)
    {
        bool epostaVar = await _db.Akademisyenler
            .AnyAsync(a => a.AkademisyenEposta == akademisyen.AkademisyenEposta
                        && a.AkademisyenId != guncellenenId);

        if (epostaVar)
            ModelState.AddModelError(nameof(Akademisyen.AkademisyenEposta),
                "Bu e-posta adresi başka bir akademisyene kayıtlı.");

        bool tcVar = await _db.Akademisyenler
            .AnyAsync(a => a.AkademisyenTc == akademisyen.AkademisyenTc
                        && a.AkademisyenId != guncellenenId);

        if (tcVar)
            ModelState.AddModelError(nameof(Akademisyen.AkademisyenTc),
                "Bu TC kimlik numarası başka bir akademisyene kayıtlı.");
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: veritabanı hatasını kullanıcı diline çevir
    // ══════════════════════════════════════════════════════
    private void VeritabaniHatasiniIsle(DbUpdateException ex)
    {
        if (ex.InnerException is SqlException sqlEx &&
            (sqlEx.Number == 2601 || sqlEx.Number == 2627))
        {
            string mesaj = sqlEx.Message;

            if (mesaj.Contains("Eposta"))
                ModelState.AddModelError(nameof(Akademisyen.AkademisyenEposta),
                    "Bu e-posta adresi başka bir akademisyene kayıtlı.");
            else if (mesaj.Contains("Tc"))
                ModelState.AddModelError(nameof(Akademisyen.AkademisyenTc),
                    "Bu TC kimlik numarası başka bir akademisyene kayıtlı.");
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
        var akademisyenler = await _db.Akademisyenler
            .Include(a => a.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .OrderBy(a => a.AkademisyenAd)
            .ThenBy(a => a.AkademisyenSoyad)
            .ToListAsync();

        return View(akademisyenler);
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
    public async Task<IActionResult> Create(Akademisyen akademisyen)
    {
        await BenzersizlikKontrolAsync(akademisyen);

        if (!ModelState.IsValid)
        {
            await BolumListesiniHazirlaAsync(akademisyen.BolumId);
            return View(akademisyen);
        }

        akademisyen.CreatedDate = DateTime.Now;
        akademisyen.UpdatedDate = null;
        akademisyen.AktifMi = true;

        try
        {
            _db.Akademisyenler.Add(akademisyen);
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateException ex)
        {
            VeritabaniHatasiniIsle(ex);
            await BolumListesiniHazirlaAsync(akademisyen.BolumId);
            return View(akademisyen);
        }

        TempData["Basarili"] = $"{akademisyen.TamAd} kaydedildi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  4) DÜZENLEME FORMU
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Edit(long id)
    {
        var akademisyen = await _db.Akademisyenler.FindAsync(id);

        if (akademisyen == null)
            return NotFound();

        await BolumListesiniHazirlaAsync(akademisyen.BolumId);
        return View(akademisyen);
    }

    // ══════════════════════════════════════════════════════
    //  5) DÜZENLEMEYİ KAYDET
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(Akademisyen akademisyen)
    {
        await BenzersizlikKontrolAsync(akademisyen, akademisyen.AkademisyenId);

        if (!ModelState.IsValid)
        {
            await BolumListesiniHazirlaAsync(akademisyen.BolumId);
            return View(akademisyen);
        }

        var mevcut = await _db.Akademisyenler.FindAsync(akademisyen.AkademisyenId);

        if (mevcut == null)
            return NotFound();

        mevcut.BolumId                = akademisyen.BolumId;
        mevcut.AkademisyenAd          = akademisyen.AkademisyenAd;
        mevcut.AkademisyenSoyad       = akademisyen.AkademisyenSoyad;
        mevcut.Unvan                  = akademisyen.Unvan;
        mevcut.AkademisyenDogumTarihi = akademisyen.AkademisyenDogumTarihi;
        mevcut.AkademisyenCinsiyet    = akademisyen.AkademisyenCinsiyet;
        mevcut.AkademisyenAdres       = akademisyen.AkademisyenAdres;
        mevcut.AkademisyenTelefon     = akademisyen.AkademisyenTelefon;
        mevcut.AkademisyenEposta      = akademisyen.AkademisyenEposta;
        mevcut.AkademisyenTc          = akademisyen.AkademisyenTc;
        mevcut.UpdatedDate            = DateTime.Now;

        try
        {
            await _db.SaveChangesAsync();
        }
        catch (DbUpdateException ex)
        {
            VeritabaniHatasiniIsle(ex);
            await BolumListesiniHazirlaAsync(akademisyen.BolumId);
            return View(akademisyen);
        }

        TempData["Basarili"] = $"{akademisyen.TamAd} güncellendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  6) SİLME ONAY SAYFASI
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Delete(long id)
    {
        var akademisyen = await _db.Akademisyenler
            .Include(a => a.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .FirstOrDefaultAsync(a => a.AkademisyenId == id);

        if (akademisyen == null)
            return NotFound();

        return View(akademisyen);
    }

    // ══════════════════════════════════════════════════════
    //  7) SİLMEYİ ONAYLA
    // ══════════════════════════════════════════════════════
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(long id)
    {
        var akademisyen = await _db.Akademisyenler.FindAsync(id);

        if (akademisyen == null)
            return NotFound();

        akademisyen.AktifMi = false;
        akademisyen.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Akademisyen silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  8) DETAY
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Details(long id)
    {
        var akademisyen = await _db.Akademisyenler
            .Include(a => a.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .FirstOrDefaultAsync(a => a.AkademisyenId == id);

        if (akademisyen == null)
            return NotFound();

        return View(akademisyen);
    }

    // ══════════════════════════════════════════════════════
    //  9) CSV DIŞA AKTARMA
    //
    //  İki Türkiye'ye özgü detay:
    //    • Türk Excel'i ayraç olarak ; bekler, , değil
    //    • UTF-8 BOM olmadan Türkçe karakterler bozuk açılır
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> CsvIndir()
    {
        var liste = await _db.Akademisyenler
            .Include(a => a.Bolum)
                .ThenInclude(b => b!.Fakulte)
            .OrderBy(a => a.AkademisyenAd)
            .Select(a => new
            {
                a.Unvan,
                a.AkademisyenAd,
                a.AkademisyenSoyad,
                BolumAdi = a.Bolum!.BolumAdi,
                FakulteAd = a.Bolum!.Fakulte!.FakulteAd,
                a.AkademisyenTelefon,
                a.AkademisyenEposta
            })
            .ToListAsync();

        var sb = new System.Text.StringBuilder();
        sb.AppendLine("Unvan;Ad;Soyad;Bolum;Fakulte;Telefon;Eposta");

        foreach (var a in liste)
        {
            sb.AppendLine($"{a.Unvan};{a.AkademisyenAd};{a.AkademisyenSoyad};" +
                          $"{a.BolumAdi};{a.FakulteAd};" +
                          $"{a.AkademisyenTelefon};{a.AkademisyenEposta}");
        }

        var bom = new byte[] { 0xEF, 0xBB, 0xBF };
        var icerik = System.Text.Encoding.UTF8.GetBytes(sb.ToString());
        var dosya = bom.Concat(icerik).ToArray();

        return File(dosya, "text/csv", "akademisyenler.csv");
    }
}
```

---

## ⌨️ Adım 2: View'lar

### Index.cshtml

```html
@model List<UBYS.Models.Akademisyen>
@{
    ViewData["Title"] = "Akademisyenler";
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
        <span class="fw-semibold">Kayıtlı akademisyenler (@Model.Count)</span>
        <div>
            <a asp-action="CsvIndir" class="btn btn-outline-secondary btn-sm">
                <i class="bi bi-download"></i> CSV
            </a>
            <a asp-action="Create" class="btn btn-primary btn-sm">
                <i class="bi bi-plus-lg"></i> Yeni akademisyen
            </a>
        </div>
    </div>

    <div class="card-body p-0">
        @if (Model.Count == 0)
        {
            <div class="text-center text-muted py-5">
                <i class="bi bi-inbox fs-1 d-block mb-2"></i>
                <p class="mb-3">Henüz akademisyen yok.</p>
                <a asp-action="Create" class="btn btn-sm btn-primary">İlk kaydı ekle</a>
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
                        <th class="text-center">Yaş</th>
                        <th>TC</th>
                        <th>İletişim</th>
                        <th class="text-end">İşlemler</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var a in Model)
                    {
                        @{
                            int yas = DateTime.Today.Year - a.AkademisyenDogumTarihi.Year;
                            if (a.AkademisyenDogumTarihi.Date > DateTime.Today.AddYears(-yas))
                                yas--;

                            string tcMaskeli = a.AkademisyenTc.Length >= 3
                                ? a.AkademisyenTc.Substring(0, 3) + "********"
                                : "***********";
                        }

                        <tr>
                            <td>
                                <a asp-action="Details" asp-route-id="@a.AkademisyenId"
                                   class="fw-semibold text-decoration-none">
                                    @a.TamAd
                                </a>
                                <div class="text-muted small">@a.AkademisyenCinsiyet</div>
                            </td>
                            <td>
                                <span class="badge bg-secondary">@a.Bolum?.BolumAdi</span>
                            </td>
                            <td class="small text-muted">@a.Bolum?.Fakulte?.FakulteAd</td>
                            <td class="text-center">@yas</td>
                            <td><code class="small">@tcMaskeli</code></td>
                            <td class="small">
                                <div>@a.AkademisyenTelefon</div>
                                <div class="text-muted">@a.AkademisyenEposta</div>
                            </td>
                            <td class="text-end text-nowrap">
                                <a asp-action="Edit" asp-route-id="@a.AkademisyenId"
                                   class="btn btn-sm btn-outline-primary">
                                    <i class="bi bi-pencil"></i>
                                </a>
                                <a asp-action="Delete" asp-route-id="@a.AkademisyenId"
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
@model UBYS.Models.Akademisyen
@{
    ViewData["Title"] = "Yeni akademisyen";
}

<div class="row">
    <div class="col-lg-10">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Create" method="post">

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <h6 class="text-muted mb-3">Kişisel bilgiler</h6>

                    <div class="row">
                        <div class="col-md-3 mb-3">
                            <label asp-for="Unvan" class="form-label"></label>
                            @* ⭐ Unvan zorunlu DEĞİL — boş seçenek var *@
                            <select asp-for="Unvan" class="form-select">
                                <option value="">— Yok —</option>
                                <option value="Prof. Dr.">Prof. Dr.</option>
                                <option value="Doç. Dr.">Doç. Dr.</option>
                                <option value="Dr. Öğr. Üyesi">Dr. Öğr. Üyesi</option>
                                <option value="Öğr. Gör.">Öğr. Gör.</option>
                                <option value="Arş. Gör.">Arş. Gör.</option>
                            </select>
                            <span asp-validation-for="Unvan" class="text-danger small"></span>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label asp-for="AkademisyenAd" class="form-label"></label>
                            <input asp-for="AkademisyenAd" class="form-control" autofocus />
                            <span asp-validation-for="AkademisyenAd" class="text-danger small"></span>
                        </div>

                        <div class="col-md-5 mb-3">
                            <label asp-for="AkademisyenSoyad" class="form-label"></label>
                            <input asp-for="AkademisyenSoyad" class="form-control" />
                            <span asp-validation-for="AkademisyenSoyad" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenDogumTarihi" class="form-label"></label>
                            <input asp-for="AkademisyenDogumTarihi" type="date" class="form-control" />
                            <span asp-validation-for="AkademisyenDogumTarihi" class="text-danger small"></span>
                        </div>

                        <div class="col-md-6 mb-3">
                            <label class="form-label">Cinsiyet</label>
                            <div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="AkademisyenCinsiyet" value="Kadın" id="cinsiyetK" />
                                    <label class="form-check-label" for="cinsiyetK">Kadın</label>
                                </div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="AkademisyenCinsiyet" value="Erkek" id="cinsiyetE" />
                                    <label class="form-check-label" for="cinsiyetE">Erkek</label>
                                </div>
                            </div>
                            <span asp-validation-for="AkademisyenCinsiyet" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="AkademisyenTc" class="form-label"></label>
                        <input asp-for="AkademisyenTc" class="form-control" maxlength="11"
                               placeholder="11 haneli TC kimlik numarası" />
                        <span asp-validation-for="AkademisyenTc" class="text-danger small"></span>
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
                        <label asp-for="AkademisyenAdres" class="form-label"></label>
                        <textarea asp-for="AkademisyenAdres" class="form-control" rows="2"></textarea>
                        <span asp-validation-for="AkademisyenAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenTelefon" class="form-label"></label>
                            <input asp-for="AkademisyenTelefon" class="form-control"
                                   placeholder="05321234567" />
                            <span asp-validation-for="AkademisyenTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenEposta" class="form-label"></label>
                            <input asp-for="AkademisyenEposta" class="form-control"
                                   placeholder="ad.soyad@@okul.edu.tr" />
                            <span asp-validation-for="AkademisyenEposta" class="text-danger small"></span>
                        </div>
                    </div>

                    <hr />
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-lg"></i> Akademisyeni kaydet
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
@model UBYS.Models.Akademisyen
@{
    ViewData["Title"] = "Akademisyeni düzenle";
}

<div class="row">
    <div class="col-lg-10">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Edit" method="post">

                    <input type="hidden" asp-for="AkademisyenId" />

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <h6 class="text-muted mb-3">Kişisel bilgiler</h6>

                    <div class="row">
                        <div class="col-md-3 mb-3">
                            <label asp-for="Unvan" class="form-label"></label>
                            @* asp-for kullandığımız için mevcut unvan otomatik seçili gelir *@
                            <select asp-for="Unvan" class="form-select">
                                <option value="">— Yok —</option>
                                <option value="Prof. Dr.">Prof. Dr.</option>
                                <option value="Doç. Dr.">Doç. Dr.</option>
                                <option value="Dr. Öğr. Üyesi">Dr. Öğr. Üyesi</option>
                                <option value="Öğr. Gör.">Öğr. Gör.</option>
                                <option value="Arş. Gör.">Arş. Gör.</option>
                            </select>
                        </div>

                        <div class="col-md-4 mb-3">
                            <label asp-for="AkademisyenAd" class="form-label"></label>
                            <input asp-for="AkademisyenAd" class="form-control" />
                            <span asp-validation-for="AkademisyenAd" class="text-danger small"></span>
                        </div>

                        <div class="col-md-5 mb-3">
                            <label asp-for="AkademisyenSoyad" class="form-label"></label>
                            <input asp-for="AkademisyenSoyad" class="form-control" />
                            <span asp-validation-for="AkademisyenSoyad" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenDogumTarihi" class="form-label"></label>
                            <input asp-for="AkademisyenDogumTarihi" type="date" class="form-control" />
                            <span asp-validation-for="AkademisyenDogumTarihi" class="text-danger small"></span>
                        </div>

                        <div class="col-md-6 mb-3">
                            <label class="form-label">Cinsiyet</label>
                            <div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="AkademisyenCinsiyet" value="Kadın" id="cinsiyetK" />
                                    <label class="form-check-label" for="cinsiyetK">Kadın</label>
                                </div>
                                <div class="form-check form-check-inline">
                                    <input class="form-check-input" type="radio"
                                           asp-for="AkademisyenCinsiyet" value="Erkek" id="cinsiyetE" />
                                    <label class="form-check-label" for="cinsiyetE">Erkek</label>
                                </div>
                            </div>
                            <span asp-validation-for="AkademisyenCinsiyet" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="AkademisyenTc" class="form-label"></label>
                        <input asp-for="AkademisyenTc" class="form-control" maxlength="11" />
                        <span asp-validation-for="AkademisyenTc" class="text-danger small"></span>
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
                        <label asp-for="AkademisyenAdres" class="form-label"></label>
                        <textarea asp-for="AkademisyenAdres" class="form-control" rows="2"></textarea>
                        <span asp-validation-for="AkademisyenAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenTelefon" class="form-label"></label>
                            <input asp-for="AkademisyenTelefon" class="form-control" />
                            <span asp-validation-for="AkademisyenTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="AkademisyenEposta" class="form-label"></label>
                            <input asp-for="AkademisyenEposta" class="form-control" />
                            <span asp-validation-for="AkademisyenEposta" class="text-danger small"></span>
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
@model UBYS.Models.Akademisyen
@{
    ViewData["Title"] = "Akademisyeni sil";
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
                    <dt class="col-sm-4">TC kimlik no</dt>
                    <dd class="col-sm-8">@Model.AkademisyenTc</dd>
                    <dt class="col-sm-4">E-posta</dt>
                    <dd class="col-sm-8">@Model.AkademisyenEposta</dd>
                </dl>

                <div class="alert alert-info small">
                    <i class="bi bi-info-circle"></i>
                    Kayıt tamamen silinmez, pasif duruma alınır.
                </div>

                <form asp-action="Delete" method="post">
                    <input type="hidden" asp-for="AkademisyenId" name="id" />
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
@model UBYS.Models.Akademisyen
@{
    ViewData["Title"] = "Akademisyen detayı";

    int yas = DateTime.Today.Year - Model.AkademisyenDogumTarihi.Year;
    if (Model.AkademisyenDogumTarihi.Date > DateTime.Today.AddYears(-yas))
        yas--;

    string basHarfler = "";
    if (Model.AkademisyenAd.Length > 0) basHarfler += Model.AkademisyenAd[0];
    if (Model.AkademisyenSoyad.Length > 0) basHarfler += Model.AkademisyenSoyad[0];
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">

            <div class="card-body border-bottom">
                <div class="d-flex align-items-center gap-3">
                    <div class="rounded-circle bg-info text-white d-flex
                                align-items-center justify-content-center fw-bold"
                         style="width: 56px; height: 56px; font-size: 1.25rem;">
                        @basHarfler.ToUpper()
                    </div>
                    <div>
                        <h5 class="mb-1">@Model.TamAd</h5>
                        <span class="badge bg-secondary">@Model.Bolum?.BolumAdi</span>
                        <div class="text-muted small mt-1">@Model.Bolum?.Fakulte?.FakulteAd</div>
                    </div>
                </div>
            </div>

            <div class="card-body">
                <h6 class="text-muted mb-3">Kişisel bilgiler</h6>
                <dl class="row mb-4">
                    <dt class="col-sm-4 fw-normal text-muted">Unvan</dt>
                    <dd class="col-sm-8">
                        @* Unvan NULL olabilir *@
                        @(string.IsNullOrWhiteSpace(Model.Unvan) ? "—" : Model.Unvan)
                    </dd>
                    <dt class="col-sm-4 fw-normal text-muted">TC kimlik no</dt>
                    <dd class="col-sm-8">@Model.AkademisyenTc</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Doğum tarihi</dt>
                    <dd class="col-sm-8">
                        @Model.AkademisyenDogumTarihi.ToString("dd MMMM yyyy")
                        <span class="text-muted">(@yas yaşında)</span>
                    </dd>
                    <dt class="col-sm-4 fw-normal text-muted">Cinsiyet</dt>
                    <dd class="col-sm-8">@Model.AkademisyenCinsiyet</dd>
                </dl>

                <h6 class="text-muted mb-3">İletişim</h6>
                <dl class="row mb-4">
                    <dt class="col-sm-4 fw-normal text-muted">Telefon</dt>
                    <dd class="col-sm-8">@Model.AkademisyenTelefon</dd>
                    <dt class="col-sm-4 fw-normal text-muted">E-posta</dt>
                    <dd class="col-sm-8">@Model.AkademisyenEposta</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Adres</dt>
                    <dd class="col-sm-8">@Model.AkademisyenAdres</dd>
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
                <a asp-action="Edit" asp-route-id="@Model.AkademisyenId" class="btn btn-primary btn-sm">
                    <i class="bi bi-pencil"></i> Düzenle
                </a>
                <a asp-action="Delete" asp-route-id="@Model.AkademisyenId" class="btn btn-outline-danger btn-sm">
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

## 🗣️ Ders sonu — karşılaştırma

Dört CRUD bitti. Şimdi durup karşılaştırın.

**Tahtaya yaz ve öğrencilere doldurttur:**

| | ADO.NET sürümü | EF sürümü |
|---|---|---|
| Repository dosyası | 4 | 0 |
| Toplam satır (yaklaşık) | ~1200 | ~400 |
| Elle yazılan SQL | ~25 sorgu | 0 |
| `WHERE is_active` tekrarı | ~10 yer | 1 yer (query filter) |
| JOIN yazımı | Elle | `.Include()` |
| Satır → nesne çevirimi | Elle, her alan | Otomatik |

**Sonra üç soru sor:**

1. **"EF ile yazmak daha mı kolaydı?"** → Evet, belirgin şekilde.
2. **"Peki hiç ADO.NET öğrenmeseydik, bugün buradaki her şeyi anlar mıydık?"** → Hayır. `Include`'un JOIN olduğunu, query filter'ın `WHERE` eklediğini bilmezdik.
3. **"Hangi durumda ADO.NET'e geri dönerdiniz?"** → Çok karmaşık raporlama sorguları, toplu veri işleme, performansın kritik olduğu yerler. (EF'te bunun için `FromSql` var — Modül 11.)

---

## ▶️ Test senaryosu

| Test | Beklenen |
|---|---|
| Liste | Unvan + ad soyad birlikte |
| Unvan seçmeden kaydet | **Kaydolur** (zorunlu değil) |
| SSMS'te bak | `Unvan` sütunu NULL |
| Detayda unvansız kayıt | `—` görünüyor, çökme yok |
| Aynı TC ikinci kez | Kırmızı uyarı, çökme yok |
| Düzenle | Unvan seçili geliyor |
| CSV indir | Türkçe karakterler Excel'de düzgün |

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Unvan boş bırakılınca hata | `[Required]` konmuş | Kaldır, `string?` yeter |
| Unvan hep boş kaydediliyor | `<option value="">` seçili kalıyor | Normal, kullanıcı seçmemiş |
| `TamAd` sütunu tabloda oluşmuş | `[NotMapped]` yok | Ekle, migration üret |
| CSV Türkçe karakterler bozuk | BOM eksik | `0xEF, 0xBB, 0xBF` ekle |
| CSV tek sütunda açılıyor | Ayraç `,` kullanılmış | `;` kullan |
| Fakülte sütunu boş | `ThenInclude` yok | Ekle |

---

## ✏️ Öğrenci alıştırması

1. Akademisyen CRUD'unu tamamla.
2. Öğrenci ve akademisyen controller'larını yan yana aç. Kaç satır **birebir aynı**? Bu tekrarı azaltmanın bir yolu var mı? (Araştır: generic repository, base controller)
3. Unvana göre filtreleme ekle: listenin üstünde açılır liste, seçilince o unvandakiler gelsin.
4. Akademisyen listesini bölüme göre grupla (her bölüm bir başlık, altında akademisyenleri).
5. **Düşünme soruları:**
   - Dört controller'da `VeritabaniHatasiniIsle` metodu neredeyse aynı. Bunu tek yere taşımanın maliyeti/faydası ne olurdu?
   - `Unvan` alanını `string` (nullable olmayan) yapsaydık ve boş gönderilseydi ne olurdu?

---

👉 Sonraki: [`09-dashboard.md`](09-dashboard.md)
