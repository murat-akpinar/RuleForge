# Proje Haritası

Yer imleri. Bir dosya eklendiğinde, taşındığında, silindiğinde veya yeni feature marker açıldığında aynı commit içinde güncellenir. Kısa tutulur.

## Dizinler
- `nginx/nginx.conf` → yönlendirme kuralları, güvenlik header'ları
- `<backend/src/...>` → <ne işe yarar>

## Feature indeksi
Koddaki `--- START FEATURE: <ad> ---` markerlarının karşılığı. Aramak için:
`grep -rn "FEATURE: <ad>" --exclude-dir=.git --exclude-dir=tmp .`

| Feature | Nerede |
|---|---|
| <user-login> | <backend/src/auth/> |

## Ortak yardımcılar
Yeni bir şey yazmadan önce buraya bak. Aynı işi yapan varsa tekrar yazma.

| Ne yapar | Nerede |
|---|---|
| <istek doğrulama şeması> | <backend/src/common/validation> |
