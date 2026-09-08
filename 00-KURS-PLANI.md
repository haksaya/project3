# UBYS — Entity Framework Core ile

**Üniversite Bilgi Yönetim Sistemi**
ASP.NET Core MVC + **EF Core** + Bootstrap 5

---

## Bu rehber neden var?

Aynı projeyi daha önce **ADO.NET** ile yaptık. Her `SELECT`, her `INSERT` bizim elimizden çıktı. Şimdi aynı projeyi **Entity Framework Core** ile yeniden yazacağız.

> **İlk derste sınıfa söyle:**
> *"Bugün yeni bir uygulama öğrenmiyoruz. Aynı uygulamayı, SQL yazmadan yapacağız. Konu değişmedi — sadece veritabanıyla konuşma şeklimiz değişti. Bu yüzden dikkatinizi tek bir soruya verin: 'Bunu ADO.NET'te nasıl yazıyorduk?'"*

**Ön koşul:** Öğrenci ADO.NET sürümünü tamamlamış olmalı. Bu rehber SQL'i, MVC'yi, Bootstrap'i bildiğinizi varsayar ve **yalnızca değişen kısımlara** odaklanır.

---

## Entity Framework nedir?

**ORM** = Object-Relational Mapper. Nesne ile tabloyu birbirine eşleyen araç.

```
   C# DÜNYASI                          VERİTABANI DÜNYASI
   ──────────                          ──────────────────
   Fakulte sınıfı        ◀── EF ──▶    Fakulteler tablosu
   Fakulte nesnesi       ◀── EF ──▶    bir satır
   fakulte.FakulteAd     ◀── EF ──▶    FakulteAd sütunu
   List<Fakulte>         ◀── EF ──▶    SELECT sonucu
```

**Kısaca:** Siz C# yazarsınız, EF onu SQL'e çevirir, çalıştırır, sonucu tekrar C# nesnesine çevirir.

---

## ADO.NET ile EF Core — yan yana

Bu tablo rehberin özetidir. Tahtaya çiz ve dersin sonuna kadar silme.

| İş | ADO.NET (eski) | EF Core (yeni) |
|---|---|---|
| **Bağlantı** | `new SqlConnection(...)` + `Open()` | `DbContext` — açma/kapama otomatik |
| **Listeleme** | `SELECT ...` + `while(Read())` + elle nesneye çevir | `_db.Fakulteler.ToList()` |
| **Tek kayıt** | `SELECT ... WHERE id=@id` | `_db.Fakulteler.Find(id)` |
| **Ekleme** | `INSERT INTO ...` + parametreler | `_db.Add(f); _db.SaveChanges();` |
| **Güncelleme** | `UPDATE ... SET ... WHERE` | Nesneyi değiştir + `SaveChanges()` |
| **Silme** | `DELETE` / `UPDATE is_active` | `_db.Remove(f)` / query filter |
| **JOIN** | SQL'de `INNER JOIN` yaz | `.Include(b => b.Fakulte)` |
| **Sayma** | `SELECT COUNT(*)` + `ExecuteScalar` | `.Count()` |
| **Filtreleme** | Dinamik SQL metni kur | `.Where(x => ...)` |
| **Sayfalama** | `OFFSET ... FETCH NEXT` | `.Skip(n).Take(m)` |
| **Tabloyu oluşturma** | Elle `CREATE TABLE` | **Migration** — koddan üretilir |
| **SQL injection** | Parametre kullanmak zorunlu | EF **otomatik** parametreler |
| **Satır → nesne** | `GetString(GetOrdinal(...))` elle | Otomatik |
| **NULL kontrolü** | `IsDBNull(...)` elle | `?` ile model'de tanımlı, otomatik |

### Ne kazandık, ne kaybettik?

| Kazandık | Kaybettik |
|---|---|
| Çok daha az kod | Üretilen SQL'i doğrudan görmüyoruz |
| Yazım hatası derleme zamanında yakalanır | Kötü LINQ = kötü SQL, farkına varmak zor |
| SQL injection'a karşı otomatik koruma | Öğrenme eğrisi (yeni kavramlar) |
| Veritabanı değiştirmek kolaylaşır | Bazı karmaşık sorgular zorlaşır |
| Migration ile şema versiyonlanır | Sihir hissi — ne olduğunu bilmezsen tehlikeli |

> **Kritik cümle:** *"EF, SQL'i sizin yerinize yazar. Ama SQL bilmiyorsanız, EF'in yazdığı kötü SQL'i fark edemezsiniz. Bu yüzden önce ADO.NET yaptık."*

---

## Bu rehberde öğrenilecek yeni kavramlar

| Kavram | Modül |
|---|---|
| `DbContext` — veritabanı oturumu | 1 |
| `DbSet<T>` — tablo karşılığı | 1 |
| Entity sınıfı ve navigasyon özellikleri | 2 |
| Migration — koddan veritabanı üretme | 3 |
| LINQ ve `IQueryable` | 4 |
| Ertelenmiş çalıştırma (deferred execution) | 4 |
| Değişiklik takibi (change tracking) | 5 |
| `SaveChanges()` — tek işlem (transaction) | 5 |
| `Include()` — eager loading | 6 |
| Global query filter — soft delete | 6 |
| `DbUpdateException` yakalama | 7 |
| `GroupBy` ve projeksiyon | 9 |
| `AsNoTracking()` — performans | 12 |
| N+1 problemi ve çözümü | 12 |
| Üretilen SQL'i loglama | 4, 12 |

---

## Ders takvimi

**10-12 ders saati.** ADO.NET sürümü 14-16 saatti — farkı öğrenciye göster.

| # | Modül | Süre | Dosya |
|---|---|---|---|
| 0 | Kurulum ve DbContext | 1 saat | `01-kurulum-ve-dbcontext.md` |
| 1 | Model sınıfları ve ilişkiler | 1 saat | `02-model-siniflari.md` |
| 2 | Migration ve veritabanı | 1 saat | `03-migration-ve-veritabani.md` |
| 3 | LINQ ve ilk liste | 1 saat | `04-linq-ve-ilk-liste.md` |
| 4 | **Fakülte CRUD** | 2 saat | `05-fakulte-crud.md` |
| 5 | **Bölüm CRUD** (ilişki) | 1-2 saat | `06-bolum-crud.md` |
| 6 | **Öğrenci CRUD** (doğrulama) | 1-2 saat | `07-ogrenci-crud.md` |
| 7 | **Akademisyen CRUD** | 1 saat | `08-akademisyen-crud.md` |
| 8 | Dashboard (LINQ ile istatistik) | 1 saat | `09-dashboard.md` |
| 9 | Arama, filtreleme, sayfalama | 1 saat | `10-arama-filtreleme-sayfalama.md` |
| 10 | Giriş sistemi | 1 saat | `11-login.md` |
| 11 | EF ipuçları ve tuzaklar | 1 saat | `12-ef-ipuclari-ve-tuzaklar.md` |

---

## Proje yapısı

ADO.NET sürümüyle karşılaştır — **`Data` klasörü değişti:**

```
UBYS/
│
├── Program.cs
├── appsettings.json
│
├── Data/
│   └── UbysDbContext.cs        ⭐ 4 repository yerine TEK dosya
│
├── Migrations/                 ⭐ YENİ — EF üretir, elle dokunulmaz
│   ├── 20260901_IlkOlusturma.cs
│   └── UbysDbContextModelSnapshot.cs
│
├── Models/
│   ├── Fakulte.cs
│   ├── Bolum.cs
│   ├── Ogrenci.cs
│   ├── Akademisyen.cs
│   ├── Kullanici.cs
│   └── DashboardViewModel.cs
│
├── Controllers/
│   ├── HomeController.cs
│   ├── FakulteController.cs
│   ├── BolumController.cs
│   ├── OgrenciController.cs
│   ├── AkademisyenController.cs
│   └── HesapController.cs
│
└── Views/                      (ADO.NET sürümüyle neredeyse aynı)
```

> **Dikkat çek:** `FakulteRepository`, `BolumRepository`, `OgrenciRepository`, `AkademisyenRepository` — dört dosya **yok oldu**. Yerine tek bir `UbysDbContext` geldi. Sebebini Modül 0'da anlatacağız.

---

## Sürüm bilgisi

Rehber **.NET 10 + EF Core 10** için yazıldı.

.NET 8 kullanıyorsanız EF Core 8 kurun — bu rehberdeki **tüm kodlar aynen çalışır**. Fark yalnızca paket sürüm numaralarındadır.

---

## Sonraki adım

👉 [`01-kurulum-ve-dbcontext.md`](01-kurulum-ve-dbcontext.md)
