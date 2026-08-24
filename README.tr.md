# Codex ve Claude Code için Antigravity Delegation

[English](README.md) | **Türkçe**

Google Antigravity CLI'yi **OpenAI Codex** veya **Claude Code** içerisinden harici bir kodlama/review ajanı olarak kullanır. Nihai doğrulama sorumluluğu ana kodlama ajanında kalır.

Paket iki platformda da aynı temel akışı uygular:

```text
Kullanıcı
  │
  ▼
Codex / Claude Code ana ajanı
  │
  ▼
Antigravity delegation skill
  │
  ▼
Platforma özel Antigravity sub-agent
  │
  ▼
agy --output-format json
  │
  ▼
Antigravity üzerinde çalışan model
  │
  ▼
Kısa ve yapılandırılmış bulgular
  │
  ▼
Ana ajan repo durumunu, diff'i, testleri ve kanıtları doğrular
```

Antigravity bilinçli olarak **bağımsız bir dış görüş** şeklinde değerlendirilir; doğruluğun kaynağı olarak kabul edilmez.

## Neden iki ayrı SKILL.md var?

Codex ve Claude Code yeniden kullanılabilir skill yapılarını destekler. Claude Code ayrıca Agent Skills açık standardını kullanır. Buna rağmen iki host'un çalıştırma mekanizması tamamen aynı değildir.

Claude Code sürümü şu host-specific frontmatter alanlarını kullanır:

```yaml
context: fork
agent: antigravity
```

Böylece skill izole edilmiş özel Claude Code `antigravity` sub-agent'ı içinde çalışır.

Codex sürümü ise ana Codex ajanına bounded work order'ı özel `antigravity` Codex sub-agent'ına devretmesini söyler.

Workflow ve güvenlik politikası aynı tutulmuştur. Ancak host'a özel alanların diğer platform tarafından sessizce görmezden gelineceğini varsaymamak için wrapper dosyaları ayrıdır.

## Dosya yapısı

```text
.
├── README.md
├── README.tr.md
├── .agents/
│   └── skills/
│       └── antigravity-delegation/
│           └── SKILL.md
├── .codex/
│   └── agents/
│       └── antigravity.toml
└── .claude/
    ├── skills/
    │   └── antigravity-delegation/
    │       └── SKILL.md
    └── agents/
        └── antigravity.md
```

## Gereksinimler

Şunlara ihtiyacın var:

- Google Antigravity CLI kurulu ve authenticate edilmiş olmalı;
- `agy` komutu `PATH` üzerinde bulunmalı;
- Codex, Claude Code veya ikisi birden;
- implementation/diff doğrulaması için Git.

Antigravity erişimini kontrol et:

```bash
agy models
```

Entegrasyon Antigravity headless mode kullanır ve `status`, `response`, `conversation_id` gibi alanları içeren JSON çıktısı bekler.

Claude Code tarafında güncel bir sürüm kullan. Paketteki skill, bir skill'i izole sub-agent context'inde çalıştırmak için dokümante edilmiş mekanizma olan `context: fork` ile `agent: antigravity` kullanır.

### Workspace ve headless izinleri

Her adapter çağrısı host'un mevcut proje dizinini Antigravity'ye aktarır:

```bash
agy -p "<PROMPT>" --add-dir "$PWD" ...
```

Böylece önceki bir oturumdan kalan Antigravity projesi, mevcut repository'nin aktif workspace dışında görünmesine neden olmaz. Eklenen workspace içindeki dosya erişimleri Antigravity'nin workspace politikasını izler.

Shell komutları headless modda yine varsayılan olarak `Ask` kullanır. Delegated review Git veya proje kontrolleri çalıştıracaksa yalnızca gerekli komutları `~/.gemini/antigravity-cli/settings.json` dosyasına ekle:

```json
{
  "permissions": {
    "allow": [
      "command(git (status|diff|rev-parse))",
      "command(npm run (build|lint|test))"
    ]
  }
}
```

Test komutu kuralını projenin kullandığı komutlarla değiştir. Eşleşen bir `deny` veya `ask` kuralı `allow` kuralından önceliklidir; daha geniş çakışan kurallar bırakma.

Kaynaklar: [Antigravity headless mode](https://antigravity.google/docs/cli/headless/#permissions-in-headless-mode) ve [fine-grained permissions](https://antigravity.google/docs/cli/permissions/).

## Kurulum

### Seçenek A — Proje bazlı kurulum

Bu paketteki gizli klasörleri projenin root dizinine kopyala.

**Codex** için:

```text
.agents/skills/antigravity-delegation/SKILL.md
.codex/agents/antigravity.toml
```

**Claude Code** için:

```text
.claude/skills/antigravity-delegation/SKILL.md
.claude/agents/antigravity.md
```

Aynı repo üzerinde iki aracı da kullanıyorsan dört dosyanın tamamını tut.

Custom agent tanımlarını ekledikten veya değiştirdikten sonra yeni bir Codex/Claude Code session başlat.

### Seçenek B — Global kurulum

#### Codex

```bash
mkdir -p ~/.agents/skills/antigravity-delegation
mkdir -p ~/.codex/agents

cp .agents/skills/antigravity-delegation/SKILL.md \
  ~/.agents/skills/antigravity-delegation/SKILL.md

cp .codex/agents/antigravity.toml \
  ~/.codex/agents/antigravity.toml
```

#### Claude Code

```bash
mkdir -p ~/.claude/skills/antigravity-delegation
mkdir -p ~/.claude/agents

cp .claude/skills/antigravity-delegation/SKILL.md \
  ~/.claude/skills/antigravity-delegation/SKILL.md

cp .claude/agents/antigravity.md \
  ~/.claude/agents/antigravity.md
```

Custom agent kurulumundan sonra kodlama ajanı session'ını yeniden başlat.

## Delegation modları

Entegrasyon üç mod tanımlar.

### `consult`

Salt okunur bağımsız reasoning.

Şunlar için kullan:

- mimari;
- debugging hipotezleri;
- tasarım trade-off'ları;
- planlama;
- alternatif yaklaşımlar;
- ikinci görüş.

### `review`

Salt okunur dış review.

Şunlar için kullan:

- code review;
- diff review;
- security review;
- regression riski analizi;
- test coverage incelemesi;
- implementation eleştirisi.

### `implement`

Antigravity'nin bounded bir aday implementation üretmesine izin verir.

Sadece gerçekten dış bir implementation faydalı olacaksa kullan.

Ardından host ajan mutlaka şunları incelemelidir:

```bash
git status --short
git diff
```

ve değişiklikleri kabul etmeden önce ilgili testleri çalıştırmalıdır.

## Kullanım

### Codex

Codex'e doğal dille söyleyebilirsin:

```text
Use the antigravity-delegation skill in review mode.
Ask Antigravity to independently review the authentication changes for
correctness, race conditions, and missing tests. Then verify its findings yourself.
```

Daha yapılandırılmış bir work order:

```text
Use antigravity-delegation.

mode: review
objective: Review the new token refresh implementation for concurrency bugs.
scope: src/auth/, tests/auth/
constraints: Do not modify files.
expected_output: Findings ranked by severity with concrete file references.
effort: high
```

Beklenen akış:

```text
Codex main
  -> antigravity-delegation skill
  -> Codex antigravity sub-agent
  -> agy
  -> yapılandırılmış rapor
  -> Codex doğrulaması
```

### Claude Code

Skill'i doğrudan çağır:

```text
/antigravity-delegation mode: review
objective: Review the new token refresh implementation for concurrency bugs.
scope: src/auth/, tests/auth/
constraints: Do not modify files.
expected_output: Findings ranked by severity with concrete file references.
effort: high
```

Claude Code skill'inde `context: fork` ve `agent: antigravity` bulunduğu için work order izole edilmiş özel Antigravity adapter sub-agent'ında çalışır ve kısa sonuç ana konuşmaya döner.

Claude Code'un gerektiğinde skill'i otomatik kullanmasını da isteyebilirsin; skill açıklaması model invocation'a açıktır.

## Antigravity modeli seçmek

Adapter'lar varsayılan olarak `gemini-3.7-flash-high` kullanır. Böylece asıl implementation işi hızlı dış worker'da kalırken host'un büyük modeli planlama ve doğrulamadan sorumlu olur.

Kullanılabilir model slug'larını listele:

```bash
agy models
```

Yalnızca varsayılanı değiştirmek istediğinde work order'a `antigravity_model` ekle:

```text
antigravity_model: gemini-3.7-flash-medium
```

Adapter şu çağrıyı üretir:

```bash
agy -p "<PROMPT>" \
  --add-dir "$PWD" \
  --model "gemini-3.7-flash-medium" \
  --output-format json \
  --effort high \
  --print-timeout 10m
```

Bilinmeyen bir model pinlenirse Antigravity headless mode sessizce başka modele geçmek yerine hata verir.

## Sonuç sözleşmesi

Platforma özel adapter kısa bir rapor döndürür:

```text
Status: SUCCESS | ERROR | BLOCKED
Mode: consult | review | implement
Antigravity model: ...
Conversation ID: ...

Summary:
...

Findings:
- ...

Files changed:
- ...

Verification performed:
- ...

Risks / unresolved questions:
- ...
```

Açıkça istenmediği sürece dış modelin ham ve uzun cevabı ana context'e taşınmaz.

## Doğrulama modeli

Nihai cevaptan host kodlama ajanı sorumludur.

Başarılı bir Antigravity sonucu exit code `0`, JSON status `SUCCESS` ve boş olmayan bir `response` gerektirir. Bu yalnızca dış çalıştırmanın tamamlandığı anlamına gelir; sonucun doğru olduğunu kanıtlamaz.

Önemli iddialar şu kaynaklarla doğrulanmalıdır:

- repository kodu;
- testler;
- build/lint/type-check sonuçları;
- loglar;
- spesifikasyonlar;
- otoritatif dokümantasyon.

Implementation görevlerinde herhangi bir değişiklik kabul edilmeden önce diff incelenmelidir.

## İzinler ve güvenlik

Paket varsayılan olarak şunu etkinleştirmez:

```bash
--dangerously-skip-permissions
```

Antigravity headless permission policy bir işlemi engelliyorsa tüm kontrolleri bypass etmek yerine gerekli en dar Antigravity izni yapılandırılmalıdır.

Adapter'lar ayrıca parent görev açıkça istemediği sürece Antigravity'nin şunları yapmasını yasaklar:

- commit;
- push;
- merge;
- deploy;
- publish;
- branch silme;
- remote state değiştirme.

`consult` ve `review` modlarında dosya değiştirmek talimat seviyesinde yasaktır.

## Maliyet ve context stratejisi

Ek sub-agent katmanı bilinçli bir tasarımdır.

İzolasyon olmadan:

```text
ana ajan -> agy -> uzun dış model cevabı -> ana context
```

Bu paketle:

```text
ana ajan
   -> küçük adapter sub-agent
      -> agy
      -> potansiyel olarak uzun dış reasoning
      -> sıkıştırılmış kanıt raporu
   -> ana context
```

Böylece ana context gereksiz dış model çıktılarıyla doldurulmaz ve ana kodlama ajanı context'ini repo ve nihai doğrulama için kullanabilir.

Adapter modelleri bilinçli olarak hafiftir:

- Codex adapter: `gpt-5.6-luna`, low reasoning;
- Claude Code adapter: `haiku`, low effort.

Asıl delegated reasoning'i varsayılan olarak `gemini-3.7-flash-high` kullanan dış worker yapar.

### Codex model override uyumluluğu

Codex runtime özel Luna override'ını reddederse şu dosyayı düzenle:

```text
.codex/agents/antigravity.toml
```

ve şu satırları kaldır:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "low"
```

Sub-agent parent Codex konfigürasyonunu inherit eder.

## Sorun giderme

### `agy: command not found`

Antigravity CLI'nin kurulu olduğunu ve Codex/Claude Code'un başladığı environment içinde göründüğünü kontrol et:

```bash
which agy
agy models
```

### Antigravity `ERROR` döndürüyor

Şunları incele:

- process exit code;
- stderr;
- JSON `error`;
- istenen model slug'ı;
- Antigravity permission policy.

Permission sorunlarını otomatik olarak `--dangerously-skip-permissions` açarak çözme.

### Claude Code sub-agent'ı görmüyor

Şunu kontrol et:

```text
.claude/agents/antigravity.md
```

veya:

```text
~/.claude/agents/antigravity.md
```

Ardından Claude Code session'ını yeniden başlat ve `/agents` üzerinden kontrol et.

### Claude Code skill'i görmüyor

Şunu kontrol et:

```text
.claude/skills/antigravity-delegation/SKILL.md
```

veya:

```text
~/.claude/skills/antigravity-delegation/SKILL.md
```

Ardından `/skills` kullan veya `/antigravity-delegation` çağır.

### Codex `antigravity` sub-agent'ını spawn edemiyor

Şunun mevcut olduğunu doğrula:

```text
.codex/agents/antigravity.toml
```

veya:

```text
~/.codex/agents/antigravity.toml
```

Custom agent'ı ekledikten sonra yeni Codex session başlat.

Hata özellikle adapter model override'ıyla ilgiliyse yukarıda anlatıldığı gibi model satırlarını kaldır.

## Tasarım ilkeleri

Paket beş kurala dayanır:

1. **Dış modeller reviewer/worker'dır; otorite değildir.**
2. **Nihai karar ana kodlama ajanına aittir.**
3. **Consult ve review varsayılan olarak read-only'dir.**
4. **Implementation bounded'dır ve diff ile doğrulanır.**
5. **Uzun dış reasoning mümkün olduğunca ana context'in dışında tutulur.**
