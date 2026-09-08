# Modül 0 — Kurulum ve DbContext

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Projeyi oluşturup EF Core'u kuracağız ve **DbContext** kavramını öğreneceğiz.

---

## ⌨️ Adım 1: Proje

```bash
dotnet new mvc -n UBYS
cd UBYS
```

---

## ⌨️ Adım 2: EF Core paketlerini kur

Üç paket gerekiyor:

```bash
# 1) SQL Server sağlayıcısı — EF'in SQL Server ile konuşmasını sağlar
dotnet add package Microsoft.EntityFrameworkCore.SqlServer

# 2) Tasarım zamanı desteği — migration üretmek için gerekli
dotnet add package Microsoft.EntityFrameworkCore.Design

# 3) Komut satırı aracı (global, bir kez kurulur)
dotnet tool install --global dotnet-ef
```

**Visual Studio kullanıyorsanız:** Projeye sağ tık → NuGet Paketlerini Yönet → ilk iki paketi ara ve kur. Üçüncüsü için Package Manager Console'da:
```powershell
Install-Package Microsoft.EntityFrameworkCore.Tools
```

### Kurulumu doğrula

```bash
dotnet ef --version
```

Şuna benzer bir çıktı görmelisin:
```
Entity Framework Core .NET Command-line Tools
10.0.0
```

> ⚠️ `dotnet ef` komutu bulunamıyorsa terminali kapatıp aç. PATH güncellemesi yeni terminal ister.

---

## 📖 Kavram: DbContext nedir?

Bu, EF'in en merkezi kavramı. Dikkatli anlat.

### ADO.NET'te ne yapıyorduk?

```csharp
// Her metotta bunu tekrar ediyorduk:
using (SqlConnection baglanti = new SqlConnection(_baglantiMetni))
using (SqlCommand komut = new SqlCommand(sql, baglanti))
{
    baglanti.Open();
    // ...
}   // bağlantı kapanır
```

### EF'te?

```csharp
var fakulteler = _db.Fakulteler.ToList();
```

Bağlantı açma, komut hazırlama, okuma, kapatma — **hepsi `DbContext`'in içinde.**

### DbContext'i nasıl anlatmalı?

Tahtaya şu benzetmeyi yaz:

```
DbContext = veritabanıyla yaptığın BİR OTURUM

    ┌─────────────────────────────────────┐
    │          UbysDbContext              │
    │                                     │
    │  DbSet<Fakulte>     Fakulteler      │ ← tablolar
    │  DbSet<Bolum>       Bolumler        │
    │  DbSet<Ogrenci>     Ogrenciler      │
    │  DbSet<Akademisyen> Akademisyenler  │
    │                                     │
    │  ┌───────────────────────────────┐  │
    │  │  DEĞİŞİKLİK TAKİPÇİSİ         │  │ ← Modül 4'te
    │  │  "hangi nesne değişti?"       │  │
    │  └───────────────────────────────┘  │
    │                                     │
    │  SaveChanges()  → hepsini yaz       │
    └─────────────────────────────────────┘
```

| Terim | Karşılığı |
|---|---|
| `DbContext` | Veritabanının kendisi (bir oturum) |
| `DbSet<Fakulte>` | `fakulte` tablosu |
| `_db.Fakulteler` | O tablodaki tüm satırlar |
| `SaveChanges()` | Yapılan değişiklikleri veritabanına yaz |

> **Önemli:** `DbContext` **kısa ömürlüdür**. Her HTTP isteği için yeni bir tane oluşturulur, istek bitince atılır. Uygulama boyunca tek bir tane tutulmaz. Sebebini Modül 4'te (değişiklik takibi) göreceğiz.

---

## ⌨️ Adım 3: Bağlantı dizesi

`appsettings.json`:

```json
{
  "ConnectionStrings": {
    "UbysDb": "Server=localhost\\SQLEXPRESS;Database=UbysDB;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

> **ADO.NET'tekiyle birebir aynı.** Bağlantı dizesi EF'e özel bir şey değil, SQL Server'a nasıl bağlanılacağını söyleyen standart metin.
>
> ⚠️ Veritabanı adı `UbysDB` — **henüz yok.** Modül 2'de EF onu bizim yerimize oluşturacak.

---

## ⌨️ Adım 4: DbContext sınıfını yaz

`Data` klasörü oluştur, içine **`UbysDbContext.cs`**:

```csharp
using Microsoft.EntityFrameworkCore;
using UBYS.Models;

namespace UBYS.Data;

/// <summary>
/// Uygulamanın veritabanı oturumu.
///
/// ADO.NET sürümündeki DÖRT repository sınıfının yerini
/// bu TEK sınıf aldı.
/// </summary>
public class UbysDbContext : DbContext
{
    // ══════════════════════════════════════════════════════
    //  YAPICI METOT
    //
    //  Bağlantı ayarlarını dışarıdan (Program.cs'ten) alıyoruz.
    //  Bu sayede sınıf, bağlantı dizesini kendisi bilmek zorunda kalmıyor.
    //  ADO.NET'te her repository IConfiguration alıyordu — aynı fikir.
    // ══════════════════════════════════════════════════════
    public UbysDbContext(DbContextOptions<UbysDbContext> options)
        : base(options)
    {
    }

    // ══════════════════════════════════════════════════════
    //  TABLOLAR
    //
    //  Her DbSet bir tabloya karşılık gelir.
    //  Özellik adı (Fakulteler) → tablo adı olur.
    //
    //  ⚠️ Model sınıflarını henüz yazmadık, bu satırlar şu an
    //     hata verecek. Modül 1'de yazacağız.
    // ══════════════════════════════════════════════════════
    public DbSet<Fakulte> Fakulteler { get; set; }
    public DbSet<Bolum> Bolumler { get; set; }
    public DbSet<Ogrenci> Ogrenciler { get; set; }
    public DbSet<Akademisyen> Akademisyenler { get; set; }
    public DbSet<Kullanici> Kullanicilar { get; set; }
}
```

### Satır satır açıklama

**`: DbContext`**
EF'in hazır sınıfından türetiyoruz. Bağlantı yönetimi, sorgu çevirisi, değişiklik takibi — hepsi oradan geliyor.

**`DbContextOptions<UbysDbContext> options`**
"Hangi veritabanına, nasıl bağlanacağım?" bilgisini taşıyan nesne. `Program.cs`'te dolduracağız.

**`public DbSet<Fakulte> Fakulteler { get; set; }`**
"`Fakulte` tipinde nesnelerin durduğu bir tablo var, adı `Fakulteler`."

Bundan sonra `_db.Fakulteler` yazdığınızda EF `Fakulteler` tablosuna gider.

---

## ⌨️ Adım 5: Program.cs'e tanıt

```csharp
using Microsoft.EntityFrameworkCore;
using UBYS.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

// ══════════════════════════════════════════════════════════
//  ⭐ EF CORE KAYDI
//
//  ADO.NET'te dört satır yazıyorduk:
//      AddScoped<FakulteRepository>();
//      AddScoped<BolumRepository>();
//      AddScoped<OgrenciRepository>();
//      AddScoped<AkademisyenRepository>();
//
//  Şimdi tek satır:
// ══════════════════════════════════════════════════════════
builder.Services.AddDbContext<UbysDbContext>(secenekler =>
    secenekler.UseSqlServer(
        builder.Configuration.GetConnectionString("UbysDb")));

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

### `AddDbContext` ne yapıyor?

| Parça | Anlamı |
|---|---|
| `AddDbContext<UbysDbContext>` | "Biri `UbysDbContext` isterse ona ver" |
| `UseSqlServer(...)` | "SQL Server kullanacağız" |
| `GetConnectionString("UbysDb")` | "Bağlantı bilgisi appsettings.json'da, bu anahtarda" |

> **`AddDbContext` varsayılan olarak `Scoped`'dır.** Yani her HTTP isteği için yeni bir `DbContext` üretilir, istek bitince atılır. ADO.NET'te repository'leri `AddScoped` ile kaydediyorduk — aynı ömür.

**Neden Singleton değil?** `DbContext` içinde değişiklik takipçisi var; hangi nesnenin değiştiğini hatırlıyor. Uygulama boyunca tek bir tane olsaydı, farklı kullanıcıların değişiklikleri birbirine karışırdı.

---

## 📖 Sağlayıcı (provider) kavramı

`UseSqlServer` yerine başka bir şey de yazabilirdik:

| Paket | Metot | Veritabanı |
|---|---|---|
| `...EntityFrameworkCore.SqlServer` | `UseSqlServer()` | SQL Server |
| `...EntityFrameworkCore.Sqlite` | `UseSqlite()` | SQLite |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | `UseNpgsql()` | PostgreSQL |
| `Pomelo...MySql` | `UseMySql()` | MySQL |

> **Sınıfa sor:** *"ADO.NET sürümünde veritabanını PostgreSQL'e çevirmek isteseydik ne yapmamız gerekirdi?"*
>
> Cevap: Her repository'deki her SQL sorgusunu gözden geçirmek. `OFFSET/FETCH` gibi SQL Server'a özel sözdizimleri çalışmazdı.
>
> EF'te: tek satır değişir. LINQ kodunuza dokunulmaz.
>
> ⚠️ Ama abartma: gerçek projelerde veritabanı değiştirmek yine de zahmetlidir. EF bunu **kolaylaştırır**, bedava yapmaz.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `dotnet ef` komutu bulunamıyor | Araç kurulmamış veya PATH güncel değil | `dotnet tool install --global dotnet-ef`, terminali yeniden aç |
| `The type or namespace 'DbContext' could not be found` | Paket kurulmamış | `Microsoft.EntityFrameworkCore.SqlServer` kur |
| `Unable to resolve service for type 'UbysDbContext'` | `Program.cs`'e `AddDbContext` eklenmemiş | Ekle |
| `Value cannot be null. (Parameter 'connectionString')` | `appsettings.json`'daki anahtar adı uyuşmuyor | `"UbysDb"` birebir aynı olmalı |
| `Your startup project doesn't reference Microsoft.EntityFrameworkCore.Design` | Design paketi eksik | `dotnet add package Microsoft.EntityFrameworkCore.Design` |
| Model sınıfları bulunamıyor hatası | Henüz yazmadık | Modül 1'de yazacağız, şimdilik normal |

---

## ✏️ Öğrenci alıştırması

1. Projeyi kur, paketleri yükle, `dotnet ef --version` çıktısını al.
2. `UbysDbContext` sınıfını yaz. (Model sınıfları olmadığı için derlenmeyecek — normal.)
3. **Düşünme sorusu:** ADO.NET sürümünde dört repository vardı, burada tek `DbContext` var. Bu iyi mi kötü mü? Bir sınıfın çok fazla iş yapması sorun olur mu?

---

👉 Sonraki: [`02-model-siniflari.md`](02-model-siniflari.md)
