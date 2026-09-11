---
name: kurulum
description: Boş bir projede ilk kurulumu yapar - keşif, altyapı kararları, iskelet ve todo listesi. docs/PROJECT.md doldurulmamışken çalıştırılır. Uygulama kodu yazılmaz.
---

# Kurulum

Bu skill projeyi **kodlamaya hazır** hale getirir. Uygulama kodu yazmaz, hello-world bile yazmaz. Kod, `docs/todo.md` kutucuklarıyla sonra gelir.

Sıra: keşif → kararlar → iskelet → todo → ilk commit. Her fazın sonunda kullanıcı onayı alınır, onaysız sonraki faza geçilmez.

## 0. Ön kontrol
- Dizinde şablon dışında dosya varsa dur ve sor: burası boş bir proje mi?
- `git config user.name` ve `git config user.email` tanımlı mı bak. Tanımlı değilse dur ve sor, kimlik uydurma.
- `git rev-parse --git-dir` başarısızsa `git init`.

## 1. Keşif
Kullanıcıya tek tek sor, cevapları biriktir:
- Ne geliştiriyoruz, kim kullanacak, temel akış ne?
- v1 kapsamında ne var? Daha da önemlisi: v1'de bilerek **ne yok**?
- Elde kaynak materyal var mı? Varsa `tmp/` içine koymasını iste, `tmp/README.md`'yi kendisi doldursun. `tmp/` salt okunur, oradaki dosyaları okursun ama değiştirmezsin.

Cevapları `docs/PROJECT.md` dosyasına yaz ve onaylat.

## 2. Kararlar
Kararları tek tek öner, gerekçesiyle sun, onay al. Her onaylanan karar `docs/decisions/NNN-konu.md` dosyasına yazılır (şablon: `docs/decisions/0000-sablon.md`).

Sıra:
1. **Bileşenler.** nginx her zaman vardır, tek giriş kapısıdır. `backend`, `frontend`, `db`, `worker`, `cache` yalnızca bu proje gerektiriyorsa eklenir. Gerekçesi olmayan bileşen eklenmez.
2. **Stack.** Her bileşen için dil, framework, sürüm. Kullanıcının bildiği ve projeye uyan en sade seçenek tercih edilir.
3. **Mimari kalıp.** Katmanlar, modül sınırları, API stili, hata biçimi.
4. **Veri.** Veritabanı, şema yaklaşımı, migration aracı. Bileşenler arasında db yoksa bu karar atlanır.
5. **Dev döngüsü.** `compose.override.yaml` içinde hangi dizin bind-mount edilecek, hot reload nasıl çalışacak, test ve lint komutları ne.

Test, format ve lint komutlarını `CLAUDE.md` içindeki **Komutlar** bölümüne yaz; oradaki `<kurulumda doldurulur>` yer tutucuları kalmasın.

## 3. İskelet
Kararlara göre üret, uygulama kodu yazma:
- Her bileşen için dizin ve Dockerfile (multi-stage, sabit imaj sürümü, root olmayan kullanıcı).
- `nginx/Dockerfile` ve `nginx/nginx.conf`: `nginxinc/nginx-unprivileged`, container içinde 8080.
- `compose.yaml` ve `compose.override.yaml`: host'a yalnızca nginx portu, healthcheck, `depends_on: condition: service_healthy`.
- `.env.example`: gereken tüm değişken isimleri. `.env` dosyasını sen oluşturma, kullanıcı doldursun.
- Ayrıntılar `.claude/rules/docker.md` dosyasındadır, ona uy.

Doğrula: `docker compose config --quiet` hatasız dönmeli. Çıktı basan biçimi kullanma, env değerlerini gösterir.

## 4. todo.md
Fazlara böl, her kutucuk tek oturumda ve tek commit'te bitecek büyüklükte olsun. Her kutucuğun ölçülebilir bir kabul kriteri olur.
- Faz 1'in ilk kutucuğu her zaman: tüm servisler nginx arkasında ayağa kalkar ve `/api/health` 200 döner.
- **Her fazın son kutucuğu güvenlik ve test kapanışıdır.** `docs/todo.md` içindeki kapanış şablonunu kopyala, komutları bu projenin stack'ine göre doldur. Bu kutucuk atlanamaz, sonraki faza kapanış geçmeden geçilmez.

Taslağı kullanıcıya göster, düzeltmelerini al, sonra yaz.

## 5. İlk commit
- `docs/MAP.md`'yi oluşturulan dizinlere göre doldur.
- Remote'u sor (adres ve depo gizli mi). Verilmezse push atlanır, kullanıcıya söylenir.
- Commit:
  ```
  git add -A
  git cliff --with-commit "chore: proje iskeleti" -o CHANGELOG.md
  git add CHANGELOG.md
  git commit -m "chore: proje iskeleti"
  ```
- Kimlik kontrolü: `git log -1 --format='%an <%ae>%n%B'` çıktısında kullanıcının kimliği görünmeli, `grep -i claude` boş dönmeli. Claude izi varsa commit'i `git commit --amend` ile düzelt ve kullanıcıyı uyar.
- Remote varsa `git push -u origin <dal>`.

Sonra dur ve şunu söyle: kararlar ve iskelet hazır, `/clear` atıp ilk kutucukla başlayabilirsin.
