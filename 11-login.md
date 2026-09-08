# Modül 10 — Giriş Sistemi

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Sisteme giriş ekranı ekleyeceğiz. Giriş yapmayan hiçbir sayfayı göremeyecek.

> **EF açısından yeni bir şey yok** — aynı `DbContext`, aynı LINQ. Bu modül ADO.NET sürümüyle neredeyse birebir aynı. Değişen sadece kullanıcıyı okuma şekli.

Tek kullanıcı, tek yetki seviyesi. Rol sistemi yok.

---

## 📖 Kavram: Şifreler asla düz metin saklanmaz

**Sınıfa sor:** *"Şifreyi veritabanına olduğu gibi yazsak ne olur?"*

1. Veritabanı sızarsa herkesin şifresi açığa çıkar
2. Veritabanına erişimi olan herkes — **siz dâhil** — şifreleri görür
3. İnsanlar aynı şifreyi başka sitelerde kullanır → zincirleme felaket

**Çözüm: hash (özet)** — geri çevrilemez tek yönlü dönüşüm.

```
"ubys123"  ──SHA256──▶  "64 karakterlik özet"
                               ▲
                    buradan geriye DÖNÜLEMEZ
```

Girişte kullanıcının yazdığını tekrar hash'ler, saklananla karşılaştırırız.

> ⚠️ **Dürüst uyarı — mutlaka söyle:**
> SHA256 bu ders için uygun ama **gerçek projede şifre için yeterli değil.** İki sebeple: çok *hızlı* (saldırgan saniyede milyarlarca deneme yapar) ve *tuz* içermiyor (aynı şifre hep aynı hash'i verir). Gerçekte BCrypt, Argon2 veya ASP.NET Core Identity kullanılır.
>
> Bunu söylemezsen öğrenci SHA256'yı doğru yöntem sanır.

---

## ⌨️ Adım 1: Şifre yardımcısı

`Data/SifreYardimcisi.cs`:

```csharp
using System.Security.Cryptography;
using System.Text;

namespace UBYS.Data;

/// <summary>
/// Şifre hash'leme işlemleri.
///
/// ⚠️ ÖĞRETİM AMAÇLIDIR. Gerçek projede BCrypt/Argon2 kullanın.
/// </summary>
public static class SifreYardimcisi
{
    /// <summary>
    /// Metnin SHA256 özetini büyük harfli onaltılık metin olarak döner.
    ///
    /// Derste çalıştırıp göster:
    ///     SifreYardimcisi.Hashle("ubys123")
    /// Çıktı, veritabanındaki değerle aynı olmalı.
    /// </summary>
    public static string Hashle(string sifre)
    {
        byte[] bayt = Encoding.UTF8.GetBytes(sifre);   // metni bayta çevir
        byte[] hash = SHA256.HashData(bayt);           // özeti hesapla
        return Convert.ToHexString(hash);              // okunabilir metne çevir
    }
}
```

---

## ⌨️ Adım 2: Admin hesabını seed ile ekle

`UbysDbContext.OnModelCreating` metoduna ekle:

```csharp
        // ══════════════════════════════════════════════════════
        //  ADMIN HESABI
        //
        //  Kullanıcı adı: admin
        //  Şifre        : ubys123
        //
        //  ⚠️ Hash'i SifreYardimcisi.Hashle("ubys123") ile
        //     üretip buraya yazacağız. Sabit olmak zorunda —
        //     seed verisi migration'a gömülür.
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Kullanici>().HasData(
            new Kullanici
            {
                KullaniciId = 1,
                KullaniciAdi = "admin",
                SifreHash = SifreYardimcisi.Hashle("ubys123"),
                AdSoyad = "Sistem Yöneticisi",
                CreatedDate = sabitTarih,     // Modül 2'de tanımladığımız değişken
                AktifMi = true
            }
        );
```

> **`SifreYardimcisi.Hashle("ubys123")` çağrısı burada güvenli** çünkü SHA256 deterministiktir — her seferinde aynı sonucu verir. Migration üretilirken bir kez hesaplanır ve sabit metin olarak dosyaya yazılır. `DateTime.Now` gibi her seferinde değişen bir şey değil.

Migration üret ve uygula:

```bash
dotnet ef migrations add AdminKullanicisi
dotnet ef database update
```

Üretilen migration dosyasını **aç ve göster** — hash'in sabit metin olarak yazıldığını görecekler:

```csharp
migrationBuilder.InsertData(
    table: "Kullanicilar",
    columns: new[] { "KullaniciId", "AdSoyad", "AktifMi", "CreatedDate", "KullaniciAdi", "SifreHash" },
    values: new object[] { 1L, "Sistem Yöneticisi", true, new DateTime(2026, 1, 1, 9, 0, 0), "admin", "A3F5..." });
```

---

## ⌨️ Adım 3: Controller

`Controllers/HesapController.cs`:

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;
using UBYS.Models;

namespace UBYS.Controllers;

// ⭐ [AllowAnonymous] ŞART!
//    Program.cs'te tüm uygulamayı "giriş zorunlu" yapacağız.
//    Bu controller muaf olmazsa, giriş sayfasına girmek için
//    giriş yapmak gerekir → SONSUZ DÖNGÜ.
[AllowAnonymous]
public class HesapController : Controller
{
    private readonly UbysDbContext _db;

    public HesapController(UbysDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  GİRİŞ FORMU
    //  GET: /Hesap/Giris
    // ══════════════════════════════════════════════════════
    [HttpGet]
    public IActionResult Giris(string? donusUrl = null)
    {
        // Zaten girmişse formu gösterme
        if (User.Identity != null && User.Identity.IsAuthenticated)
            return RedirectToAction("Index", "Home");

        ViewBag.DonusUrl = donusUrl;
        return View();
    }

    // ══════════════════════════════════════════════════════
    //  GİRİŞİ İŞLE
    //  POST: /Hesap/Giris
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Giris(GirisViewModel model, string? donusUrl = null)
    {
        ViewBag.DonusUrl = donusUrl;

        if (!ModelState.IsValid)
            return View(model);

        // ⭐ EF ile doğrulama
        //    Karşılaştırmayı VERİTABANINDA yapıyoruz:
        //    "bu kullanıcı adı VE bu hash'e sahip satır var mı?"
        //    Böylece hash hiç uygulamaya taşınmıyor.
        //
        //    ⚠️ SQL injection endişesi YOK — EF her değeri
        //       otomatik parametreye çevirir. Şifre alanına
        //       ' OR '1'='1 yazsalar bile sadece metin olarak aranır.
        string hash = SifreYardimcisi.Hashle(model.Sifre);

        Kullanici? kullanici = await _db.Kullanicilar
            .FirstOrDefaultAsync(k => k.KullaniciAdi == model.KullaniciAdi
                                   && k.SifreHash == hash);

        if (kullanici == null)
        {
            // ⚠️⚠️ "Kullanıcı yok" ile "şifre yanlış"ı AYIRMA!
            //    Ayrı söyleseydik saldırgan, sisteme kayıtlı kullanıcı
            //    adlarını tek tek deneyerek öğrenirdi.
            //    Buna "kullanıcı sayımı" (user enumeration) denir.
            //    Belirsizlik KASITLIDIR.
            ModelState.AddModelError("", "Kullanıcı adı veya şifre hatalı.");
            return View(model);
        }

        // Claim = kullanıcı hakkında bir bilgi parçası.
        // Çereze şifrelenerek yazılır, her istekte sunucuya gelir.
        var iddialar = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, kullanici.KullaniciId.ToString()),
            new Claim(ClaimTypes.Name, kullanici.AdSoyad),
            new Claim("KullaniciAdi", kullanici.KullaniciAdi)
        };

        var kimlik = new ClaimsIdentity(iddialar,
            CookieAuthenticationDefaults.AuthenticationScheme);

        var ozellikler = new AuthenticationProperties
        {
            IsPersistent = model.BeniHatirla,   // tarayıcı kapansa da yaşasın mı?
            AllowRefresh = true
        };

        await HttpContext.SignInAsync(
            CookieAuthenticationDefaults.AuthenticationScheme,
            new ClaimsPrincipal(kimlik),
            ozellikler);

        // ⚠️ Url.IsLocalUrl kontrolü ŞART!
        //    Olmasaydı saldırgan şöyle bir bağlantı hazırlayabilirdi:
        //      /Hesap/Giris?donusUrl=https://sahte-site.com
        //    Kullanıcı giriş sonrası sahte siteye giderdi (open redirect).
        if (!string.IsNullOrEmpty(donusUrl) && Url.IsLocalUrl(donusUrl))
            return Redirect(donusUrl);

        return RedirectToAction("Index", "Home");
    }

    // ══════════════════════════════════════════════════════
    //  ÇIKIŞ
    //  POST: /Hesap/Cikis
    //
    //  ⚠️ Neden POST? Çıkış durumu DEĞİŞTİREN bir işlem.
    //     GET olsaydı kötü niyetli bir sitedeki
    //         <img src=".../Hesap/Cikis">
    //     etiketi kullanıcıyı habersizce çıkartırdı.
    // ══════════════════════════════════════════════════════
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Cikis()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        TempData["Bilgi"] = "Oturumunuz kapatıldı.";
        return RedirectToAction(nameof(Giris));
    }
}
```

---

## ⌨️ Adım 4: Giriş sayfası

`Views/Hesap/Giris.cshtml`:

```html
@model UBYS.Models.GirisViewModel
@{
    Layout = null;   @* ⭐ Giriş sayfasında sol menü olmasın *@
}

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Giriş - UBYS</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.css" rel="stylesheet">
</head>
<body class="bg-light">

    <div class="container">
        <div class="row justify-content-center align-items-center" style="min-height: 100vh;">
            <div class="col-md-4">

                <div class="card border-0 shadow">
                    <div class="card-body p-4">

                        <div class="text-center mb-4">
                            <i class="bi bi-mortarboard-fill fs-1 text-primary"></i>
                            <h5 class="mt-2 mb-1">UBYS</h5>
                            <p class="text-muted small mb-0">Devam etmek için giriş yapın</p>
                        </div>

                        @if (TempData["Bilgi"] != null)
                        {
                            <div class="alert alert-info py-2 small">
                                <i class="bi bi-info-circle"></i> @TempData["Bilgi"]
                            </div>
                        }

                        <form asp-action="Giris" method="post">

                            <input type="hidden" name="donusUrl" value="@ViewBag.DonusUrl" />

                            <div asp-validation-summary="ModelOnly"
                                 class="alert alert-danger py-2 small"></div>

                            <div class="mb-3">
                                <label asp-for="KullaniciAdi" class="form-label"></label>
                                <div class="input-group">
                                    <span class="input-group-text bg-white">
                                        <i class="bi bi-person"></i>
                                    </span>
                                    <input asp-for="KullaniciAdi" class="form-control"
                                           autocomplete="username" autofocus />
                                </div>
                                <span asp-validation-for="KullaniciAdi" class="text-danger small"></span>
                            </div>

                            <div class="mb-3">
                                <label asp-for="Sifre" class="form-label"></label>
                                <div class="input-group">
                                    <span class="input-group-text bg-white">
                                        <i class="bi bi-lock"></i>
                                    </span>
                                    <input asp-for="Sifre" type="password" class="form-control"
                                           autocomplete="current-password" />
                                </div>
                                <span asp-validation-for="Sifre" class="text-danger small"></span>
                            </div>

                            <div class="form-check mb-3">
                                <input asp-for="BeniHatirla" class="form-check-input" />
                                <label asp-for="BeniHatirla" class="form-check-label small"></label>
                            </div>

                            <button type="submit" class="btn btn-primary w-100">
                                <i class="bi bi-box-arrow-in-right"></i> Giriş yap
                            </button>

                        </form>

                    </div>
                </div>

                @* ⚠️⚠️ CANLIYA ÇIKMADAN ÖNCE SİL! *@
                <div class="alert alert-warning small mt-3 mb-0">
                    <i class="bi bi-info-circle"></i>
                    <strong>Ders ortamı:</strong> <code>admin</code> / <code>ubys123</code>
                    <div class="mt-1 text-muted" style="font-size: .75rem;">
                        Bu kutu gerçek projede silinmelidir.
                    </div>
                </div>

            </div>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
    <partial name="_ValidationScriptsPartial" />

</body>
</html>
```

---

## ⌨️ Adım 5: Program.cs

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.Authorization;
using Microsoft.EntityFrameworkCore;
using UBYS.Data;

var builder = WebApplication.CreateBuilder(args);

// ── 1. MVC + "giriş zorunlu" filtresi ───────────────────────
//
// ⭐ Neden global filtre, her controller'a [Authorize] değil?
//    Yarın yeni bir controller yazıp [Authorize] koymayı unutursan
//    o sayfa herkese açık kalır. Global filtreyle varsayılan KAPALI olur.
//    GÜVENLİK İLKESİ: varsayılan hep en kısıtlayıcı seçenek olmalı.
builder.Services.AddControllersWithViews(secenekler =>
{
    var politika = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    secenekler.Filters.Add(new AuthorizeFilter(politika));
});

// ── 2. Çerez tabanlı kimlik doğrulama ───────────────────────
builder.Services
    .AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(secenekler =>
    {
        secenekler.LoginPath = "/Hesap/Giris";
        secenekler.ReturnUrlParameter = "donusUrl";   // controller parametresiyle aynı olmalı!
        secenekler.ExpireTimeSpan = TimeSpan.FromHours(8);
        secenekler.SlidingExpiration = true;
        secenekler.Cookie.HttpOnly = true;             // JS erişemez → XSS koruması
        secenekler.Cookie.SameSite = SameSiteMode.Lax; // CSRF koruması
        secenekler.Cookie.Name = "UBYS.Oturum";
    });

// ── 3. EF Core ──────────────────────────────────────────────
builder.Services.AddDbContext<UbysDbContext>(secenekler =>
{
    secenekler.UseSqlServer(
        builder.Configuration.GetConnectionString("UbysDb"));

    if (builder.Environment.IsDevelopment())
    {
        secenekler.LogTo(Console.WriteLine, LogLevel.Information);
        secenekler.EnableSensitiveDataLogging();   // ⚠️ canlıda ASLA
    }
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

// ⭐⭐ SIRA KRİTİK
app.UseAuthentication();   // ÖNCE: "sen kimsin?" (çerezi okur)
app.UseAuthorization();    // SONRA: "girebilir mi?" (filtreyi uygular)

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

> **Ters yazma deneyi:** `UseAuthentication` ve `UseAuthorization` satırlarının yerini bilerek değiştir. Doğru şifreyle giren kullanıcı bile içeri alınmaz, üstelik hata mesajı hiçbir ipucu vermez — sadece giriş sayfasına döner durur. Bu deneyi yaptır; öğrencinin ileride saatlerini kurtarır.

---

## ⌨️ Adım 6: Layout'a kullanıcı menüsü

`Views/Shared/_Layout.cshtml`'deki üst çubuğa:

```html
<div class="d-flex justify-content-between align-items-center border-bottom pb-2 mb-4">

    <h4 class="mb-0">@ViewData["Title"]</h4>

    <div class="d-flex align-items-center gap-3">

        <span class="text-muted small d-none d-md-inline">
            <i class="bi bi-calendar3"></i> @DateTime.Now.ToString("dd MMMM yyyy")
        </span>

        @* ⭐ @User her view'da hazır bulunur — controller'dan göndermeye gerek yok.
             ASP.NET Core çerezdeki claim'leri okuyup bu nesneyi doldurur.
             HesapController'da yazdığımız
                 new Claim(ClaimTypes.Name, kullanici.AdSoyad)
             satırının karşılığı burada ekrana geliyor. *@
        @if (User.Identity != null && User.Identity.IsAuthenticated)
        {
            <div class="dropdown">
                <button class="btn btn-sm btn-outline-secondary dropdown-toggle"
                        data-bs-toggle="dropdown">
                    <i class="bi bi-person-circle"></i> @User.Identity.Name
                </button>
                <ul class="dropdown-menu dropdown-menu-end">
                    <li>
                        <span class="dropdown-item-text small text-muted">
                            @User.FindFirst("KullaniciAdi")?.Value
                        </span>
                    </li>
                    <li><hr class="dropdown-divider"></li>
                    <li>
                        @* Çıkış POST olmalı — sebebi controller'da yazılı *@
                        <form asp-controller="Hesap" asp-action="Cikis" method="post">
                            <button type="submit" class="dropdown-item text-danger">
                                <i class="bi bi-box-arrow-right"></i> Çıkış yap
                            </button>
                        </form>
                    </li>
                </ul>
            </div>
        }

    </div>
</div>
```

---

## ▶️ Test senaryosu

| Test | Beklenen |
|---|---|
| `/` adresine git | Giriş sayfasına yönlendirir |
| Yanlış şifre | "Kullanıcı adı veya şifre hatalı" |
| Doğru giriş | Panele gider, sağ üstte ad görünür |
| `/Ogrenci` yaz (giriş yapmadan) | Giriş sayfası + `?donusUrl=/Ogrenci` |
| Giriş yap | Doğrudan `/Ogrenci`'ye gider |
| Çıkış yap | Giriş sayfası, "oturum kapatıldı" mesajı |
| Şifre alanına `' OR '1'='1` | Girilemiyor (EF parametreliyor) |

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Sonsuz yönlendirme döngüsü | `HesapController`'da `[AllowAnonymous]` yok | Ekle |
| Giriş oluyor ama sayfa hâlâ engelli | `UseAuthentication` yanlış sırada | `UseAuthorization`'dan **önce** |
| Şifre doğru ama girmiyor | Seed'deki hash farklı | `SifreYardimcisi.Hashle("ubys123")` çıktısıyla karşılaştır |
| `@User.Identity.Name` boş | Claim eklenmemiş | `ClaimTypes.Name` claim'i var mı? |
| Giriş sonrası hep panele gidiyor | `ReturnUrlParameter` ile parametre adı farklı | İkisi de `donusUrl` |
| Kullanıcı menüsü açılmıyor | Bootstrap JS yok | Layout'ta `bootstrap.bundle.min.js` |
| Kullanıcı bulunamıyor | Query filter `AktifMi` engelliyor | Kullanıcı aktif mi kontrol et |

---

## ✏️ Öğrenci alıştırması

1. Giriş sistemini kur, tüm test senaryolarını geçir.
2. `SifreYardimcisi.Hashle("ubys123")` çıktısını ekrana yazdır, veritabanındakiyle karşılaştır. Sonra `"ubys124"` ile dene — çıktı ne kadar değişti?
3. Şifre değiştirme ekranı yaz: mevcut şifre + yeni şifre + yeni şifre tekrar.
   İpucu: `[Compare("YeniSifre")]` özniteliği iki alanın eşitliğini kontrol eder.
4. Kayıtları kimin oluşturduğunu sakla: `Fakulte` modeline `OlusturanKullaniciId` ekle.
   ```csharp
   var kullaniciId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
   ```
5. **Düşünme soruları:**
   - ADO.NET sürümünde giriş sorgusunda parametre kullanmak **zorundaydık.** EF'te neden endişelenmiyoruz?
   - Şifreleri düz metin saklasaydık ve veritabanı sızsaydı, sadece bu site mi etkilenirdi?
   - Çerez çalınırsa ne olur? Nasıl korunulur?

---

👉 Sonraki: [`12-ef-ipuclari-ve-tuzaklar.md`](12-ef-ipuclari-ve-tuzaklar.md)
