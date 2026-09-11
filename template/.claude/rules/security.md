---
paths:
  - "**/*.{py,ts,tsx,js,jsx,mjs,go,rs,java,kt,rb,php,cs}"
  - "**/*.{vue,svelte,astro,html}"
  - "**/*.sql"
---

# Güvenlik ve Kod Kalitesi

Kaynak: OWASP Top 10 ve SonarQube "Sonar way" kural seti. Kod yazarken uyulur, faz kapanışında tek tek kontrol edilir.

## Sır yönetimi
- Sır, şifre, token, API anahtarı ve bağlantı adresi koda, teste, log'a veya örnek veriye girmez. Yalnızca env değişkeni.
- Yeni env değişkeni eklenince `.env.example`'a da eklenir (yalnızca isim ve kısa açıklama, değer değil).
- Log'a giden veri maskelenir: şifre, token, kimlik numarası, kart, e-posta.

## Girdi ve çıktı
- Dış girdi sunucu tarafında şemayla doğrulanır: tip, uzunluk, aralık, biçim, izinli değerler. İstemci doğrulaması güvenlik kontrolü değildir.
- SQL yalnızca parametreli sorguyla yazılır. String birleştirme yasak; ORM kullanılıyorsa raw SQL de parametreli olur.
- Kabuk komutu çalıştırılırken shell kullanılmaz, argüman dizisi verilir. Kullanıcı verisi komut satırına gömülmez.
- HTML çıktısı şablon motorunun otomatik kaçışıyla üretilir. `innerHTML` ve `dangerouslySetInnerHTML` kullanılmaz.
- Dosya yükleme: tip magic byte ile doğrulanır, rastgele ad verilir, webroot dışında saklanır, boyut sınırlanır.
- Dış adrese istek kullanıcı girdisinden türetiliyorsa izinli alan adı listesi uygulanır (SSRF).

## Kimlik ve yetki
- Yetki kontrolü servis katmanında yapılır. Arayüzde gizlemek kontrol değildir.
- Her kayıt erişiminde sahiplik kontrolü yapılır: kayıt gerçekten bu kullanıcıya mı ait.
- Şifre argon2id veya bcrypt (cost ≥ 12) ile saklanır. Kendi hash veya şifreleme algoritmanı yazma.
- Token ömrü kısadır ve çıkış sonrası geçersiz kılınabilir olmalıdır.
- Güvenlik amaçlı rastgelelik kriptografik üreteçten alınır (`secrets`, `crypto.randomBytes`), `random`/`Math.random` kullanılmaz.
- Hata yanıtı iç bilgi sızdırmaz: stack trace, SQL hatası, dosya yolu, sürüm bilgisi dışarı çıkmaz.

## SonarQube kuralları
- Cognitive complexity fonksiyon başına ≤ 15. Aşarsa fonksiyon bölünür.
- Fonksiyon ≤ 50 satır, parametre ≤ 5, iç içe blok ≤ 3 seviye.
- Duplikasyon ≤ %3. Aynı blok üçüncü kez yazılacaksa ortak fonksiyona çıkar ve `docs/MAP.md` → Ortak yardımcılar bölümüne yazılır.
- Boş `catch` yok. Hata ya işlenir ya bağlam eklenip yükseltilir; yutulup `null` dönülmez.
- Ölü kod ve yorum satırına alınmış kod commit'lenmez, geçmiş zaten git'te.
- Sihirli sayı ve metin yok, isimlendirilmiş sabit kullanılır.
- `any`, örtük tip ve tip zorlama yok. Public fonksiyonların dönüş tipi açıktır.
- Kaynaklar (dosya, bağlantı, soket) `with` / `defer` / `try-finally` ile kapatılır.
- Kodda `TODO` bırakılmaz; iş `docs/todo.md`'ye kutucuk olarak girer.
- Dallanma, hesaplama veya yetki kontrolü içeren her fonksiyon en az bir testle gelir.

## Faz kapanış kapısı
- Yeni kodda satır kapsamı ≥ %80, duplikasyon ≤ %3, yeni blocker/critical bulgu 0.
- Bağımlılık ve imaj taramasında kritik veya yüksek CVE yok.
- Sonar sunucusu kurulursa `sonar-project.properties` tek bir kutucukta eklenir; o güne kadar bu liste elle kontrol edilir.
