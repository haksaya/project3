# Modül 4 — Fakülte CRUD

**Süre:** 2 ders saati

---

## 🎯 Bu derste ne yapacağız

İlk tam CRUD. **Yeni kavramlar: değişiklik takibi ve `SaveChanges()`.**

> ⭐ Bu modül kursun kalbi. Buradaki kalıp kalan üç tabloda tekrarlanacak.

---

## 📖 Kavram: Değişiklik takibi (change tracking)

EF'in en önemli ve en çok şaşırtan özelliği.

### ADO.NET'te güncelleme

```csharp
string sql = "UPDATE fakulte SET fakulte_ad = @ad WHERE fakulte_id = @id";
komut.Parameters.AddWithValue("@ad", fakulte.FakulteAd);
komut.Parameters.AddWithValue("@id", fakulte.FakulteId);
komut.ExecuteNonQuery();
```

**Biz söyledik:** "Şu sütunu, şu değere, şu satırda değiştir."

### EF'te güncelleme

```csharp
var fakulte = await _db.Fakulteler.FindAsync(5);
fakulte.FakulteAd = "Yeni Ad";
await _db.SaveChangesAsync();
```

**Hiç `UPDATE` yazmadık.** Nasıl oldu?

### Cevap: DbContext her şeyi izliyor

```
1. FindAsync(5)
   → EF kaydı çeker
   → Aynı zamanda "orijinal hâlinin fotoğrafını" saklar

        Bellekte:
        ┌─────────────────────────────────────────┐
        │ Fakulte #5                              │
        │   ORİJİNAL: FakulteAd = "Mühendislik"   │
        │   GÜNCEL  : FakulteAd = "Mühendislik"   │
        │   DURUM   : Unchanged                   │
        └─────────────────────────────────────────┘

2. fakulte.FakulteAd = "Yeni Ad"
   → EF fark eder

        ┌─────────────────────────────────────────┐
        │ Fakulte #5                              │
        │   ORİJİNAL: FakulteAd = "Mühendislik"   │
        │   GÜNCEL  : FakulteAd = "Yeni Ad"    ⭐ │
        │   DURUM   : Modified                    │
        └─────────────────────────────────────────┘

3. SaveChangesAsync()
   → EF farkı görür ve SADECE değişen sütun için UPDATE üretir:

        UPDATE [Fakulteler]
        SET [FakulteAd] = @p0
        WHERE [FakulteId] = @p1
```

> ⭐ **Dikkat:** EF sadece **değişen sütunu** günceller. Sekiz alanlı bir kayıtta tek alanı değiştirirseniz, `UPDATE`'te tek sütun olur. ADO.NET'te hepsini yazıyorduk.

### Nesne durumları

| Durum | Anlamı | `SaveChanges` ne yapar |
|---|---|---|
| `Unchanged` | Değişmedi | Hiçbir şey |
| `Added` | Yeni eklendi | `INSERT` |
| `Modified` | Değişti | `UPDATE` |
| `Deleted` | Silinecek | `DELETE` |
| `Detached` | Takip edilmiyor | Hiçbir şey |

### ⚠️ Bunun sonucu: takip edilmeyen nesne güncellenmez

```csharp
// ❌ ÇALIŞMAZ — bu nesne DbContext'ten gelmedi
var f = new Fakulte { FakulteId = 5, FakulteAd = "Yeni" };
await _db.SaveChangesAsync();     // hiçbir şey olmaz

// ✅ Çözüm 1: veritabanından çek, değiştir
var f = await _db.Fakulteler.FindAsync(5);
f.FakulteAd = "Yeni";
await _db.SaveChangesAsync();

// ✅ Çözüm 2: EF'e "bu nesneyi güncelle" de
_db.Fakulteler.Update(f);         // tüm sütunları günceller
await _db.SaveChangesAsync();
```

Formdan gelen nesne **takip edilmiyordur** (Detached). Bu yüzden `Edit` metodunda ikisinden birini yapmak zorundayız. Aşağıda birinci yolu kullanacağız ve nedenini açıklayacağız.

---

## 📖 Kavram: `SaveChanges()` bir işlemdir (transaction)

```csharp
_db.Add(fakulte1);
_db.Add(fakulte2);
_db.Remove(fakulte3);
await _db.SaveChangesAsync();     // ← üçü BİRLİKTE, tek transaction
```

Üçünden biri başarısız olursa **hiçbiri** kaydedilmez. Ya hepsi ya hiçbiri.

> ADO.NET'te bunu sağlamak için `SqlTransaction` yazmak gerekiyordu. EF'te bedava geliyor.

---

## ⌨️ Adım 1: Soft delete için global query filter ⭐

ADO.NET sürümünde her sorguya elle `WHERE is_active = '1'` yazıyorduk. Unutulduğu her yerde silinen kayıtlar geri geliyordu.

EF'te bunu **tek yerde** halledebiliriz.

`UbysDbContext.OnModelCreating` metoduna ekle (indekslerden sonra):

```csharp
        // ══════════════════════════════════════════════════════
        //  GLOBAL QUERY FILTER — soft delete
        //
        //  Bu satırlardan sonra, bu tablolara yapılan HER sorguya
        //  EF otomatik olarak "WHERE AktifMi = 1" ekler.
        //
        //  Artık .Where(f => f.AktifMi) yazmamıza gerek YOK.
        //  Unutma riski de ortadan kalkıyor.
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Fakulte>().HasQueryFilter(f => f.AktifMi);
        modelBuilder.Entity<Bolum>().HasQueryFilter(b => b.AktifMi);
        modelBuilder.Entity<Ogrenci>().HasQueryFilter(o => o.AktifMi);
        modelBuilder.Entity<Akademisyen>().HasQueryFilter(a => a.AktifMi);
        modelBuilder.Entity<Kullanici>().HasQueryFilter(k => k.AktifMi);
```

Migration üret ve uygula:
```bash
dotnet ef migrations add SoftDeleteFiltresi
dotnet ef database update
```

> **Not:** Query filter veritabanı şemasını değiştirmez, bu yüzden migration boş çıkabilir. Sorun değil — filtre çalışma zamanında uygulanır.

### Artık sorgular sadeleşiyor

```csharp
// ÖNCE
var liste = await _db.Fakulteler.Where(f => f.AktifMi).ToListAsync();

// SONRA — filtre otomatik ekleniyor
var liste = await _db.Fakulteler.ToListAsync();
```

Konsoldaki SQL'e bak — `WHERE [f].[AktifMi] = CAST(1 AS bit)` hâlâ orada.

### Filtreyi geçici olarak kapatmak

Pasif kayıtları görmek istersen:

```csharp
var pasifler = await _db.Fakulteler
    .IgnoreQueryFilters()               // ⭐ filtreyi atla
    .Where(f => !f.AktifMi)
    .ToListAsync();
```

> **Sınıfa sor:** *"ADO.NET projesinde `WHERE is_active = '1'` yazmayı kaç yerde unutma riskimiz vardı?"*
> Her repository'de 2-3 sorgu × 4 repository ≈ 10 yer. Şimdi tek satır.

---

## ⌨️ Adım 2: Controller — tam CRUD

`Controllers/FakulteController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class FakulteController : Controller
{
    private readonly UbysDbContext _db;

    public FakulteController(UbysDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  1) LİSTELEME
    //  GET: /Fakulte
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Index()
    {
        // Query filter sayesinde .Where(f => f.AktifMi) yazmıyoruz
        var fakulteler = await _db.Fakulteler
            .OrderBy(f => f.FakulteAd)
            .ToListAsync();

        return View(fakulteler);
    }

    // ══════════════════════════════════════════════════════
    //  2) YENİ KAYIT FORMU
    //  GET: /Fakulte/Create
    // ══════════════════════════════════════════════════════
    public IActionResult Create()
    {
        return View();
    }

    // ══════════════════════════════════════════════════════
    //  3) YENİ KAYDI KAYDET
    //  POST: /Fakulte/Create
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(Fakulte fakulte)
    {
        if (!ModelState.IsValid)
            return View(fakulte);

        // Sistem alanlarını biz dolduruyoruz — kullanıcıdan almıyoruz
        fakulte.CreatedDate = DateTime.Now;
        fakulte.UpdatedDate = null;
        fakulte.AktifMi = true;

        // ⭐ İKİ SATIRLIK EKLEME
        // ADO.NET'te 15 satırlık INSERT + parametreler vardı.
        _db.Fakulteler.Add(fakulte);        // durumu: Added
        await _db.SaveChangesAsync();       // INSERT çalışır

        // ⭐ Bonus: SaveChanges sonrası fakulte.FakulteId DOLMUŞ olur.
        //    EF, veritabanının ürettiği IDENTITY değerini nesneye geri yazar.
        //    ADO.NET'te bunun için SCOPE_IDENTITY() yazmamız gerekiyordu.

        TempData["Basarili"] = $"\"{fakulte.FakulteAd}\" kaydedildi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  4) DÜZENLEME FORMU
    //  GET: /Fakulte/Edit/5
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Edit(long id)
    {
        var fakulte = await _db.Fakulteler.FindAsync(id);

        // Kullanıcı adres çubuğuna olmayan bir id yazabilir
        if (fakulte == null)
            return NotFound();

        return View(fakulte);
    }

    // ══════════════════════════════════════════════════════
    //  5) DÜZENLEMEYİ KAYDET
    //  POST: /Fakulte/Edit/5
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(Fakulte fakulte)
    {
        if (!ModelState.IsValid)
            return View(fakulte);

        // ⭐ ÖNCE VERİTABANINDAN ÇEK, SONRA DEĞİŞTİR
        //
        // Neden doğrudan _db.Update(fakulte) yazmıyoruz?
        //
        // Çünkü formda olmayan alanlar (CreatedDate, AktifMi) boş gelir.
        // Update() tüm sütunları yazar ve o alanları SIFIRLAR.
        //
        // Bu yöntemde sadece istediğimiz alanları değiştiriyoruz,
        // gerisine dokunmuyoruz. Daha güvenli.
        var mevcut = await _db.Fakulteler.FindAsync(fakulte.FakulteId);

        if (mevcut == null)
            return NotFound();

        mevcut.FakulteAd      = fakulte.FakulteAd;
        mevcut.FakulteAdres   = fakulte.FakulteAdres;
        mevcut.FakulteTelefon = fakulte.FakulteTelefon;
        mevcut.FakulteEposta  = fakulte.FakulteEposta;
        mevcut.UpdatedDate    = DateTime.Now;

        // ⭐ Add/Update yok! "mevcut" nesnesi zaten takip ediliyor.
        //    EF neyin değiştiğini biliyor.
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Fakülte güncellendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  6) SİLME ONAY SAYFASI
    //  GET: /Fakulte/Delete/5
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Delete(long id)
    {
        var fakulte = await _db.Fakulteler.FindAsync(id);

        if (fakulte == null)
            return NotFound();

        // Silinebilir mi? Kullanıcıya ÖNCEDEN söyleyelim
        ViewBag.BolumSayisi = await _db.Bolumler
            .CountAsync(b => b.FakulteId == id);

        return View(fakulte);
    }

    // ══════════════════════════════════════════════════════
    //  7) SİLMEYİ ONAYLA — soft delete
    //  POST: /Fakulte/Delete/5
    // ══════════════════════════════════════════════════════
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(long id)
    {
        var fakulte = await _db.Fakulteler.FindAsync(id);

        if (fakulte == null)
            return NotFound();

        // İlişkili kayıt kontrolü
        int bolumSayisi = await _db.Bolumler.CountAsync(b => b.FakulteId == id);

        if (bolumSayisi > 0)
        {
            TempData["Uyari"] = $"Bu fakültede {bolumSayisi} bölüm var. " +
                                 "Önce bölümleri silmelisiniz.";
            return RedirectToAction(nameof(Index));
        }

        // ⭐ SOFT DELETE
        // _db.Remove(fakulte) yazsaydık gerçekten SİLİNİRDİ.
        // Biz sadece işaretliyoruz. Query filter onu listeden gizleyecek.
        fakulte.AktifMi = false;
        fakulte.UpdatedDate = DateTime.Now;

        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Fakülte silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  8) DETAY
    //  GET: /Fakulte/Details/5
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Details(long id)
    {
        // ⭐ Include ile bölümleri de getiriyoruz
        // (Modül 5'te ayrıntılı anlatacağız)
        var fakulte = await _db.Fakulteler
            .Include(f => f.Bolumler)
            .FirstOrDefaultAsync(f => f.FakulteId == id);

        if (fakulte == null)
            return NotFound();

        return View(fakulte);
    }

    // ══════════════════════════════════════════════════════
    //  9) PASİF KAYITLAR
    //  GET: /Fakulte/Pasifler
    // ══════════════════════════════════════════════════════
    public async Task<IActionResult> Pasifler()
    {
        var pasifler = await _db.Fakulteler
            .IgnoreQueryFilters()            // ⭐ soft delete filtresini atla
            .Where(f => !f.AktifMi)
            .OrderBy(f => f.FakulteAd)
            .ToListAsync();

        return View(pasifler);
    }

    // ══════════════════════════════════════════════════════
    //  10) GERİ GETİR
    //  POST: /Fakulte/GeriAl/5
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> GeriAl(long id)
    {
        // Pasif kaydı bulmak için filtreyi atlamalıyız,
        // yoksa FindAsync bile onu bulamaz
        var fakulte = await _db.Fakulteler
            .IgnoreQueryFilters()
            .FirstOrDefaultAsync(f => f.FakulteId == id);

        if (fakulte == null)
            return NotFound();

        fakulte.AktifMi = true;
        fakulte.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Fakülte geri getirildi.";
        return RedirectToAction(nameof(Pasifler));
    }
}
```

### 📖 `nameof(Index)` nedir?

```csharp
return RedirectToAction(nameof(Index));   // "Index" metin olarak üretilir
```

`"Index"` yazmakla aynı sonucu verir. Farkı: metodun adını değiştirirseniz derleyici bunu yakalar. Metin yazsaydınız sessizce bozulurdu.

### 📖 Neden `Edit`'te önce çekiyoruz?

Bu, öğrencinin en çok soracağı soru. Üç yolu karşılaştır:

```csharp
// ── YOL 1: Çek, değiştir  (bizim seçimimiz) ────────────
var mevcut = await _db.Fakulteler.FindAsync(fakulte.FakulteId);
mevcut.FakulteAd = fakulte.FakulteAd;
// ... sadece istediğimiz alanlar
await _db.SaveChangesAsync();
// ✅ CreatedDate ve AktifMi korunur
// ✅ Sadece değişen sütunlar UPDATE'e girer
// ❌ Ekstra bir SELECT sorgusu çalışır

// ── YOL 2: Update ile ──────────────────────────────────
_db.Fakulteler.Update(fakulte);
await _db.SaveChangesAsync();
// ✅ Tek sorgu
// ❌ TÜM sütunlar yazılır
// ❌ Formda olmayan CreatedDate = 0001-01-01 olur, AktifMi = false olur!

// ── YOL 3: Update + gizli alanlar ──────────────────────
// View'a <input type="hidden" asp-for="CreatedDate" /> koyarsanız
// Yol 2 de çalışır. Ama kullanıcı F12 ile o alanı değiştirebilir.
```

> **Kural:** Kullanıcının değiştirmemesi gereken alanlar (kayıt tarihi, aktiflik, sahiplik) forma **hiç konmamalı** ve sunucuda veritabanından okunmalı. Yol 1 bunu sağlıyor.

---

## ⌨️ Adım 3: View'lar

### Index.cshtml

```html
@model List<UBYS.Models.Fakulte>
@{
    ViewData["Title"] = "Fakülteler";
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
        <span class="fw-semibold">Kayıtlı fakülteler (@Model.Count)</span>
        <div>
            <a asp-action="Pasifler" class="btn btn-outline-secondary btn-sm">
                <i class="bi bi-archive"></i> Pasifler
            </a>
            <a asp-action="Create" class="btn btn-primary btn-sm">
                <i class="bi bi-plus-lg"></i> Yeni fakülte
            </a>
        </div>
    </div>

    <div class="card-body p-0">
        @if (Model.Count == 0)
        {
            <div class="text-center text-muted py-5">
                <i class="bi bi-inbox fs-1 d-block mb-2"></i>
                <p class="mb-3">Henüz fakülte eklenmemiş.</p>
                <a asp-action="Create" class="btn btn-sm btn-primary">İlk kaydı oluştur</a>
            </div>
        }
        else
        {
            <table class="table table-hover align-middle mb-0">
                <thead class="table-light">
                    <tr>
                        <th>#</th>
                        <th>Fakülte adı</th>
                        <th>Telefon</th>
                        <th>E-posta</th>
                        <th>Kayıt tarihi</th>
                        <th class="text-end">İşlemler</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var fakulte in Model)
                    {
                        <tr>
                            <td class="text-muted">@fakulte.FakulteId</td>
                            <td>
                                <a asp-action="Details" asp-route-id="@fakulte.FakulteId"
                                   class="fw-semibold text-decoration-none">
                                    @fakulte.FakulteAd
                                </a>
                            </td>
                            <td>@fakulte.FakulteTelefon</td>
                            <td>@fakulte.FakulteEposta</td>
                            <td>@fakulte.CreatedDate.ToString("dd.MM.yyyy")</td>
                            <td class="text-end text-nowrap">
                                <a asp-action="Edit" asp-route-id="@fakulte.FakulteId"
                                   class="btn btn-sm btn-outline-primary">
                                    <i class="bi bi-pencil"></i>
                                </a>
                                <a asp-action="Delete" asp-route-id="@fakulte.FakulteId"
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
@model UBYS.Models.Fakulte
@{
    ViewData["Title"] = "Yeni fakülte";
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Create" method="post">

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <div class="mb-3">
                        <label asp-for="FakulteAd" class="form-label"></label>
                        <input asp-for="FakulteAd" class="form-control" autofocus />
                        <span asp-validation-for="FakulteAd" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="FakulteAdres" class="form-label"></label>
                        <textarea asp-for="FakulteAdres" class="form-control" rows="3"></textarea>
                        <span asp-validation-for="FakulteAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="FakulteTelefon" class="form-label"></label>
                            <input asp-for="FakulteTelefon" class="form-control"
                                   placeholder="02121234567" />
                            <span asp-validation-for="FakulteTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="FakulteEposta" class="form-label"></label>
                            <input asp-for="FakulteEposta" class="form-control"
                                   placeholder="ornek@@okul.edu.tr" />
                            <span asp-validation-for="FakulteEposta" class="text-danger small"></span>
                        </div>
                    </div>

                    <hr />
                    <button type="submit" class="btn btn-primary">
                        <i class="bi bi-check-lg"></i> Fakülteyi kaydet
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
@model UBYS.Models.Fakulte
@{
    ViewData["Title"] = "Fakülteyi düzenle";
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">
            <div class="card-body">
                <form asp-action="Edit" method="post">

                    @* ⭐ SADECE ID gizli alanı var.
                       CreatedDate ve AktifMi'yi TAŞIMIYORUZ — çünkü
                       controller onları veritabanından okuyor.
                       ADO.NET sürümünde üç gizli alan vardı, burada bir tane. *@
                    <input type="hidden" asp-for="FakulteId" />

                    <div asp-validation-summary="ModelOnly" class="alert alert-danger"></div>

                    <div class="mb-3">
                        <label asp-for="FakulteAd" class="form-label"></label>
                        <input asp-for="FakulteAd" class="form-control" />
                        <span asp-validation-for="FakulteAd" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="FakulteAdres" class="form-label"></label>
                        <textarea asp-for="FakulteAdres" class="form-control" rows="3"></textarea>
                        <span asp-validation-for="FakulteAdres" class="text-danger small"></span>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label asp-for="FakulteTelefon" class="form-label"></label>
                            <input asp-for="FakulteTelefon" class="form-control" />
                            <span asp-validation-for="FakulteTelefon" class="text-danger small"></span>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label asp-for="FakulteEposta" class="form-label"></label>
                            <input asp-for="FakulteEposta" class="form-control" />
                            <span asp-validation-for="FakulteEposta" class="text-danger small"></span>
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
@model UBYS.Models.Fakulte
@{
    ViewData["Title"] = "Fakülteyi sil";
    int bolumSayisi = ViewBag.BolumSayisi ?? 0;
}

<div class="row">
    <div class="col-lg-7">
        <div class="card border-danger shadow-sm">
            <div class="card-header bg-danger text-white">
                <i class="bi bi-exclamation-triangle"></i> Silme onayı
            </div>
            <div class="card-body">

                <dl class="row mb-4">
                    <dt class="col-sm-4">Fakülte adı</dt>
                    <dd class="col-sm-8 fw-semibold">@Model.FakulteAd</dd>
                    <dt class="col-sm-4">Telefon</dt>
                    <dd class="col-sm-8">@Model.FakulteTelefon</dd>
                    <dt class="col-sm-4">E-posta</dt>
                    <dd class="col-sm-8">@Model.FakulteEposta</dd>
                </dl>

                @if (bolumSayisi > 0)
                {
                    <div class="alert alert-warning">
                        <i class="bi bi-x-circle"></i>
                        Bu fakültede <strong>@bolumSayisi bölüm</strong> var.
                        Silmeden önce bölümleri kaldırmalısınız.
                    </div>
                    <a asp-action="Index" class="btn btn-outline-secondary">Listeye dön</a>
                }
                else
                {
                    <div class="alert alert-info small">
                        Kayıt tamamen silinmez, pasif duruma alınır.
                        "Pasifler" sayfasından geri getirilebilir.
                    </div>

                    <form asp-action="Delete" method="post">
                        <input type="hidden" asp-for="FakulteId" name="id" />
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
@model UBYS.Models.Fakulte
@{
    ViewData["Title"] = "Fakülte detayı";
}

<div class="row">
    <div class="col-lg-8">
        <div class="card border-0 shadow-sm">

            <div class="card-body border-bottom">
                <h5 class="mb-1">@Model.FakulteAd</h5>
                <span class="text-muted small">@Model.FakulteAdres</span>
            </div>

            <div class="card-body border-bottom">
                <dl class="row mb-0">
                    <dt class="col-sm-4 fw-normal text-muted">Telefon</dt>
                    <dd class="col-sm-8">@Model.FakulteTelefon</dd>
                    <dt class="col-sm-4 fw-normal text-muted">E-posta</dt>
                    <dd class="col-sm-8">@Model.FakulteEposta</dd>
                    <dt class="col-sm-4 fw-normal text-muted">Kayıt tarihi</dt>
                    <dd class="col-sm-8">@Model.CreatedDate.ToString("dd.MM.yyyy HH:mm")</dd>
                </dl>
            </div>

            @* ⭐ Include ile gelen bölümler *@
            <div class="card-body">
                <h6 class="text-muted mb-3">Bölümler (@Model.Bolumler.Count)</h6>

                @if (Model.Bolumler.Count == 0)
                {
                    <p class="text-muted small mb-0">Bu fakültede henüz bölüm yok.</p>
                }
                else
                {
                    <ul class="list-group list-group-flush">
                        @foreach (var bolum in Model.Bolumler)
                        {
                            <li class="list-group-item d-flex justify-content-between">
                                <span>@bolum.BolumAdi</span>
                                <span class="text-muted small">@bolum.BolumTelefon</span>
                            </li>
                        }
                    </ul>
                }
            </div>

            <div class="card-footer bg-white">
                <a asp-action="Edit" asp-route-id="@Model.FakulteId" class="btn btn-primary btn-sm">
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

### Pasifler.cshtml

```html
@model List<UBYS.Models.Fakulte>
@{
    ViewData["Title"] = "Pasif fakülteler";
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
        <span class="fw-semibold">Silinmiş fakülteler (@Model.Count)</span>
        <a asp-action="Index" class="btn btn-outline-secondary btn-sm">
            <i class="bi bi-arrow-left"></i> Aktif listeye dön
        </a>
    </div>

    <div class="card-body p-0">
        @if (Model.Count == 0)
        {
            <p class="text-muted text-center py-5 mb-0">Pasif kayıt yok.</p>
        }
        else
        {
            <table class="table table-hover align-middle mb-0">
                <thead class="table-light">
                    <tr>
                        <th>Fakülte adı</th>
                        <th>Silinme tarihi</th>
                        <th class="text-end">İşlem</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var f in Model)
                    {
                        <tr class="text-muted">
                            <td>@f.FakulteAd</td>
                            <td>@(f.UpdatedDate?.ToString("dd.MM.yyyy HH:mm") ?? "—")</td>
                            <td class="text-end">
                                <form asp-action="GeriAl" asp-route-id="@f.FakulteId"
                                      method="post" class="d-inline">
                                    <button type="submit" class="btn btn-sm btn-outline-success">
                                        <i class="bi bi-arrow-counterclockwise"></i> Geri getir
                                    </button>
                                </form>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        }
    </div>
</div>
```

---

## ▶️ Test senaryosu

| # | Test | Beklenen | Konsolda görülecek SQL |
|---|---|---|---|
| 1 | Liste | Seed'deki 3 fakülte | `SELECT ... WHERE AktifMi = 1` |
| 2 | Boş form gönder | Kırmızı hatalar | *(SQL yok)* |
| 3 | Yeni kayıt | Listeye eklendi | `INSERT INTO Fakulteler` |
| 4 | Düzenle → tek alan değiştir | Güncellendi | `UPDATE ... SET FakulteAd = @p0` — **tek sütun!** |
| 5 | Sil | Listeden gitti | `UPDATE ... SET AktifMi = 0` — **DELETE değil** |
| 6 | SSMS'te bak | Kayıt duruyor, `AktifMi = 0` | |
| 7 | Pasifler sayfası | Silinen kayıt burada | `IgnoreQueryFilters` |
| 8 | Geri getir | Aktif listeye döndü | |
| 9 | Bölümü olan fakülteyi sil | Uyarı, silinmiyor | |

> **4. maddeyi mutlaka yaptır.** Sadece adı değiştirin ve konsoldaki `UPDATE`'e bakın — sadece o sütun var. Bu, change tracking'in en somut kanıtı.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Güncelleme kaydediliyor ama değişmiyor | `SaveChangesAsync()` unutulmuş | Ekle |
| `CreatedDate` 0001-01-01 oluyor | `Update()` ile tüm sütunlar yazılmış | Önce `FindAsync` ile çek |
| Silince kayıt gerçekten gidiyor | `_db.Remove()` kullanılmış | `AktifMi = false` yap |
| Pasif kayıt bulunamıyor | Query filter engelliyor | `IgnoreQueryFilters()` |
| `The instance of entity type cannot be tracked` | Aynı id iki kez takip ediliyor | Aynı `DbContext`'te iki kez çekme, veya `AsNoTracking` |
| Yeni kayıtta id 0 kalıyor | `SaveChanges` öncesi okunmuş | `SaveChanges` sonrası oku |
| `DbUpdateException` (benzersizlik) | Aynı e-posta ikinci kez | Modül 6'da yakalayacağız |
| `ObjectDisposedException` | `DbContext` istek bitince atıldı | `await` unutulmuş olabilir |

---

## ✏️ Öğrenci alıştırması

1. Fakülte CRUD'unu tamamla, dokuz test senaryosunu geçir.
2. Konsoldaki `UPDATE` sorgusunu incele. Sadece bir alanı değiştirdiğinde kaç sütun güncelleniyor?
3. `Edit` metodunu **Yol 2** (`_db.Update(fakulte)`) ile yaz ve dene. `CreatedDate` ne oluyor? Sonra Yol 1'e geri dön.
4. Silme işlemini `_db.Remove(fakulte)` ile değiştir ve dene. Ne oldu? (Sonra geri al!)
5. **Düşünme soruları:**
   - Query filter olmasaydı kaç yere `Where(f => f.AktifMi)` yazmamız gerekirdi?
   - `SaveChanges()` çağrılmadan önce `fakulte.FakulteId` kaçtır? Sonra kaç olur? Breakpoint koyup bak.
   - İki farklı kullanıcı aynı fakülteyi aynı anda düzenlerse ne olur?

---

👉 Sonraki: [`06-bolum-crud.md`](06-bolum-crud.md)
