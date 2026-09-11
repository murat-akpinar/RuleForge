# RuleForge

Bu depo tek bir şey barındırır: `template/`, boş bir dizine kopyalanıp Claude Code ile projeye başlamak için kullanılan şablon. Buradaki kurallar **şablonu geliştirirken** geçerlidir; şablonun kendi kuralları `template/CLAUDE.md` içindedir.

## Şablonu değiştirirken
- `template/CLAUDE.md` 200 satırın altında kalır. Yalnızca belirli dosyalarda geçerli olan kural oraya değil, `paths:` frontmatter'ı olan bir `.claude/rules/*.md` dosyasına yazılır.
- Bir kural hem `template/CLAUDE.md` hem bir rules dosyasında yer alıyorsa çelişmediğinden emin ol. Çelişen talimatlarda Claude birini rastgele seçer.
- Yeni kural eklemeden önce sor: bu, modelin varsayılan davranışından gerçekten farklı mı? Değilse ekleme, sadece token harcar.

## Değişiklikten sonra doğrula
```bash
python3 -c "import json,pathlib; json.loads(pathlib.Path('template/.claude/settings.json').read_text())"
python3 -c "import re,yaml,pathlib,glob
for f in glob.glob('template/.claude/**/*.md', recursive=True):
    m = re.match(r'^---\n(.*?)\n---\n', pathlib.Path(f).read_text(), re.S)
    if m: yaml.safe_load(m.group(1))"
```
`cliff.toml` değiştiyse geçici bir depoda birkaç commit atıp `git cliff --with-commit "feat: x" -o CHANGELOG.md` çıktısına bak.

## Dil
Dokümanlar ve commit açıklamaları Türkçe, kod ve tanımlayıcılar İngilizce. Commit başlığı Conventional Commits.
