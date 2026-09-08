# Modül 2 — Migration ve Veritabanı

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Yazdığımız C# sınıflarından **veritabanını üreteceğiz**. Tek satır `CREATE TABLE` yazmadan.

---

## 📖 Kavram: Migration nedir?

**Sınıfa sor:** *"ADO.NET projesinde `bolum` tablosuna yeni bir sütun eklemek isteseydik ne yapardık?"*

Cevap:
1. SSMS'i aç, `ALTER TABLE` yaz
2. Model sınıfına özelliği ekle
3. Repository'deki `SELECT`, `INSERT`, `UPDATE` sorgularını güncelle
4. Arkadaşına "sen de şu ALTER TABLE'ı çalıştır" de
5. Canlı sunucuda da elle çalıştırmayı unutma

**Beş adım, hepsi elle, biri unutulursa sistem bozulur.**

### Migration ile

1. Model sınıfına özelliği ekle
2. `dotnet ef migrations add SutunEklendi`
3. `dotnet ef database update`

**Ve en önemlisi:** Bu değişiklik bir **dosya** olarak projede durur. Arkadaşın projeyi çekince `database update` çalıştırır, aynı değişiklik onun veritabanında da olur.

```
Migration = veritabanı şemasının VERSİYON KONTROLÜ
```

Tahtaya çiz:

```
  Model sınıfları              Migration dosyaları           Veritabanı
  ───────────────              ───────────────────           ──────────
  Fakulte.cs                   001_IlkOlusturma.cs           Fakulteler
  Bolum.cs        ──add──▶     002_UnvanEklendi.cs   ──update──▶  Bolumler
  Ogrenci.cs                   003_IndeksEklendi.cs          Ogrenciler
                                      │
                              her biri "ne değişti"yi
                                 kod olarak tutar
```

---

## ⌨️ Adım 1: İlk migration'ı üret

Terminalde, proje klasöründe:

```bash
dotnet ef migrations add IlkOlusturma
```

**Visual Studio'da** (Package Manager Console):
```powershell
Add-Migration IlkOlusturma
```

Çıktı:
```
Build started...
Build succeeded.
Done. To undo this action, use 'ef migrations remove'
```

Projede yeni bir klasör oluştu: **`Migrations/`**

```
Migrations/
├── 20260901120000_IlkOlusturma.cs              ← değişiklikler
├── 20260901120000_IlkOlusturma.Designer.cs     ← EF'in iç kaydı
└── UbysDbContextModelSnapshot.cs               ← modelin son hâli
```

---

## 📖 Migration dosyasının içi

`Migrations/..._IlkOlusturma.cs` dosyasını aç. **Bu dosyayı öğrenciye mutlaka gösterin** — EF'in sihir olmadığını burada anlarlar.

```csharp
public partial class IlkOlusturma : Migration
{
    /// <summary>
    /// İleri gitmek: değişikliği UYGULA
    /// </summary>
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Fakulteler",
            columns: table => new
            {
                FakulteId = table.Column<long>(type: "bigint", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                FakulteAd = table.Column<string>(type: "nvarchar(255)", maxLength: 255, nullable: false),
                FakulteAdres = table.Column<string>(type: "nvarchar(500)", maxLength: 500, nullable: false),
                FakulteTelefon = table.Column<string>(type: "nvarchar(20)", maxLength: 20, nullable: false),
                FakulteEposta = table.Column<string>(type: "nvarchar(255)", maxLength: 255, nullable: false),
                CreatedDate = table.Column<DateTime>(type: "datetime2", nullable: false),
                UpdatedDate = table.Column<DateTime>(type: "datetime2", nullable: true),
                AktifMi = table.Column<bool>(type: "bit", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Fakulteler", x => x.FakulteId);
            });

        // ... diğer tablolar, indeksler, yabancı anahtarlar
    }

    /// <summary>
    /// Geri almak: değişikliği KALDIR
    /// </summary>
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Ogrenciler");
        migrationBuilder.DropTable(name: "Akademisyenler");
        migrationBuilder.DropTable(name: "Bolumler");
        migrationBuilder.DropTable(name: "Fakulteler");
    }
}
```

### Öğrenciye gösterilecek eşleşmeler

Model sınıfındaki her satırın burada karşılığı var. Tahtada eşleştir:

| Model'de yazdığımız | Migration'da oluşan | SQL'de |
|---|---|---|
| `public long FakulteId` | `.Annotation("SqlServer:Identity", "1, 1")` | `BIGINT IDENTITY(1,1)` |
| `[StringLength(255)]` | `maxLength: 255` | `NVARCHAR(255)` |
| `[Required]` | `nullable: false` | `NOT NULL` |
| `DateTime?` | `nullable: true` | `NULL` |
| `bool AktifMi` | `type: "bit"` | `BIT` |
| `.HasIndex(...).IsUnique()` | `CreateIndex(unique: true)` | `CREATE UNIQUE INDEX` |

> **Kritik cümle:** *"Gördüğünüz gibi EF sihir yapmıyor. Sizin yazdığınız `CREATE TABLE`'ı sizin yerinize yazıyor. Fark şu: siz bir kez yazıp unutuyordunuz, EF her değişikliği kayıt altına alıyor."*

### `Up` ve `Down`

| Metot | Ne zaman çalışır |
|---|---|
| `Up()` | `database update` → değişikliği uygula |
| `Down()` | Geri alınca → değişikliği iptal et |

`Down` metodu sayesinde yanlış giden bir migration geri alınabilir. Bu, elle yazılan `ALTER TABLE`'da olmayan bir güvence.

---

## ⌨️ Adım 2: Veritabanını oluştur

```bash
dotnet ef database update
```

**Visual Studio:**
```powershell
Update-Database
```

Çıktı:
```
Build started...
Build succeeded.
Applying migration '20260901120000_IlkOlusturma'.
Done.
```

### Kontrol et

SSMS'i aç:

```sql
USE UbysDB;
GO

SELECT name FROM sys.tables ORDER BY name;
```

Sonuç:
```
Akademisyenler
Bolumler
Fakulteler
Kullanicilar
Ogrenciler
__EFMigrationsHistory      ← ⭐ bu ne?
```

### 📖 `__EFMigrationsHistory` tablosu

```sql
SELECT * FROM __EFMigrationsHistory;
```

| MigrationId | ProductVersion |
|---|---|
| 20260901120000_IlkOlusturma | 10.0.0 |

**EF, hangi migration'ları uyguladığını burada tutar.** `database update` çalıştırdığınızda önce bu tabloya bakar, uygulanmamış olanları bulur ve sadece onları çalıştırır.

> **Bu yüzden `database update` komutunu istediğiniz kadar çalıştırabilirsiniz** — zaten uygulanmış olanı tekrar uygulamaz.

---

## ⌨️ Adım 3: Başlangıç verisi (seed)

Test verisini de koddan verebiliriz. `OnModelCreating` metodunun sonuna ekle:

```csharp
        // ══════════════════════════════════════════════════════
        //  BAŞLANGIÇ VERİSİ (SEED DATA)
        //
        //  ⚠️ HasData ile verilen kayıtların ID'leri ELLE yazılır.
        //     EF bunları migration dosyasına INSERT olarak koyar.
        //
        //  ⚠️ DateTime.Now KULLANILAMAZ!
        //     Seed verisi migration'a sabit olarak yazılır; her
        //     "migrations add" komutunda tarih değişirse EF
        //     "model değişti" sanar ve boş migration üretir.
        //     Bu yüzden SABİT tarih kullanıyoruz.
        // ══════════════════════════════════════════════════════
        var sabitTarih = new DateTime(2026, 1, 1, 9, 0, 0);

        modelBuilder.Entity<Fakulte>().HasData(
            new Fakulte
            {
                FakulteId = 1,
                FakulteAd = "Mühendislik Fakültesi",
                FakulteAdres = "Merkez Kampüs A Blok",
                FakulteTelefon = "02124440101",
                FakulteEposta = "muhendislik@ornekuni.edu.tr",
                CreatedDate = sabitTarih,
                AktifMi = true
            },
            new Fakulte
            {
                FakulteId = 2,
                FakulteAd = "Fen-Edebiyat Fakültesi",
                FakulteAdres = "Merkez Kampüs B Blok",
                FakulteTelefon = "02124440102",
                FakulteEposta = "fenedebiyat@ornekuni.edu.tr",
                CreatedDate = sabitTarih,
                AktifMi = true
            },
            new Fakulte
            {
                FakulteId = 3,
                FakulteAd = "İktisadi ve İdari Bilimler Fakültesi",
                FakulteAdres = "Güney Kampüs C Blok",
                FakulteTelefon = "02124440103",
                FakulteEposta = "iibf@ornekuni.edu.tr",
                CreatedDate = sabitTarih,
                AktifMi = true
            }
        );

        modelBuilder.Entity<Bolum>().HasData(
            new Bolum
            {
                BolumId = 1, FakulteId = 1,
                BolumAdi = "Bilgisayar Mühendisliği",
                BolumAdres = "A Blok 3. Kat",
                BolumTelefon = "02124440201",
                BolumEposta = "bilgisayar@ornekuni.edu.tr",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Bolum
            {
                BolumId = 2, FakulteId = 1,
                BolumAdi = "Makine Mühendisliği",
                BolumAdres = "A Blok 2. Kat",
                BolumTelefon = "02124440202",
                BolumEposta = "makine@ornekuni.edu.tr",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Bolum
            {
                BolumId = 3, FakulteId = 2,
                BolumAdi = "Matematik",
                BolumAdres = "B Blok 2. Kat",
                BolumTelefon = "02124440203",
                BolumEposta = "matematik@ornekuni.edu.tr",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Bolum
            {
                BolumId = 4, FakulteId = 3,
                BolumAdi = "İşletme",
                BolumAdres = "C Blok 1. Kat",
                BolumTelefon = "02124440204",
                BolumEposta = "isletme@ornekuni.edu.tr",
                CreatedDate = sabitTarih, AktifMi = true
            }
        );

        modelBuilder.Entity<Ogrenci>().HasData(
            new Ogrenci
            {
                OgrenciId = 1, BolumId = 1,
                OgrenciAd = "Ayşe", OgrenciSoyad = "Yılmaz",
                OgrenciSinif = 3,
                OgrenciDogumTarihi = new DateTime(2003, 4, 12),
                OgrenciCinsiyet = "Kadın",
                OgrenciAdres = "Bağcılar / İstanbul",
                OgrenciTelefon = "05321110001",
                OgrenciEposta = "ayse.yilmaz@ogr.ornekuni.edu.tr",
                OgrenciTc = "42950115298",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Ogrenci
            {
                OgrenciId = 2, BolumId = 1,
                OgrenciAd = "Mehmet", OgrenciSoyad = "Demir",
                OgrenciSinif = 2,
                OgrenciDogumTarihi = new DateTime(2004, 9, 25),
                OgrenciCinsiyet = "Erkek",
                OgrenciAdres = "Kadıköy / İstanbul",
                OgrenciTelefon = "05321110002",
                OgrenciEposta = "mehmet.demir@ogr.ornekuni.edu.tr",
                OgrenciTc = "21199630090",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Ogrenci
            {
                OgrenciId = 3, BolumId = 2,
                OgrenciAd = "Zeynep", OgrenciSoyad = "Kaya",
                OgrenciSinif = 4,
                OgrenciDogumTarihi = new DateTime(2002, 1, 30),
                OgrenciCinsiyet = "Kadın",
                OgrenciAdres = "Üsküdar / İstanbul",
                OgrenciTelefon = "05321110003",
                OgrenciEposta = "zeynep.kaya@ogr.ornekuni.edu.tr",
                OgrenciTc = "26706917794",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Ogrenci
            {
                OgrenciId = 4, BolumId = 3,
                OgrenciAd = "Emre", OgrenciSoyad = "Şahin",
                OgrenciSinif = 1,
                OgrenciDogumTarihi = new DateTime(2005, 6, 8),
                OgrenciCinsiyet = "Erkek",
                OgrenciAdres = "Beylikdüzü / İstanbul",
                OgrenciTelefon = "05321110004",
                OgrenciEposta = "emre.sahin@ogr.ornekuni.edu.tr",
                OgrenciTc = "39294742426",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Ogrenci
            {
                OgrenciId = 5, BolumId = 4,
                OgrenciAd = "Gülşah", OgrenciSoyad = "Öztürk",
                OgrenciSinif = 3,
                OgrenciDogumTarihi = new DateTime(2003, 11, 17),
                OgrenciCinsiyet = "Kadın",
                OgrenciAdres = "Ataşehir / İstanbul",
                OgrenciTelefon = "05321110005",
                OgrenciEposta = "gulsah.ozturk@ogr.ornekuni.edu.tr",
                OgrenciTc = "30202008634",
                CreatedDate = sabitTarih, AktifMi = true
            }
        );

        modelBuilder.Entity<Akademisyen>().HasData(
            new Akademisyen
            {
                AkademisyenId = 1, BolumId = 1,
                AkademisyenAd = "Hakan", AkademisyenSoyad = "Güneş",
                Unvan = "Prof. Dr.",
                AkademisyenDogumTarihi = new DateTime(1978, 3, 15),
                AkademisyenCinsiyet = "Erkek",
                AkademisyenAdres = "Beşiktaş / İstanbul",
                AkademisyenTelefon = "05339990001",
                AkademisyenEposta = "hakan.gunes@ornekuni.edu.tr",
                AkademisyenTc = "24449676978",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Akademisyen
            {
                AkademisyenId = 2, BolumId = 1,
                AkademisyenAd = "Ebru", AkademisyenSoyad = "Tekin",
                Unvan = "Doç. Dr.",
                AkademisyenDogumTarihi = new DateTime(1985, 11, 2),
                AkademisyenCinsiyet = "Kadın",
                AkademisyenAdres = "Sarıyer / İstanbul",
                AkademisyenTelefon = "05339990002",
                AkademisyenEposta = "ebru.tekin@ornekuni.edu.tr",
                AkademisyenTc = "42492946718",
                CreatedDate = sabitTarih, AktifMi = true
            },
            new Akademisyen
            {
                AkademisyenId = 3, BolumId = 3,
                AkademisyenAd = "Cem", AkademisyenSoyad = "Yavuz",
                Unvan = "Dr. Öğr. Üyesi",
                AkademisyenDogumTarihi = new DateTime(1983, 5, 24),
                AkademisyenCinsiyet = "Erkek",
                AkademisyenAdres = "Maltepe / İstanbul",
                AkademisyenTelefon = "05339990003",
                AkademisyenEposta = "cem.yavuz@ornekuni.edu.tr",
                AkademisyenTc = "75359841260",
                CreatedDate = sabitTarih, AktifMi = true
            }
        );
```

### Seed verisini uygula

```bash
dotnet ef migrations add BaslangicVerisi
dotnet ef database update
```

SSMS'te kontrol:
```sql
SELECT * FROM Fakulteler;
SELECT * FROM Ogrenciler;
```

> ⭐ **Bu, migration'ın gücünün en net göründüğü an.** İkinci migration sadece `INSERT` satırları içeriyor — tabloları yeniden oluşturmuyor. EF neyin değiştiğini anladı ve sadece farkı üretti. `Migrations/..._BaslangicVerisi.cs` dosyasını aç ve göster.

---

## ⌨️ Adım 4: Değişiklik denemesi

Migration'ın asıl faydasını görmek için canlı bir deney yapın.

**1.** `Fakulte` sınıfına yeni bir özellik ekle:

```csharp
    [StringLength(20)]
    [Display(Name = "Kısa ad")]
    public string? KisaAd { get; set; }
```

**2.** Migration üret:
```bash
dotnet ef migrations add FakulteKisaAdEklendi
```

**3.** Üretilen dosyayı aç:
```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(
        name: "KisaAd",
        table: "Fakulteler",
        type: "nvarchar(20)",
        maxLength: 20,
        nullable: true);
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.DropColumn(name: "KisaAd", table: "Fakulteler");
}
```

**4.** Uygula:
```bash
dotnet ef database update
```

> **Sınıfa sor:** *"Bu işi ADO.NET projesinde yapmak kaç adım sürerdi? Kaç dosyaya dokunurduk?"*

**5.** Geri al (isteğe bağlı deney):
```bash
# Bir önceki migration'a dön
dotnet ef database update BaslangicVerisi

# Migration dosyasını da sil
dotnet ef migrations remove
```

---

## 📋 Migration komutları — özet kart

Öğrenciye dağıt.

| İş | .NET CLI | Package Manager Console |
|---|---|---|
| Migration üret | `dotnet ef migrations add <Ad>` | `Add-Migration <Ad>` |
| Veritabanına uygula | `dotnet ef database update` | `Update-Database` |
| Belirli bir sürüme dön | `dotnet ef database update <Ad>` | `Update-Database <Ad>` |
| Son migration'ı sil | `dotnet ef migrations remove` | `Remove-Migration` |
| Migration listesi | `dotnet ef migrations list` | `Get-Migration` |
| Üretilecek SQL'i gör | `dotnet ef migrations script` | `Script-Migration` |
| Veritabanını sil | `dotnet ef database drop` | `Drop-Database` |

### ⚠️ Altın kural

> **Veritabanına uygulanmış bir migration dosyasını ELLE DEĞİŞTİRME veya SİLME.**
>
> Değişiklik gerekiyorsa yeni bir migration üret. Uygulanmış migration'ı silmek, `__EFMigrationsHistory` ile gerçek şemayı uyumsuz hâle getirir ve düzeltmesi zordur.
>
> Henüz `database update` yapmadıysanız `migrations remove` ile silebilirsiniz — o güvenlidir.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `No DbContext was found` | Design paketi yok veya proje derlenmiyor | `dotnet build` ile hatayı gör |
| `Cannot open database "UbysDB"` | Veritabanı yok | `dotnet ef database update` |
| `There is already an object named 'Fakulteler'` | Tablo elle oluşturulmuş | `dotnet ef database drop` sonra `update` |
| `The model has pending changes` | Model değişti, migration üretilmedi | `migrations add` çalıştır |
| Her `migrations add` boş migration üretiyor | Seed'de `DateTime.Now` kullanılmış | Sabit tarih kullan |
| `PendingModelChangesWarning` | Aynı sebep | Sabit tarih kullan |
| `Build failed` | Kod hatası var | Önce derle, sonra migration |
| `String or binary data would be truncated` | Seed verisi `StringLength`'i aşıyor | Veriyi kısalt veya sınırı büyüt |

---

## 💬 Sınıf tartışması: Code First mi, Database First mi?

İki yaklaşım var:

| | Code First | Database First |
|---|---|---|
| Başlangıç | C# sınıfları | Var olan veritabanı |
| Veritabanı | Koddan üretilir | Zaten vardır |
| Uygun olduğu durum | Yeni proje | Eski sisteme bağlanma |
| Komut | `migrations add` | `dbcontext scaffold` |

Biz **Code First** kullanıyoruz. Ama ADO.NET projesindeki hazır `OkulDB`'ye bağlanmak isteseydik:

```bash
dotnet ef dbcontext scaffold "Server=...;Database=OkulDB;..." Microsoft.EntityFrameworkCore.SqlServer -o Models
```

Bu komut var olan tabloları okur ve C# sınıflarını **sizin yerinize üretir.**

> **Sınıfa sor:** *"Bir bankada 20 yıllık bir veritabanı var ve siz yeni bir arayüz yazacaksınız. Hangisini kullanırsınız?"*
> Database First. Var olan şemaya dokunamazsınız, ona uymak zorundasınız.

---

## ✏️ Öğrenci alıştırması

1. İki migration'ı üret ve uygula. SSMS'te tabloların ve verilerin geldiğini doğrula.
2. `Ogrenci` sınıfına `Not` adında `decimal?` bir alan ekle, migration üret, üretilen dosyayı **oku**, uygula. Sonra geri al.
3. `dotnet ef migrations script` çalıştır. Çıkan SQL'i incele — bu, elle yazacağın script'in aynısı mı?
4. **Düşünme soruları:**
   - `__EFMigrationsHistory` tablosunu silseydik ne olurdu?
   - Migration dosyaları git'e eklenmeli mi? Neden?
   - Takım arkadaşınla aynı anda ikiniz de migration üretirseniz ne olur?

---

👉 Sonraki: [`04-linq-ve-ilk-liste.md`](04-linq-ve-ilk-liste.md)
