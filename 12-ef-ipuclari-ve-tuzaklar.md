# Modül 11 — EF İpuçları ve Tuzaklar

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Proje bitti. Şimdi **EF'i güvenle kullanmak için bilinmesi gerekenler.**

Bu modül kod yazma değil, **anlama** modülü. Öğrenci buradaki her başlığı bir kez görürse, ileride saatlerce hata aramaktan kurtulur.

> **Derse şöyle başla:** *"Bugüne kadar EF'in bizim için ne yaptığını gördük. Bugün EF'in bizi nasıl yanıltabileceğini göreceğiz."*

---

## 1️⃣ N+1 problemi — canlı gösterim

EF projelerindeki **en yaygın performans hatası.** Anlatmak yetmez, gösterin.

### Deney

`BolumController.Index` metodunu geçici olarak şöyle yaz:

```csharp
// ❌ KÖTÜ — Include YOK, döngü içinde erişim var
public async Task<IActionResult> Index()
{
    var bolumler = await _db.Bolumler.ToListAsync();

    foreach (var b in bolumler)
    {
        // Her döngüde ayrı sorgu!
        var fakulte = await _db.Fakulteler.FindAsync(b.FakulteId);
        b.Fakulte = fakulte;
    }

    return View(bolumler);
}
```

Konsola bak:

```sql
SELECT ... FROM [Bolumler]                      ← 1 sorgu
SELECT ... FROM [Fakulteler] WHERE FakulteId=1  ← +1
SELECT ... FROM [Fakulteler] WHERE FakulteId=1  ← +1
SELECT ... FROM [Fakulteler] WHERE FakulteId=2  ← +1
SELECT ... FROM [Fakulteler] WHERE FakulteId=3  ← +1
```

**4 bölüm → 5 sorgu.** 500 bölüm olsa **501 sorgu.**

### Doğrusu

```csharp
// ✅ İYİ — tek sorgu
var bolumler = await _db.Bolumler
    .Include(b => b.Fakulte)
    .ToListAsync();
```

Konsolda **tek** `INNER JOIN`'li sorgu.

### Nereden anlaşılır?

| Belirti | Muhtemel sebep |
|---|---|
| Konsolda aynı sorgu defalarca | N+1 |
| Sayfa listede yavaş, detayda hızlı | N+1 |
| Kayıt sayısı arttıkça sayfa katlanarak yavaşlıyor | N+1 |

> **Altın kural:** *"Bir `foreach` döngüsünün içinde `await` görüyorsanız, durup düşünün."*

---

## 2️⃣ AsNoTracking — okuma sorguları için

EF her çektiği kaydın "orijinal fotoğrafını" bellekte tutar (Modül 4, değişiklik takibi). Bu **sadece güncelleme yapacaksanız** gereklidir.

Listeleme sayfalarında bu bellek ve işlem israfıdır.

```csharp
// Sadece göstereceğiz, güncellemeyeceğiz
var ogrenciler = await _db.Ogrenciler
    .AsNoTracking()                  // ⭐ takip etme
    .Include(o => o.Bolum)
    .ToListAsync();
```

| | Takipli (varsayılan) | `AsNoTracking()` |
|---|---|---|
| Bellek | Her kayıt için iki kopya | Tek kopya |
| Hız | Yavaş | ⭐ Belirgin şekilde hızlı |
| `SaveChanges` işe yarar mı? | ✅ Evet | ❌ Hayır |

### Ne zaman kullanılır?

| Senaryo | Kullan? |
|---|---|
| `Index` — liste sayfası | ✅ Evet |
| `Details` — detay sayfası | ✅ Evet |
| `Delete` (GET) — onay ekranı | ✅ Evet |
| `Edit` (GET) — form doldurma | ⚠️ Evet ama POST'ta tekrar çekiyoruz zaten |
| `Edit` (POST) — güncelleme | ❌ **HAYIR** |
| `Delete` (POST) — soft delete | ❌ **HAYIR** |

> **Deney:** `Edit` POST metodundaki `FindAsync` yerine `AsNoTracking().FirstOrDefaultAsync(...)` yaz. Kaydet. **Hiçbir şey değişmez** — EF o nesneyi takip etmediği için değişikliği görmez. Sonra geri al.

### Tüm proje için varsayılan yapmak

```csharp
builder.Services.AddDbContext<UbysDbContext>(secenekler =>
{
    secenekler.UseSqlServer(...);
    secenekler.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
});
```

⚠️ Bunu yaparsanız güncelleme yapacağınız her sorguya `.AsTracking()` eklemeniz gerekir. Ders projesinde **yapmayın** — kafa karıştırır.

---

## 3️⃣ "The LINQ expression could not be translated"

En sık karşılaşılan ve en çok korkutan hata.

### Neden olur?

EF, `Where`/`Select`/`OrderBy` içindeki her şeyi **SQL'e çevirmek** zorunda. Çeviremediği bir şey görürse bu hatayı verir.

### Çevrilemeyen tipik örnekler

```csharp
// ❌ Kendi yazdığınız metot — SQL'de karşılığı yok
.Where(o => YasHesapla(o.OgrenciDogumTarihi) > 20)

// ❌ [NotMapped] özellik
.Where(o => o.TamAd.Contains("Ayşe"))

// ❌ Karmaşık C# ifadesi
.Where(o => o.OgrenciTc.Substring(0, 3).Reverse().ToString() == "123")
```

### Çözüm 1: SQL'e çevrilebilir yaz

```csharp
// ❌
.Where(o => o.TamAd.Contains(arama))

// ✅ Alanları ayrı ayrı kontrol et
.Where(o => o.OgrenciAd.Contains(arama) || o.OgrenciSoyad.Contains(arama))
```

### Çözüm 2: Önce veriyi çek, sonra C# ile filtrele

```csharp
// Önce SQL'e çevrilebilenler
var liste = await _db.Ogrenciler
    .Where(o => o.OgrenciSinif == 3)      // bu çevrilir
    .ToListAsync();                        // ⭐ artık bellekte

// Sonra bellekte, C# ile
var sonuc = liste.Where(o => YasHesapla(o.OgrenciDogumTarihi) > 20).ToList();
```

> ⚠️ **Dikkat:** `ToListAsync()` çağrıldığı anda **tüm eşleşen satırlar belleğe gelir.** 100.000 kayıt varsa hepsi gelir. Bu yöntemi ancak veri kümesi küçükse kullanın.

### Çözüm 3: EF'in bildiği metotları kullan

EF şunları SQL'e çevirebilir:

| C# | SQL |
|---|---|
| `.Contains()`, `.StartsWith()`, `.EndsWith()` | `LIKE` |
| `.ToUpper()`, `.ToLower()` | `UPPER()`, `LOWER()` |
| `.Substring()` | `SUBSTRING()` |
| `.Length` | `LEN()` |
| `.Trim()` | `LTRIM(RTRIM())` |
| `DateTime.Now` | `GETDATE()` |
| `.Year`, `.Month`, `.Day` | `DATEPART()` |
| `+` (metin birleştirme) | `+` |
| `Math.Abs()`, `Math.Round()` | `ABS()`, `ROUND()` |

---

## 4️⃣ Ham SQL çalıştırmak — FromSql

Bazen LINQ yetmez: karmaşık raporlar, saklı yordamlar (stored procedure), veritabanına özgü özellikler.

```csharp
// ⭐ Parametreli — GÜVENLİ
var ogrenciler = await _db.Ogrenciler
    .FromSql($"SELECT * FROM Ogrenciler WHERE OgrenciSinif = {sinif}")
    .ToListAsync();
```

> **`FromSql` içindeki `$"..."` sihirli bir şey.** Normal metin birleştirme gibi görünür ama EF onu **otomatik parametreye** çevirir. SQL injection olmaz.

### ⚠️ Ama `FromSqlRaw` farklı

```csharp
// ❌ TEHLİKELİ — SQL injection açığı!
_db.Ogrenciler.FromSqlRaw("SELECT * FROM Ogrenciler WHERE OgrenciAd = '" + arama + "'")

// ✅ FromSqlRaw kullanacaksanız parametre verin
_db.Ogrenciler.FromSqlRaw("SELECT * FROM Ogrenciler WHERE OgrenciAd = {0}", arama)
```

> **Sınıfa sor:** *"ADO.NET dersinde SQL injection'ı öğrenmiştik. EF bizi otomatik koruyor demiştik. Peki bu satır neden tehlikeli?"*
>
> Çünkü `FromSqlRaw`, EF'in koruma katmanını **atlıyor.** Adındaki "Raw" (ham) bunu haber veriyor. EF sizi korur ama korumasını kendi elinizle kapatabilirsiniz.

### Veri döndürmeyen komutlar

```csharp
// Toplu güncelleme — EF'i hiç kullanmadan
await _db.Database.ExecuteSqlAsync(
    $"UPDATE Ogrenciler SET AktifMi = 0 WHERE BolumId = {bolumId}");
```

⚠️ Bu komut değişiklik takipçisini **atlar.** Bellekteki nesneler eski hâlde kalır.

---

## 5️⃣ Toplu işlemler — ExecuteUpdate / ExecuteDelete

Klasik yöntemin sorunu:

```csharp
// ❌ 1000 öğrenciyi belleğe çeker, 1000 UPDATE üretir
var ogrenciler = await _db.Ogrenciler.Where(o => o.BolumId == 5).ToListAsync();
foreach (var o in ogrenciler)
    o.AktifMi = false;
await _db.SaveChangesAsync();
```

EF Core 7 ve sonrasında **tek sorguyla** yapılabiliyor:

```csharp
// ✅ Tek UPDATE, veri hiç belleğe gelmiyor
await _db.Ogrenciler
    .Where(o => o.BolumId == 5)
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.AktifMi, false)
        .SetProperty(o => o.UpdatedDate, DateTime.Now));
```

Üretilen SQL:
```sql
UPDATE [o] SET [AktifMi] = CAST(0 AS bit), [UpdatedDate] = @p
FROM [Ogrenciler] AS [o] WHERE [o].[BolumId] = @bolumId
```

Silme için:
```csharp
await _db.Ogrenciler.Where(o => !o.AktifMi).ExecuteDeleteAsync();
```

> ⚠️ **Uyarılar:**
> - Değişiklik takipçisini atlar — bellekteki nesneler güncellenmez
> - `SaveChanges` çağrılmaz, komut **hemen** çalışır
> - Bu yüzden transaction dışında kalır

---

## 6️⃣ Eşzamanlılık (concurrency)

**Sınıfa sor:** *"İki kişi aynı öğrenciyi aynı anda düzenlerse ne olur?"*

```
09:00:00  Ali   öğrenciyi açar (sınıf: 2)
09:00:05  Ayşe  aynı öğrenciyi açar (sınıf: 2)
09:00:30  Ali   sınıfı 3 yapar, kaydeder      ✅
09:01:00  Ayşe  telefonu değiştirir, kaydeder ✅
          → Ali'nin değişikliği KAYBOLDU
```

Buna **son yazan kazanır** (last write wins) denir. Sessizce veri kaybı olur.

### Çözüm: satır sürümü

`Ogrenci` modeline ekle:

```csharp
    /// <summary>
    /// Satır sürümü. SQL Server bu sütunu her UPDATE'te
    /// KENDİLİĞİNDEN değiştirir. Biz hiç yazmayız.
    /// </summary>
    [Timestamp]
    public byte[]? SatirSurumu { get; set; }
```

Migration üret ve uygula. Artık EF `UPDATE`'e şunu ekler:

```sql
UPDATE Ogrenciler SET ... WHERE OgrenciId = @id AND SatirSurumu = @eskiSurum
```

Başkası araya girdiyse `SatirSurumu` değişmiştir, hiçbir satır eşleşmez, EF hata fırlatır:

```csharp
try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    ModelState.AddModelError("",
        "Bu kayıt siz düzenlerken başka biri tarafından değiştirildi. " +
        "Sayfayı yenileyip tekrar deneyin.");
    return View(ogrenci);
}
```

> Ders projesinde tek kullanıcı olduğu için gerekmez, ama **kavramı bilmek gerekir.** Gerçek projelerde bu sessiz veri kaybı çok pahalıya mal olur.

---

## 7️⃣ Migration sorunları

### "The model has pending changes"

Model değişti ama migration üretilmedi.
```bash
dotnet ef migrations add DegisiklikAdi
dotnet ef database update
```

### Her `migrations add` boş migration üretiyor

Seed verisinde `DateTime.Now` kullanılmış olabilir. Sabit tarih kullanın (Modül 2).

### Takım arkadaşıyla çakışma

İkiniz de aynı anda migration ürettiyseniz `ModelSnapshot` dosyası çakışır.

**Çözüm:**
1. Kendi migration'ınızı geri alın: `dotnet ef migrations remove`
2. Arkadaşınızın değişikliklerini çekin: `git pull`
3. Migration'ı **yeniden** üretin

> ⚠️ Merge çakışmasını elle çözmeye çalışmayın. `ModelSnapshot` üretilen bir dosyadır, elle düzeltilmez.

### Yanlış migration uygulandı

```bash
# Önceki sürüme dön
dotnet ef database update OncekiMigrationAdi

# Sonra migration dosyasını sil
dotnet ef migrations remove
```

### Her şey karıştı, sıfırdan başlamak istiyorum

```bash
dotnet ef database drop        # veritabanını sil (⚠️ veri gider!)
# Migrations klasörünü sil
dotnet ef migrations add IlkOlusturma
dotnet ef database update
```

> ⚠️ Bu yöntem **sadece geliştirme ortamında** kullanılır. Canlı veritabanında asla.

---

## 8️⃣ EF'te yapılmaması gerekenler — özet kart

Öğrenciye dağıt.

| ❌ Yapma | ✅ Yap | Neden |
|---|---|---|
| `foreach` içinde `await` sorgu | `Include` | N+1 |
| Sadece saymak için `ToList().Count` | `CountAsync()` | Gereksiz veri |
| Listelemede takip | `AsNoTracking()` | Bellek ve hız |
| `FromSqlRaw` + metin birleştirme | `FromSql` veya parametre | SQL injection |
| `Update()` ile tüm nesneyi yazma | Önce çek, alanları değiştir | Alan kaybı |
| Tüm sütunları çekip birini kullanma | `Select` ile projeksiyon | Gereksiz veri |
| Uygulanmış migration'ı elle değiştirme | Yeni migration üret | Şema tutarsızlığı |
| Lazy loading açmak | `Include` | Gizli N+1 |
| Canlıda `EnableSensitiveDataLogging` | Sadece geliştirmede | Veri sızıntısı |
| Seed'de `DateTime.Now` | Sabit tarih | Sürekli boş migration |

---

## 9️⃣ Performans kontrol listesi

Bir sayfa yavaşsa sırayla bak:

```
1. Konsoldaki SQL'e bak
   └─ Kaç sorgu çalışıyor?
      ├─ Çok fazla   → N+1 var, Include ekle
      └─ Tek ama yavaş → devam et

2. Sorgu kaç sütun çekiyor?
   └─ Hepsi lazım mı? → Select ile projeksiyon yap

3. Kaç satır dönüyor?
   └─ Hepsi lazım mı? → Skip/Take ile sayfala

4. WHERE'deki sütunlar indeksli mi?
   └─ Değilse → HasIndex ekle, migration üret

5. Sadece okuyor muyuz?
   └─ Evet → AsNoTracking() ekle

6. Hâlâ yavaş mı?
   └─ SSMS'te execution plan'a bak
```

---

## 🔟 EF'i kapatıp SQL'e dönmek

Bazen EF doğru araç değildir:

| Durum | Neden EF zorlanır |
|---|---|
| Karmaşık raporlama (çoklu pivot, window function) | LINQ karşılığı yok veya çok karmaşık |
| Milyonlarca satır toplu içe aktarma | Değişiklik takibi çöker |
| Veritabanına özgü özellikler | Sağlayıcı desteklemeyebilir |
| Saklı yordam çağırma | `FromSql` gerekir |

**Çözümler:**
- `FromSql` ile ham SQL
- `ExecuteSqlAsync` ile komut
- Dapper (hafif ORM) ile birlikte kullanmak
- ADO.NET'e dönmek

> **Kritik cümle — dersi bununla bitir:**
> *"EF öğrenmek, SQL'i unutmak demek değil. Bugün öğrendiğiniz her şey — Include'un JOIN olduğunu, query filter'ın WHERE eklediğini, Skip/Take'in OFFSET/FETCH olduğunu — ancak SQL bildiğiniz için anlayabildiniz. ADO.NET dersini boşuna yapmadık."*

---

## 🎓 Kurs sonu değerlendirmesi

### Öğrenci artık şunları yapabiliyor

- ✅ EF Core kurup `DbContext` yazabiliyor
- ✅ Entity sınıflarından migration ile veritabanı üretebiliyor
- ✅ LINQ ile sorgu yazabiliyor ve **ürettiği SQL'i okuyabiliyor**
- ✅ İlişkileri `Include`/`ThenInclude` ile yükleyebiliyor
- ✅ Değişiklik takibinin nasıl çalıştığını biliyor
- ✅ N+1 problemini tanıyor ve çözebiliyor
- ✅ Query filter ile soft delete kurabiliyor
- ✅ `DbUpdateException` yakalayabiliyor
- ✅ EF'in ne zaman yetmediğini biliyor

### Son karşılaştırma — tahtaya yaz

| | ADO.NET | EF Core |
|---|---|---|
| Ders saati | 14-16 | 10-12 |
| Veri erişim dosyası | 4 repository | 1 DbContext |
| Elle yazılan SQL | ~25 sorgu | 0 |
| Veritabanı oluşturma | Elle `CREATE TABLE` | Migration |
| En büyük risk | SQL injection | N+1, gizli maliyetler |
| En büyük avantajı | Tam kontrol | Hız ve az kod |

### Bundan sonra

```
BURADASINIZ
    │
    ├──▶ ASP.NET Core Identity      (hazır kullanıcı/rol sistemi)
    ├──▶ Web API + JavaScript       (mobil uygulama da bağlanabilir)
    ├──▶ Repository / Unit of Work  (katmanlı mimari)
    ├──▶ AutoMapper                 (model ↔ DTO dönüşümü)
    ├──▶ Birim testleri             (InMemory provider ile)
    └──▶ Git, CI/CD, yayınlama
```

---

## ✏️ Öğrenci alıştırması

1. **N+1 deneyini yap.** Kötü kodu yaz, konsoldaki sorgu sayısını say, `Include` ekle, tekrar say. İkisinin ekran görüntüsünü al.

2. **`AsNoTracking` ölçümü.** `Index` metoduna `AsNoTracking()` ekle. Konsolda süre farkını gör. (5 kayıtta fark görünmeyebilir — bu da bir ders: *"Optimizasyon ölçmeden yapılmaz."*)

3. **Çeviri hatasını yaşa.** `Where(o => o.TamAd.Contains("a"))` yaz, hatayı al, oku, çöz.

4. **`FromSql` dene.** Öğrenci listesini ham SQL ile çek. Sonra `FromSqlRaw` + metin birleştirme ile yaz ve `' OR '1'='1` girmeyi dene. Ne oldu?

5. **Eşzamanlılık kur.** `Ogrenci`'ye `[Timestamp]` ekle, migration üret. İki tarayıcı sekmesinde aynı öğrenciyi aç, ikisinde de kaydet. Hangisi hata aldı?

6. **Kendi kontrol listeni yaz.** Bu modüldeki tuzaklardan hangisi sana en çok tekrarlayacağın hata gibi geldi? Üç madde yaz, masanın üstüne yapıştır.

---

## 📚 Kaynaklar

- Resmî belgeler: `learn.microsoft.com/ef/core`
- Performans rehberi: `learn.microsoft.com/ef/core/performance`
- LINQ örnekleri: `learn.microsoft.com/dotnet/csharp/linq`

> Belgeleri okumayı öğrenmek, bu kursta öğrendiğiniz her şeyden daha uzun ömürlü bir beceridir. EF sürüm değiştirir, belgeler kalır.

---

🏁 **Kurs tamamlandı.**
