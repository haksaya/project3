# Modül 3 — LINQ ve İlk Liste

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Veritabanından veri okuyup ekrana basacağız. **Yeni kavram: LINQ.**

Ayrıca EF'in ürettiği SQL'i **gözümüzle göreceğiz** — bu modülün en önemli kısmı.

---

## 📖 Kavram: LINQ nedir?

**LINQ** (Language Integrated Query) = C# içine gömülü sorgu dili.

Yan yana koy:

```csharp
// ── ADO.NET ─────────────────────────────────────────
string sql = @"SELECT fakulte_id, fakulte_ad, ...
               FROM fakulte
               WHERE is_active = '1'
               ORDER BY fakulte_ad";

using (SqlConnection baglanti = new SqlConnection(_baglantiMetni))
using (SqlCommand komut = new SqlCommand(sql, baglanti))
{
    baglanti.Open();
    using (SqlDataReader okuyucu = komut.ExecuteReader())
    {
        while (okuyucu.Read())
        {
            Fakulte f = new Fakulte();
            f.FakulteId = okuyucu.GetInt64(okuyucu.GetOrdinal("fakulte_id"));
            f.FakulteAd = okuyucu.GetString(okuyucu.GetOrdinal("fakulte_ad"));
            // ... 6 satır daha
            liste.Add(f);
        }
    }
}
```

```csharp
// ── EF Core ─────────────────────────────────────────
var liste = _db.Fakulteler
               .Where(f => f.AktifMi)
               .OrderBy(f => f.FakulteAd)
               .ToList();
```

**25 satır → 4 satır.** Ve ikisi de aynı SQL'i çalıştırıyor.

---

## 📖 LINQ ↔ SQL sözlüğü

Bu tabloyu öğrenciye dağıt. Rehberin en çok bakılacak sayfası.

| SQL | LINQ | Örnek |
|---|---|---|
| `SELECT *` | `.ToList()` | `_db.Fakulteler.ToList()` |
| `WHERE` | `.Where(...)` | `.Where(f => f.AktifMi)` |
| `ORDER BY` | `.OrderBy(...)` | `.OrderBy(f => f.FakulteAd)` |
| `ORDER BY ... DESC` | `.OrderByDescending(...)` | `.OrderByDescending(f => f.CreatedDate)` |
| İkinci sıralama | `.ThenBy(...)` | `.OrderBy(a).ThenBy(b)` |
| `WHERE id = 5` (tek satır) | `.Find(5)` / `.FirstOrDefault(...)` | `_db.Fakulteler.Find(5)` |
| `COUNT(*)` | `.Count()` | `_db.Ogrenciler.Count()` |
| `SUM(x)` | `.Sum(o => o.X)` | |
| `AVG(x)` | `.Average(o => o.X)` | |
| `MAX(x)` | `.Max(o => o.X)` | |
| `TOP 5` | `.Take(5)` | |
| `OFFSET n` | `.Skip(n)` | |
| `INNER JOIN` | `.Include(...)` | `.Include(b => b.Fakulte)` |
| `GROUP BY` | `.GroupBy(...)` | |
| `LIKE '%x%'` | `.Contains("x")` | `.Where(f => f.FakulteAd.Contains("Müh"))` |
| `LIKE 'x%'` | `.StartsWith("x")` | |
| `SELECT sütun1, sütun2` | `.Select(x => new {...})` | |
| `EXISTS` | `.Any(...)` | `.Any(f => f.AktifMi)` |
| `DISTINCT` | `.Distinct()` | |

### `f => f.AktifMi` bu ne?

**Lambda ifadesi.** Okunuşu: *"f alıyorum, f'in AktifMi değerini dönüyorum."*

```csharp
.Where(f => f.AktifMi)
       │    └─ koşul
       └─ her satıra verilen geçici ad (istediğin ismi verebilirsin)
```

Uzun hâli şuna denk:
```csharp
.Where(delegate(Fakulte f) { return f.AktifMi; })
```

> Öğrenci lambda'yı ilk kez görüyorsa birkaç örnek yaptır. `x =>` ifadesindeki `x` sadece bir isim — `f`, `fakulte`, `item` da olabilir.

---

## 📖 Kavram: Ertelenmiş çalıştırma (deferred execution)

⭐ **Bu, EF'in en çok yanlış anlaşılan davranışı.** Mutlaka anlat.

```csharp
var sorgu = _db.Fakulteler.Where(f => f.AktifMi);   // ← veritabanına GİTMEZ
```

Bu satır çalıştığında **hiçbir SQL çalışmaz.** Sadece "ne isteyeceğimizin tarifi" hazırlanır.

```csharp
var liste = sorgu.ToList();    // ← ŞİMDİ veritabanına gider
```

### Sorguyu hangi metotlar çalıştırır?

| Erteler (IQueryable döner) | Çalıştırır (sonuç döner) |
|---|---|
| `.Where()` | `.ToList()` |
| `.OrderBy()` | `.ToArray()` |
| `.Include()` | `.First()` / `.FirstOrDefault()` |
| `.Select()` | `.Single()` / `.SingleOrDefault()` |
| `.Skip()` / `.Take()` | `.Count()` |
| | `.Any()` |
| | `.Sum()` / `.Max()` / `.Average()` |
| | `foreach` döngüsü |

### Neden önemli?

Çünkü sorguyu **parça parça kurabiliriz:**

```csharp
IQueryable<Fakulte> sorgu = _db.Fakulteler;          // henüz SQL yok

if (aramaVar)
    sorgu = sorgu.Where(f => f.FakulteAd.Contains(arama));

if (sadeceAktif)
    sorgu = sorgu.Where(f => f.AktifMi);

sorgu = sorgu.OrderBy(f => f.FakulteAd);

var sonuc = sorgu.ToList();     // ← TEK sorgu çalışır, tüm koşullar birleşmiş
```

> **ADO.NET'te ne yapıyorduk?** Metin olarak SQL kuruyorduk:
> ```csharp
> if (aramaVar) kosullar += " AND fakulte_ad LIKE @arama ";
> ```
> Aynı fikir, ama artık metin değil **kod** birleştiriyoruz. Yazım hatası derleme zamanında yakalanıyor.

Modül 9'da (filtreleme) bunu bolca kullanacağız.

---

## ⌨️ Adım 1: Controller

`Controllers/FakulteController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

public class FakulteController : Controller
{
    // ⭐ ADO.NET'te FakulteRepository enjekte ediyorduk.
    //    Şimdi doğrudan DbContext.
    private readonly UbysDbContext _db;

    public FakulteController(UbysDbContext db)
    {
        _db = db;
    }

    // GET: /Fakulte
    public async Task<IActionResult> Index()
    {
        var fakulteler = await _db.Fakulteler
            .Where(f => f.AktifMi)              // WHERE AktifMi = 1
            .OrderBy(f => f.FakulteAd)          // ORDER BY FakulteAd
            .ToListAsync();                     // çalıştır

        return View(fakulteler);
    }
}
```

### 📖 `async` / `await` neden?

ADO.NET sürümünde senkron yazmıştık. EF'te **asenkron** yazmak standarttır.

```csharp
public async Task<IActionResult> Index()          // async ekle, Task<> ile sar
{
    var liste = await _db.Fakulteler.ToListAsync();   // await + ...Async()
    return View(liste);
}
```

| Senkron | Asenkron |
|---|---|
| `IActionResult` | `async Task<IActionResult>` |
| `.ToList()` | `await ...ToListAsync()` |
| `.FirstOrDefault()` | `await ...FirstOrDefaultAsync()` |
| `.Count()` | `await ...CountAsync()` |
| `.Any()` | `await ...AnyAsync()` |
| `_db.SaveChanges()` | `await _db.SaveChangesAsync()` |

**Ne işe yarıyor?** Veritabanı cevap verirken sunucunun o iş parçacığı boşta bekler. `await` sayesinde o sırada başka kullanıcılara hizmet verebilir.

> Öğrenciye: *"Şimdilik kuralı ezberleyin: metoda `async Task<>` yazın, EF metotlarının `Async` bitenlerini `await` ile çağırın. Ayrıntısını ileri seviye derslerde göreceksiniz."*
>
> ⚠️ `await` yazmayı unutursanız kod derlenir ama beklenmedik davranır. Visual Studio bunu yeşil dalgalı çizgiyle uyarır — uyarıyı ciddiye alın.

`ToListAsync`, `FirstOrDefaultAsync` gibi metotlar için `using Microsoft.EntityFrameworkCore;` satırı gerekir.

---

## ⌨️ Adım 2: View

`Views/Fakulte/Index.cshtml`:

```html
@model List<UBYS.Models.Fakulte>
@{
    ViewData["Title"] = "Fakülteler";
}

<div class="card border-0 shadow-sm">
    <div class="card-header bg-white d-flex justify-content-between align-items-center">
        <span class="fw-semibold">Kayıtlı fakülteler (@Model.Count)</span>
    </div>

    <div class="card-body p-0">
        <table class="table table-hover align-middle mb-0">
            <thead class="table-light">
                <tr>
                    <th>#</th>
                    <th>Fakülte adı</th>
                    <th>Telefon</th>
                    <th>E-posta</th>
                    <th>Kayıt tarihi</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var fakulte in Model)
                {
                    <tr>
                        <td>@fakulte.FakulteId</td>
                        <td class="fw-semibold">@fakulte.FakulteAd</td>
                        <td>@fakulte.FakulteTelefon</td>
                        <td>@fakulte.FakulteEposta</td>
                        <td>@fakulte.CreatedDate.ToString("dd.MM.yyyy")</td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>
```

> **View tarafında hiçbir şey değişmedi.** ADO.NET sürümündeki view'ın aynısı. Bu, öğrenciye söylenmeye değer: *"Değişen sadece verinin nereden geldiği. Ekran katmanı bundan habersiz."*

---

## ▶️ Adım 3: Üretilen SQL'i gör ⭐

Bu, modülün en öğretici kısmı. **Atlama.**

`Program.cs`'teki `AddDbContext` çağrısını genişlet:

```csharp
builder.Services.AddDbContext<UbysDbContext>(secenekler =>
{
    secenekler.UseSqlServer(
        builder.Configuration.GetConnectionString("UbysDb"));

    // ⭐ Sadece GELİŞTİRME ortamında: üretilen SQL'i konsola yaz
    if (builder.Environment.IsDevelopment())
    {
        secenekler.LogTo(Console.WriteLine, LogLevel.Information);

        // Parametre DEĞERLERİNİ de göster
        // ⚠️ Canlıda ASLA açma — şifreler, TC numaraları loglara düşer!
        secenekler.EnableSensitiveDataLogging();
    }
});
```

`using Microsoft.Extensions.Logging;` eklemeyi unutma.

Uygulamayı çalıştır, `/Fakulte` sayfasına git, **terminale bak:**

```sql
SELECT [f].[FakulteId], [f].[AktifMi], [f].[CreatedDate], [f].[FakulteAd],
       [f].[FakulteAdres], [f].[FakulteEposta], [f].[FakulteTelefon],
       [f].[UpdatedDate]
FROM [Fakulteler] AS [f]
WHERE [f].[AktifMi] = CAST(1 AS bit)
ORDER BY [f].[FakulteAd]
```

### Öğrenciyle beraber inceleyin

Sorular sor:

1. **"Bu SQL'i siz yazsaydınız nasıl yazardınız?"** → Neredeyse aynısı.
2. **"`WHERE` nereden geldi?"** → `.Where(f => f.AktifMi)`
3. **"`ORDER BY` nereden geldi?"** → `.OrderBy(f => f.FakulteAd)`
4. **"`SELECT *` yerine neden sütunlar tek tek yazılmış?"** → EF hangi sütunlara ihtiyacı olduğunu bilir, gereksiz veri çekmez.

> **Kritik cümle:** *"Bundan sonra EF'e güvenmeyin, kontrol edin. Yavaş bir sayfa gördüğünüzde ilk yapacağınız şey buraya bakmak. EF'in ne yazdığını görmeden 'EF yavaş' diyemezsiniz."*

---

## 📖 `Find` ve `FirstOrDefault` farkı

Tek kayıt getirmenin iki yolu var:

```csharp
// 1) Find — SADECE birincil anahtarla
var f = await _db.Fakulteler.FindAsync(5);

// 2) FirstOrDefault — herhangi bir koşulla
var f = await _db.Fakulteler.FirstOrDefaultAsync(x => x.FakulteEposta == "a@b.com");
```

| | `Find` | `FirstOrDefault` |
|---|---|---|
| Ne ile arar | Sadece PK | Herhangi bir koşul |
| Önbellek | ⭐ Önce bellekte arar, yoksa veritabanına gider | Her zaman veritabanına gider |
| `Include` ile | ❌ kullanılamaz | ✅ kullanılabilir |
| Bulamazsa | `null` | `null` |

> **`Find`'ın önbellek davranışı önemli:** Aynı `DbContext` içinde o kaydı daha önce çektiyseniz, ikinci `Find` veritabanına **hiç gitmez.** Bu bazen istediğiniz şeydir, bazen değil.

**Benzer metotlar:**

| Metot | Kayıt yoksa | Birden fazla varsa |
|---|---|---|
| `First()` | ❌ hata | ilkini döner |
| `FirstOrDefault()` | `null` | ilkini döner |
| `Single()` | ❌ hata | ❌ hata |
| `SingleOrDefault()` | `null` | ❌ hata |

> `Single`, "bu koşula tam olarak bir kayıt uymalı" demek istediğinizde kullanılır. Fazlası varsa hata vermesi **iyi bir şeydir** — veri tutarsızlığını erken yakalar.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `'DbSet<Fakulte>' does not contain 'ToListAsync'` | using eksik | `using Microsoft.EntityFrameworkCore;` |
| `Cannot await 'List<Fakulte>'` | `ToList()` yazılmış | `ToListAsync()` |
| Metot `async` ama `await` yok uyarısı | `await` unutulmuş | Ekle |
| `Invalid object name 'Fakulteler'` | Migration uygulanmamış | `dotnet ef database update` |
| Liste boş geliyor | Seed verisi yok veya `AktifMi` false | SSMS'ten kontrol et |
| Konsolda SQL görünmüyor | `LogTo` eklenmemiş veya Production ortamı | Kontrol et |
| `The LINQ expression could not be translated` | Kullandığın C# metodu SQL'e çevrilemiyor | Modül 11'e bak |

---

## ✏️ Öğrenci alıştırması

1. Fakülte listesini çalıştır, konsolda SQL'i gör, ekran görüntüsü al.
2. Şu sorguları yaz ve her birinin ürettiği SQL'i incele:
   - Adı "M" ile başlayan fakülteler
   - En son eklenen 2 fakülte
   - Toplam aktif fakülte sayısı
   - "Mühendislik" kelimesi geçen fakülte var mı? (`Any`)
3. `.ToList()` satırını sil, `var sorgu = _db.Fakulteler.Where(...)` yaz ve breakpoint koy. `sorgu` değişkeninin içinde ne var? Veritabanına gitmiş mi?
4. **Düşünme soruları:**
   - `Where` ile `ToList` yerlerini değiştirsek ne olurdu?
     `_db.Fakulteler.ToList().Where(f => f.AktifMi)` — bu neden kötü?
   - `Find` ile `FirstOrDefault` arasında hangisini ne zaman kullanmalı?

---

👉 Sonraki: [`05-fakulte-crud.md`](05-fakulte-crud.md)
