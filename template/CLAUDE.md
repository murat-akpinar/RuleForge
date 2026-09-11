# Proje Kuralları

## Oturum başında
1. Şu sırayla oku: `docs/PROJECT.md`, `docs/todo.md`, `docs/MAP.md`, varsa `tmp/README.md`.
2. `docs/PROJECT.md` doldurulmamışsa kurulum yapılmamıştır: `kurulum` skill'ini çalıştır, kod yazma.
3. Doldurulmuşsa `docs/todo.md` içindeki ilk işaretlenmemiş kutucuktan devam et.

## Dil
- Kod, tanımlayıcılar ve feature marker adları İngilizce.
- `docs/`, commit açıklaması ve durum özetleri Türkçe.
- Commit başlığı Conventional Commits: `feat` `fix` `docs` `chore` `refactor` `test` `perf`.

## Çalışma sırası
Karar → iskelet → todo → kod. Altyapı kararı (veritabanı, framework, kimlik doğrulama, kuyruk, dış servis) gerekiyorsa önce sor; kararı `docs/decisions/NNN-konu.md` dosyasına yaz, sonra uygula. Karar değişirse eski dosya düzenlenmez, "NNN'in yerine geçer" diyen yeni bir karar dosyası yazılır.

## Dizin yapısı
```
CLAUDE.md              CHANGELOG.md (git-cliff üretir, elle düzenlenmez)
cliff.toml             compose.yaml / compose.override.yaml
.env.example           .claude/settings.json, .claude/rules/, .claude/skills/
docs/PROJECT.md        amaç, kapsam, kapsam dışı
docs/MAP.md            proje haritası
docs/todo.md           fazlar ve kutucuklar
docs/decisions/        NNN-konu.md karar kayıtları
nginx/                 Dockerfile + nginx.conf
tmp/                   kaynak materyal; git'e ve imaja girmez, salt okunur
```
Her container kendi dizininde, kendi Dockerfile'ıyla durur (`backend/`, `frontend/`, `worker/`…). Bileşen yalnızca projede gerekiyorsa oluşturulur, peşinen değil.

## Komutlar
- Dev: `docker compose up --build`
- Prod benzeri: `docker compose -f compose.yaml up --build` (override yüklenmez)
- Log: `docker compose logs -f <servis>` · Kabuk: `docker compose exec <servis> sh`
- Test: `<kurulumda doldurulur>`
- Format + lint: `<kurulumda doldurulur>`

## Kod stili
- Fonksiyonel yaz: küçük, saf fonksiyonlar. Yan etkiler (I/O, DB, HTTP) kenarlarda toplanır.
- Tekrar yok. Yeni bir şey yazmadan önce `docs/MAP.md` içindeki **Ortak yardımcılar** bölümüne bak; aynı işi yapan varsa onu kullan.
- Yorum yalnızca kodun **neden** öyle olduğu açık değilse yazılır.
- Feature blokları işaretlenir, ad İngilizce ve kebab-case olur, `docs/MAP.md` ile birebir aynı yazılır:
  ```
  // --- START FEATURE: user-login ---
  // --- END FEATURE: user-login ---
  ```
- Ayrıntılı kurallar: `.claude/rules/security.md` (güvenlik + SonarQube), `.claude/rules/docker.md`.

## Bağımlılıklar
- Yeni paket eklemeden önce sor. Birkaç satır kod aynı işi görüyorsa paket eklenmez.
- Sürümler sabit, lock dosyaları commit'lenir.

## docs/MAP.md
Üç bölüm: **Dizinler**, **Feature indeksi**, **Ortak yardımcılar**. Dosya eklendiğinde, taşındığında, silindiğinde veya yeni feature marker açıldığında aynı commit içinde güncellenir. Kısa tutulur, dosya dosya sayılmaz.

## İş akışı (her kutucuk için)
1. Tek kutucuk al, sadece onu yap. Kapsam dışı bir ihtiyaç görürsen yapma, özete "önerilen" olarak yaz.
2. Doğrula: build çalışıyor, kabul kriteri sağlanıyor, testler geçiyor, format ve lint temiz. Biri bile başarısızsa commit yok.
3. Kutucuğu `[x]` yap, `docs/MAP.md`'yi güncelle. Kontrol:
   `grep -rn "FEATURE:" --exclude-dir=.git --exclude-dir=tmp --exclude-dir=node_modules .`
   çıktısındaki her ad MAP'te geçmeli.
4. Commit ve push:
   ```
   git add -A
   git cliff --with-commit "feat: kısa açıklama" -o CHANGELOG.md
   git add CHANGELOG.md
   git commit -m "feat: kısa açıklama"
   git push
   ```
5. Kimlik: yalnızca benim git kimliğim. `Co-Authored-By`, `Generated with Claude`, `Claude-Session` satırı olmaz. İlk commit'ten sonra bir kez doğrula: `git log -1 --format=%B | grep -i claude` boş çıkmalı.
6. Push başarısız olursa force deneme, hatayı raporla.
7. Özet şu formatta:
   ```
   **Yapılan:** ...
   **Değişen dosyalar:** ...
   **Doğrulama:** <komut> → <sonuç>
   **Önerilen (kapsam dışı):** ...
   **Sıradaki kutucuk:** ...

   ✅ Kutu bitti — commit, push, git-cliff yapıldı. /clear atabilirsin.
   ```

## Faz bitişi
Her fazın son kutucuğu güvenlik ve test kapanışıdır, atlanamaz ve sonraki faza geçilmez. Kapanış geçtikten sonra tag için sor. Onay verirsem:
```
git cliff --bump -o CHANGELOG.md
git add CHANGELOG.md && git commit -m "chore: sürüm $(git cliff --bumped-version)"
git tag "$(git cliff --bumped-version)"
git push --follow-tags   # commit ve tag birlikte gider
```
Sürüm numarası `git cliff --bumped-version` ile hesaplanır, uydurulmaz.

## Takılınca
Aynı sorunu 2 denemede çözemezsen dur. Ne denediğini ve hatayı yaz, bana sor. Aynı yaklaşımı üçüncü kez deneme.

## Sınırlar
- `tmp/` salt okunur. Git'te yedeği yok: silme, taşıma, üzerine yazma yok. Okumak serbest.
- `.env` okunmaz. Değişken isimleri `.env.example`'dadır. `docker compose config` çıktısı env değerlerini çözüp basar, doğrulama için `docker compose config --quiet` kullan.
- Onayım olmadan çalıştırma: `docker compose down -v`, `git clean`, `rm -rf`.
- Hiçbir koşulda: `git push --force`, `git reset --hard`.
