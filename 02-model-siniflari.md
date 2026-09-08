# Modül 1 — Model Sınıfları ve İlişkiler

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Dört tablonun C# karşılığını yazacağız. **Yeni kavram: navigasyon özellikleri.**

---

## 📖 Kavram: Entity nedir?

**Entity** = veritabanındaki bir tabloya karşılık gelen C# sınıfı.

ADO.NET'te de model sınıflarımız vardı. Fark ne?

| | ADO.NET model | EF entity |
|---|---|---|
| Amaç | Veriyi taşımak | Veriyi taşımak **+ tabloyu tanımlamak** |
| Tablo nasıl oluşur? | Elle `CREATE TABLE` yazarız | **EF bu sınıftan üretir** |
| İlişki | Sadece `FakulteId` (sayı) | `FakulteId` **+ `Fakulte` nesnesi** |
| Sütun tipleri | SQL'de belirlenir | C# tipi + öznitelikler belirler |

> **Kritik fark:** Artık `CREATE TABLE` yazmıyoruz. Sınıfı yazıyoruz, tablo ondan üretiliyor. Buna **Code First** denir.

---

## 📖 Kavram: Navigasyon özelliği

Bu, EF'in en çok işe yarayan özelliği. Dikkatli anlat.

### ADO.NET'te ilişki nasıldı?

```csharp
public class Bolum
{
    public long BolumId { get; set; }
    public long FakulteId { get; set; }      // sadece sayı
    public string BolumAdi { get; set; }

    public string FakulteAd { get; set; }    // JOIN'den elle doldurduğumuz alan
}
```

Fakülte adını almak için SQL'de `INNER JOIN` yazıp elle dolduruyorduk.

### EF'te?

```csharp
public class Bolum
{
    public long BolumId { get; set; }
    public long FakulteId { get; set; }      // yabancı anahtar
    public string BolumAdi { get; set; }

    public Fakulte? Fakulte { get; set; }    // ⭐ NAVİGASYON ÖZELLİĞİ
}
```

Artık şunu yazabiliyoruz:

```csharp
bolum.Fakulte.FakulteAd      // fakültenin adı
bolum.Fakulte.FakulteEposta  // fakültenin e-postası
```

**Nokta ile gidiyoruz.** JOIN yazmıyoruz.

### İki yönlü ilişki

Ters yönü de tanımlayabiliriz:

```csharp
public class Fakulte
{
    public long FakulteId { get; set; }
    public string FakulteAd { get; set; }

    public List<Bolum> Bolumler { get; set; } = new();   // ⭐ ters navigasyon
}
```

Şimdi:
```csharp
fakulte.Bolumler.Count           // bu fakültenin kaç bölümü var
fakulte.Bolumler[0].BolumAdi     // ilk bölümün adı
```

Tahtaya çiz:

```
   Fakulte                          Bolum
   ┌─────────────────┐              ┌──────────────────┐
   │ FakulteId       │◀─────────────│ FakulteId   (FK) │
   │ FakulteAd       │              │ BolumId          │
   │                 │              │ BolumAdi         │
   │ List<Bolum>     │─────────────▶│ Fakulte          │
   │   Bolumler      │  "çok" tarafı│   (tek nesne)    │
   └─────────────────┘              └──────────────────┘
        1 fakülte                       N bölüm
```

> ⚠️ **Uyarı:** Navigasyon özelliği yazmak, verinin **otomatik geleceği** anlamına gelmez. Onu getirmeyi ayrıca söylemeniz gerekir (`Include`). Modül 5'te göreceğiz. Bu, öğrencilerin en çok takıldığı noktadır.

---

## ⌨️ Adım 1: Fakulte

`Models/Fakulte.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace UBYS.Models;

public class Fakulte
{
    // ⭐ ANAHTAR KURALI:
    // EF, "Id" veya "<SınıfAdı>Id" adlı özelliği otomatik olarak
    // birincil anahtar kabul eder. Burada "FakulteId" → PK.
    // Ayrıca [Key] yazmaya gerek yok.
    //
    // long tipi olduğu için SQL'de BIGINT olur ve
    // otomatik artan (IDENTITY) olarak ayarlanır.
    public long FakulteId { get; set; }

    [Required(ErrorMessage = "Fakülte adı zorunludur.")]
    [StringLength(255, ErrorMessage = "En fazla 255 karakter olabilir.")]
    [Display(Name = "Fakülte adı")]
    public string FakulteAd { get; set; } = "";

    [Required(ErrorMessage = "Adres zorunludur.")]
    [StringLength(500)]
    [Display(Name = "Adres")]
    public string FakulteAdres { get; set; } = "";

    [Required(ErrorMessage = "Telefon zorunludur.")]
    [Phone(ErrorMessage = "Geçerli bir telefon numarası girin.")]
    [StringLength(20)]
    [Display(Name = "Telefon")]
    public string FakulteTelefon { get; set; } = "";

    [Required(ErrorMessage = "E-posta zorunludur.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string FakulteEposta { get; set; } = "";

    // ── Denetim alanları ─────────────────────────────────────
    [Display(Name = "Kayıt tarihi")]
    public DateTime CreatedDate { get; set; } = DateTime.Now;

    [Display(Name = "Güncelleme tarihi")]
    public DateTime? UpdatedDate { get; set; }

    // ⭐ ADO.NET sürümünde bu alan NVARCHAR(255) idi ve '1'/'0' yazıyorduk.
    //    Orada "yanlış tasarım" demiştik. Code First sayesinde
    //    artık doğrusunu yapabiliyoruz: bool → SQL'de BIT olur.
    public bool AktifMi { get; set; } = true;

    // ── Navigasyon özelliği ──────────────────────────────────
    // Bu fakülteye bağlı bölümler.
    // ⚠️ Veritabanında böyle bir SÜTUN YOK. EF bunu ilişkiden anlar.
    //
    // [NotMapped] yazmaya gerek yok — EF, koleksiyon tipindeki
    // navigasyon özelliklerini sütun sanmaz.
    public List<Bolum> Bolumler { get; set; } = new();
}
```

### 📖 Öznitelikler ne işe yarıyor?

ADO.NET'te öznitelikler **sadece form doğrulaması** içindi. EF'te **çift görev** yapıyorlar:

| Öznitelik | Form doğrulaması | Veritabanı etkisi |
|---|---|---|
| `[Required]` | Boş bırakılamaz | Sütun `NOT NULL` olur |
| `[StringLength(255)]` | En fazla 255 karakter | Sütun `NVARCHAR(255)` olur |
| `[EmailAddress]` | E-posta formatı | *(etkisi yok)* |
| `[Display(Name=)]` | Etiket metni | *(etkisi yok)* |

> **Bunu vurgula:** `[StringLength(255)]` yazmazsanız EF o sütunu `NVARCHAR(MAX)` yapar. Gereksiz yer kaplar ve indekslenemez. **Her metin alanına uzunluk verin.**

---

## ⌨️ Adım 2: Bolum

`Models/Bolum.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace UBYS.Models;

public class Bolum
{
    public long BolumId { get; set; }

    // ⭐ YABANCI ANAHTAR
    // "Fakulte" navigasyon özelliği + "FakulteId" alanı bir arada
    // olduğunda EF bunun bir ilişki olduğunu KENDİLİĞİNDEN anlar.
    // Ayrıca [ForeignKey] yazmaya gerek yok.
    [Required(ErrorMessage = "Fakülte seçmelisiniz.")]
    [Display(Name = "Bağlı olduğu fakülte")]
    public long FakulteId { get; set; }

    [Required(ErrorMessage = "Bölüm adı zorunludur.")]
    [StringLength(255)]
    [Display(Name = "Bölüm adı")]
    public string BolumAdi { get; set; } = "";

    [Required(ErrorMessage = "Adres zorunludur.")]
    [StringLength(500)]
    [Display(Name = "Adres")]
    public string BolumAdres { get; set; } = "";

    [Required(ErrorMessage = "Telefon zorunludur.")]
    [Phone(ErrorMessage = "Geçerli bir telefon numarası girin.")]
    [StringLength(20)]
    [Display(Name = "Telefon")]
    public string BolumTelefon { get; set; } = "";

    [Required(ErrorMessage = "E-posta zorunludur.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string BolumEposta { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public DateTime? UpdatedDate { get; set; }
    public bool AktifMi { get; set; } = true;

    // ── Navigasyon özellikleri ───────────────────────────────

    // "Bir" tarafı: bu bölüm HANGİ fakülteye ait?
    //
    // ⚠️ Sondaki ? ÖNEMLİ:
    //    Form gönderildiğinde bu nesne dolu gelmez (sadece FakulteId gelir).
    //    ? koymazsak [Required] gibi davranır ve ModelState geçersiz olur.
    //    Bu, EF'te en sık yaşanan hatalardan biridir — Modül 5'te tekrar değineceğiz.
    [Display(Name = "Fakülte")]
    public Fakulte? Fakulte { get; set; }

    // "Çok" tarafı: bu bölümdeki öğrenciler ve akademisyenler
    public List<Ogrenci> Ogrenciler { get; set; } = new();
    public List<Akademisyen> Akademisyenler { get; set; } = new();
}
```

### ⚠️ `Fakulte?` sondaki soru işareti — en kritik detay

Bu satırın altını çiz:

```csharp
public Fakulte? Fakulte { get; set; }
```

**Neden `?` şart?**

Kullanıcı bölüm ekleme formunu doldurup gönderdiğinde tarayıcı şunu yollar:
```
FakulteId=2&BolumAdi=Bilgisayar+Mühendisliği&...
```

`Fakulte` nesnesi gelmez — sadece id gelir. `?` olmasaydı ASP.NET Core "Fakulte alanı zorunlu" der, `ModelState.IsValid` **false** olur ve kayıt hiç yapılmaz.

> **Deney (Modül 5'te yaptır):** `?` işaretini sil, bölüm eklemeyi dene. Form sürekli geri gelir, hata mesajı da anlaşılmaz. Sonra geri koy.

---

## ⌨️ Adım 3: Ogrenci

`Models/Ogrenci.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace UBYS.Models;

public class Ogrenci
{
    public long OgrenciId { get; set; }

    [Required(ErrorMessage = "Bölüm seçmelisiniz.")]
    [Display(Name = "Bölüm")]
    public long BolumId { get; set; }

    [Required(ErrorMessage = "Ad zorunludur.")]
    [StringLength(100)]
    [Display(Name = "Ad")]
    public string OgrenciAd { get; set; } = "";

    [Required(ErrorMessage = "Soyad zorunludur.")]
    [StringLength(100)]
    [Display(Name = "Soyad")]
    public string OgrenciSoyad { get; set; } = "";

    [Required(ErrorMessage = "Sınıf zorunludur.")]
    [Range(1, 6, ErrorMessage = "Sınıf 1 ile 6 arasında olmalıdır.")]
    [Display(Name = "Sınıf")]
    public int OgrenciSinif { get; set; }

    [Required(ErrorMessage = "Doğum tarihi zorunludur.")]
    [DataType(DataType.Date)]
    [Display(Name = "Doğum tarihi")]
    public DateTime OgrenciDogumTarihi { get; set; }

    [Required(ErrorMessage = "Cinsiyet seçmelisiniz.")]
    [StringLength(10)]
    [Display(Name = "Cinsiyet")]
    public string OgrenciCinsiyet { get; set; } = "";

    [Required(ErrorMessage = "Adres zorunludur.")]
    [StringLength(500)]
    [Display(Name = "Adres")]
    public string OgrenciAdres { get; set; } = "";

    [Required(ErrorMessage = "Telefon zorunludur.")]
    [Phone(ErrorMessage = "Geçerli bir telefon numarası girin.")]
    [StringLength(20)]
    [Display(Name = "Telefon")]
    public string OgrenciTelefon { get; set; } = "";

    [Required(ErrorMessage = "E-posta zorunludur.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string OgrenciEposta { get; set; } = "";

    [Required(ErrorMessage = "TC kimlik numarası zorunludur.")]
    [RegularExpression(@"^[1-9][0-9]{10}$",
        ErrorMessage = "TC kimlik numarası 11 haneli olmalı ve 0 ile başlamamalıdır.")]
    [StringLength(11)]
    [Display(Name = "TC kimlik no")]
    public string OgrenciTc { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public DateTime? UpdatedDate { get; set; }
    public bool AktifMi { get; set; } = true;

    // ── Navigasyon ───────────────────────────────────────────
    [Display(Name = "Bölüm")]
    public Bolum? Bolum { get; set; }

    // ── Hesaplanan özellik ───────────────────────────────────
    // ⚠️ [NotMapped] ŞART!
    //    Olmazsa EF bunu bir sütun sanır ve tabloya "TamAd" ekler.
    //    Sadece "get" olan özellikleri EF zaten atlar ama
    //    niyeti açıkça belirtmek daha güvenlidir.
    [System.ComponentModel.DataAnnotations.Schema.NotMapped]
    public string TamAd => OgrenciAd + " " + OgrenciSoyad;
}
```

### 📖 `[NotMapped]`

Bu öznitelik EF'e der ki: *"Bu özelliği görmezden gel, tabloda sütunu olmasın."*

Nerede gerekir?
- Hesaplanan özellikler (`TamAd`, `Yas`)
- Sadece ekranda kullanılan geçici alanlar
- Formdan gelen ama saklanmayacak veriler

> ADO.NET'te bu sorun yoktu — SQL'i biz yazdığımız için o alanı hiç yazmıyorduk. EF'te ise "her özellik bir sütundur" varsayımı geçerli, istisnaları belirtmek gerekiyor.

---

## ⌨️ Adım 4: Akademisyen

`Models/Akademisyen.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace UBYS.Models;

public class Akademisyen
{
    public long AkademisyenId { get; set; }

    [Required(ErrorMessage = "Bölüm seçmelisiniz.")]
    [Display(Name = "Bölüm")]
    public long BolumId { get; set; }

    [Required(ErrorMessage = "Ad zorunludur.")]
    [StringLength(100)]
    [Display(Name = "Ad")]
    public string AkademisyenAd { get; set; } = "";

    [Required(ErrorMessage = "Soyad zorunludur.")]
    [StringLength(100)]
    [Display(Name = "Soyad")]
    public string AkademisyenSoyad { get; set; } = "";

    [StringLength(50)]
    [Display(Name = "Unvan")]
    public string? Unvan { get; set; }          // isteğe bağlı → NULL olabilir

    [Required(ErrorMessage = "Doğum tarihi zorunludur.")]
    [DataType(DataType.Date)]
    [Display(Name = "Doğum tarihi")]
    public DateTime AkademisyenDogumTarihi { get; set; }

    [Required(ErrorMessage = "Cinsiyet seçmelisiniz.")]
    [StringLength(10)]
    [Display(Name = "Cinsiyet")]
    public string AkademisyenCinsiyet { get; set; } = "";

    [Required(ErrorMessage = "Adres zorunludur.")]
    [StringLength(500)]
    [Display(Name = "Adres")]
    public string AkademisyenAdres { get; set; } = "";

    [Required(ErrorMessage = "Telefon zorunludur.")]
    [Phone(ErrorMessage = "Geçerli bir telefon numarası girin.")]
    [StringLength(20)]
    [Display(Name = "Telefon")]
    public string AkademisyenTelefon { get; set; } = "";

    [Required(ErrorMessage = "E-posta zorunludur.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string AkademisyenEposta { get; set; } = "";

    [Required(ErrorMessage = "TC kimlik numarası zorunludur.")]
    [RegularExpression(@"^[1-9][0-9]{10}$",
        ErrorMessage = "TC kimlik numarası 11 haneli olmalı ve 0 ile başlamamalıdır.")]
    [StringLength(11)]
    [Display(Name = "TC kimlik no")]
    public string AkademisyenTc { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public DateTime? UpdatedDate { get; set; }
    public bool AktifMi { get; set; } = true;

    [Display(Name = "Bölüm")]
    public Bolum? Bolum { get; set; }

    [NotMapped]
    public string TamAd => (Unvan == null ? "" : Unvan + " ") + AkademisyenAd + " " + AkademisyenSoyad;
}
```

---

## ⌨️ Adım 5: Kullanici

Giriş sistemi için (Modül 10'da kullanacağız):

```csharp
using System.ComponentModel.DataAnnotations;

namespace UBYS.Models;

public class Kullanici
{
    public long KullaniciId { get; set; }

    [Required]
    [StringLength(100)]
    [Display(Name = "Kullanıcı adı")]
    public string KullaniciAdi { get; set; } = "";

    [Required]
    [StringLength(255)]
    public string SifreHash { get; set; } = "";     // şifrenin kendisi değil, özeti

    [Required]
    [StringLength(255)]
    [Display(Name = "Ad soyad")]
    public string AdSoyad { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public bool AktifMi { get; set; } = true;
}


/// <summary>
/// Giriş formunun taşıyıcısı — veritabanı tablosu DEĞİL.
/// Bu yüzden DbContext'te DbSet'i yok.
/// </summary>
public class GirisViewModel
{
    [Required(ErrorMessage = "Kullanıcı adı gerekli.")]
    [Display(Name = "Kullanıcı adı")]
    public string KullaniciAdi { get; set; } = "";

    [Required(ErrorMessage = "Şifre gerekli.")]
    [DataType(DataType.Password)]
    [Display(Name = "Şifre")]
    public string Sifre { get; set; } = "";

    [Display(Name = "Beni hatırla")]
    public bool BeniHatirla { get; set; }
}
```

---

## ⌨️ Adım 6: Benzersizlik kısıtları — Fluent API

Öznitelikle yapılamayan bazı ayarlar var. Bunlar için `DbContext` içinde **Fluent API** kullanılır.

`Data/UbysDbContext.cs`'e ekle:

```csharp
    /// <summary>
    /// Öznitelikle anlatılamayan model ayarları burada yapılır.
    /// EF, migration üretirken bu metodu çalıştırır.
    /// </summary>
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // ── BENZERSİZLİK İNDEKSLERİ ──────────────────────────
        // ADO.NET sürümünde bunları elle CREATE UNIQUE INDEX ile yazmıştık.
        // Şimdi burada tanımlıyoruz, migration onları üretecek.

        modelBuilder.Entity<Fakulte>()
            .HasIndex(f => f.FakulteEposta)
            .IsUnique();

        modelBuilder.Entity<Bolum>()
            .HasIndex(b => b.BolumEposta)
            .IsUnique();

        modelBuilder.Entity<Ogrenci>()
            .HasIndex(o => o.OgrenciEposta)
            .IsUnique();

        modelBuilder.Entity<Ogrenci>()
            .HasIndex(o => o.OgrenciTc)
            .IsUnique();

        modelBuilder.Entity<Akademisyen>()
            .HasIndex(a => a.AkademisyenEposta)
            .IsUnique();

        modelBuilder.Entity<Akademisyen>()
            .HasIndex(a => a.AkademisyenTc)
            .IsUnique();

        modelBuilder.Entity<Kullanici>()
            .HasIndex(k => k.KullaniciAdi)
            .IsUnique();

        // ── SİLME DAVRANIŞI ──────────────────────────────────
        // Varsayılan: Cascade (fakülte silinirse bölümleri de silinir)
        // Biz bunu İSTEMİYORUZ — zaten soft delete kullanacağız.
        // Restrict: bağlı kaydı olan satır silinemez.

        modelBuilder.Entity<Bolum>()
            .HasOne(b => b.Fakulte)
            .WithMany(f => f.Bolumler)
            .HasForeignKey(b => b.FakulteId)
            .OnDelete(DeleteBehavior.Restrict);

        modelBuilder.Entity<Ogrenci>()
            .HasOne(o => o.Bolum)
            .WithMany(b => b.Ogrenciler)
            .HasForeignKey(o => o.BolumId)
            .OnDelete(DeleteBehavior.Restrict);

        modelBuilder.Entity<Akademisyen>()
            .HasOne(a => a.Bolum)
            .WithMany(b => b.Akademisyenler)
            .HasForeignKey(a => a.BolumId)
            .OnDelete(DeleteBehavior.Restrict);
    }
```

### 📖 Öznitelik mi, Fluent API mi?

| | Öznitelik | Fluent API |
|---|---|---|
| Nerede | Model sınıfının içinde | `OnModelCreating` içinde |
| Okunabilirlik | Alanın yanında, kolay görülür | Ayrı yerde |
| Güç | Sınırlı | Her şey yapılabilir |
| Örnek | `[Required]`, `[StringLength]` | Benzersiz indeks, silme davranışı, birleşik anahtar |

> **Kural:** Basit şeyler öznitelikle, öznitelikle yapılamayanlar Fluent API ile. İkisi bir arada kullanılabilir; çakışırsa Fluent API kazanır.

### 📖 `DeleteBehavior` seçenekleri

| Değer | Fakülte silinince bölümler |
|---|---|
| `Cascade` | Otomatik silinir *(varsayılan — tehlikeli!)* |
| `Restrict` | Silmeye izin verilmez, hata verir ✅ *bizim seçimimiz* |
| `SetNull` | `FakulteId` NULL olur *(alan nullable olmalı)* |
| `NoAction` | Veritabanına bırakılır |

> **Sınıfa sor:** *"Cascade bıraksaydık ve biri yanlışlıkla bir fakülteyi silseydi ne olurdu?"*
> O fakültenin tüm bölümleri, bölümlerin tüm öğrencileri ve akademisyenleri **zincirleme silinirdi.** Tek tıkla binlerce kayıt gider. Bu yüzden `Restrict` seçiyoruz.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `The entity type requires a primary key` | Anahtar bulunamadı | `<SınıfAdı>Id` veya `Id` adında özellik olmalı, ya da `[Key]` ekle |
| Form hep geçersiz, sebebi belirsiz | Navigasyon özelliğinde `?` yok | `public Fakulte? Fakulte` yap |
| Tabloda beklenmedik bir sütun var | Hesaplanan özellik sütun sanılmış | `[NotMapped]` ekle |
| Metin sütunları `NVARCHAR(MAX)` olmuş | `[StringLength]` yazılmamış | Her metin alanına ekle |
| `Introducing FOREIGN KEY constraint may cause cycles` | Cascade döngüsü | `OnDelete(DeleteBehavior.Restrict)` |
| `Unable to determine the relationship` | İlişki belirsiz | Fluent API ile `HasOne/WithMany` yaz |

---

## ✏️ Öğrenci alıştırması

1. Beş model sınıfını ve `OnModelCreating` metodunu yaz. Proje **derlenmeli**.
2. `Fakulte` sınıfındaki `Bolumler` listesini silersen ne olur? Dene, derleme hatasını oku.
3. **Düşünme sorusu:** `Bolum` sınıfında hem `FakulteId` hem `Fakulte` var. İkisi de aynı bilgiyi taşıyor gibi. Neden ikisi birden gerekli?

---

👉 Sonraki: [`03-migration-ve-veritabani.md`](03-migration-ve-veritabani.md)
