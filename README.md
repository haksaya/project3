# UBYS — Entity Framework Core ile

**Üniversite Bilgi Yönetim Sistemi**
ASP.NET Core MVC + EF Core + Bootstrap 5

Aynı projeyi daha önce **ADO.NET** ile yaptık. Bu rehber onu **Entity Framework Core** ile yeniden yazar.

---

## Bu rehberin farkı

Konu değişmedi — fakülte, bölüm, öğrenci, akademisyen. **Değişen sadece veritabanıyla konuşma şeklimiz.**

Her modülde şu soru sorulur: *"Bunu ADO.NET'te nasıl yazıyorduk?"*

```
   C# DÜNYASI                          VERİTABANI DÜNYASI
   ──────────                          ──────────────────
   Fakulte sınıfı        ◀── EF ──▶    Fakulteler tablosu
   Fakulte nesnesi       ◀── EF ──▶    bir satır
   fakulte.FakulteAd     ◀── EF ──▶    FakulteAd sütunu
   List<Fakulte>         ◀── EF ──▶    SELECT sonucu
```

---

## ADO.NET ile EF Core — yan yana

| İş | ADO.NET | EF Core |
|---|---|---|
| Listeleme | `SELECT` + `while(Read())` + elle nesneye çevir | `_db.Fakulteler.ToList()` |
| Ekleme | `INSERT` + parametreler | `_db.Add(f); _db.SaveChanges();` |
| Güncelleme | `UPDATE ... SET ... WHERE` | Nesneyi değiştir + `SaveChanges()` |
| JOIN | SQL'de `INNER JOIN` yaz | `.Include(b => b.Fakulte)` |
| Sayfalama | `OFFSET ... FETCH NEXT` | `.Skip(n).Take(m)` |
| Tablo oluşturma | Elle `CREATE TABLE` | **Migration** |
| SQL injection | Parametre kullanmak zorunlu | Otomatik korunur |

---

## Modüller

Dosyalar sırayla okunacak şekilde birbirine bağlıdır.

| # | Modül | Süre | Ana kavram |
|---|---|---|---|
| — | [Kurs planı](00-KURS-PLANI.md) | — | Karşılaştırma, takvim, mimari |
| 0 | [Kurulum ve DbContext](01-kurulum-ve-dbcontext.md) | 1 saat | `DbContext`, `DbSet` |
| 1 | [Model sınıfları ve ilişkiler](02-model-siniflari.md) | 1 saat | Entity, navigasyon özellikleri |
| 2 | [Migration ve veritabanı](03-migration-ve-veritabani.md) | 1 saat | Code First, seed verisi |
| 3 | [LINQ ve ilk liste](04-linq-ve-ilk-liste.md) | 1 saat | LINQ, ertelenmiş çalıştırma |
| 4 | [Fakülte CRUD](05-fakulte-crud.md) | 2 saat | Değişiklik takibi, query filter |
| 5 | [Bölüm CRUD](06-bolum-crud.md) | 1-2 saat | `Include`, eager loading |
| 6 | [Öğrenci CRUD](07-ogrenci-crud.md) | 1-2 saat | `DbUpdateException`, `ThenInclude` |
| 7 | [Akademisyen CRUD](08-akademisyen-crud.md) | 1 saat | Kalıbın pekişmesi |
| 8 | [Dashboard](09-dashboard.md) | 1 saat | `GroupBy`, projeksiyon |
| 9 | [Arama, filtreleme, sayfalama](10-arama-filtreleme-sayfalama.md) | 1 saat | `IQueryable`, `Skip`/`Take` |
| 10 | [Giriş sistemi](11-login.md) | 1 saat | Çerez kimlik doğrulama |
| 11 | [EF ipuçları ve tuzaklar](12-ef-ipuclari-ve-tuzaklar.md) | 1 saat | N+1, `AsNoTracking`, performans |

Toplam **10-12 ders saati**. ADO.NET sürümü 14-16 saatti.

---

## Modül şablonu

| Bölüm | İşlevi |
|---|---|
| 🎯 Bu derste ne yapacağız | Derse başlarken tahtaya yazılacak hedef |
| 📖 Kavram | Kod yazmadan önceki teori |
| ⌨️ Adım adım kod | Satır satır açıklamalı, **eksiksiz** kod |
| ▶️ Çalıştır ve gör | Ekranda ve **konsolda** görünmesi gereken |
| ⚠️ Sık yapılan hatalar | Derste karşılaşılacak sorunlar ve çözümleri |
| ✏️ Öğrenci alıştırması | Ders sonu ödevi |

Öğrenciye bırakılmış eksik kod **yoktur**. Tüm controller'lar, model'ler ve view'lar tam yazılmıştır.

---

## Rehberin pedagojik omurgası

İki şey her modülde tekrarlanır:

**1. Karşılaştırma.** Her yeni EF özelliği, ADO.NET'teki karşılığıyla yan yana gösterilir. Öğrenci neyin değiştiğini görür.

**2. Üretilen SQL'i okumak.** `Program.cs`'e eklenen tek satırla EF'in ürettiği SQL konsola yazılır:

```csharp
secenekler.LogTo(Console.WriteLine, LogLevel.Information);
```

Her modülde o SQL beraber incelenir. Amaç EF'in sihir olmadığını göstermek.

> *"EF, SQL'i sizin yerinize yazar. Ama SQL bilmiyorsanız, EF'in yazdığı kötü SQL'i fark edemezsiniz. Bu yüzden önce ADO.NET yaptık."*

---

## Başlarken

**Gerekenler**

- .NET SDK 10.0 *(veya 8.0 — tüm kodlar ikisinde de çalışır)*
- SQL Server Express + SSMS
- Visual Studio 2022 veya VS Code + C# Dev Kit

**Kurulum**

```bash
dotnet new mvc -n UBYS
cd UBYS

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet tool install --global dotnet-ef

dotnet ef --version
```

Ardından [Modül 0](01-kurulum-ve-dbcontext.md) ile başlayın.

Veritabanını **elle oluşturmayın** — Modül 2'de migration ile üretilecek.

---

## Ön koşul

Bu rehber, öğrencinin **ADO.NET sürümünü tamamladığını** varsayar. SQL'i, MVC'yi ve Bootstrap'i bildiğinizi kabul eder ve yalnızca değişen kısımlara odaklanır.

İlk kez veri erişimi öğrenen bir grup için önce ADO.NET sürümü işlenmelidir. Aksi hâlde `Include`'un JOIN, query filter'ın `WHERE` olduğu anlaşılmaz — EF sihir gibi görünür ve öğrenci hata ayıklayamaz.

---

## İşlenen tasarım düzeltmeleri

Code First'e geçmek, ADO.NET sürümünde eleştirdiğimiz tasarım hatalarını düzeltme fırsatı verdi:

| Konu | ADO.NET sürümü | EF sürümü |
|---|---|---|
| Aktiflik alanı | `NVARCHAR(255)` → `'1'` | `bool` → `BIT` |
| Soft delete filtresi | Her sorguda elle `WHERE` | Global query filter (tek satır) |
| Benzersizlik indeksleri | Elle `CREATE UNIQUE INDEX` | Fluent API + migration |
| Silme davranışı | Tanımsız | `DeleteBehavior.Restrict` |
| Şema değişikliği | Elle `ALTER TABLE` | Migration dosyası |

---

## Lisans

Eğitim amaçlı serbestçe kullanılabilir, çoğaltılabilir ve uyarlanabilir.

> ⚠️ Rehberdeki şifre hash'leme örneği (SHA256) **öğretim amaçlıdır**, gerçek projeler için yeterli değildir. Gerekçesi ve doğrusu Modül 10'da açıklanmıştır.
