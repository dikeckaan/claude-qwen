# claude-qwen

[Claude Code](https://code.claude.com)'u Apple Silicon Mac'te **yerel bir GGUF modeliyle** (varsayılan olarak Qwen3.8-27B) çalıştıran tek dosyalık bir başlatıcı. Bulut API'si ya da internet bağlantısı gerekmez; model kendi makinende [llama.cpp](https://github.com/ggml-org/llama.cpp) ile çalışır.

*English: a single-file launcher that runs Claude Code against a local GGUF model (default: Qwen3.8-27B) served by llama.cpp's Anthropic-compatible `/v1/messages` endpoint on Apple Silicon. Includes a model picker, a Hugging Face GGUF downloader, auto start/stop of the server, and a patched Qwen3.8 chat template that accepts Claude Code's mid-conversation system messages.*

## Özellikler

- `claude-qwen` yazınca bir model seçersin, sunucu gerekirse başlar ve Claude Code açılır.
- **Model seçim menüsü:** `models/` klasöründeki tüm GGUF dosyalarını otomatik bulur.
- **Model indirici:** `claude-qwen download <hf-repo>` repodaki GGUF dosyalarını boyutlarıyla ve RAM'e sığıp sığmadığıyla listeler. İndirme yarıda kalırsa kaldığı yerden devam eder.
- **Otomatik kapanma:** son oturum kapanınca sunucu da kapanır ve RAM boşalır.
- **Şablon düzeltmesi:** Qwen3.8'in orijinal sohbet şablonu, Claude Code'un konuşma ortasında gönderdiği system mesajlarında `System message must be at the beginning` hatası verir. `chat_template.jinja` bu sorunu düzeltilmiş halidir.
- **İsteğe bağlı GPU bellek ayarı:** sunucu açıkken macOS'un GPU bellek sınırını (`iogpu.wired_limit_mb`) yükseltir, kapanınca varsayılana döndürür. Sınır yeterliyse MTP ile spekülatif çözmeyi otomatik açar (~%60 daha hızlı).

## Kurulum

### Homebrew

```bash
brew install dikeckaan/claude-qwen/claude-qwen   # llama.cpp'yi de kurar
claude-qwen download unsloth/Qwen3.8-27B-GGUF    # listeden bir quant seç
claude-qwen
```

Modeller ve loglar `~/.claude-qwen` klasörüne yazılır. Başka bir klasör için `CLAUDE_QWEN_HOME` değişkenini kullan.

### Git ile

```bash
git clone https://github.com/dikeckaan/claude-qwen.git ~/claude-qwen
~/claude-qwen/claude-qwen setup              # llama.cpp (brew) kurar, komutu ~/.local/bin'e bağlar
claude-qwen download unsloth/Qwen3.8-27B-GGUF  # listeden bir quant seç
claude-qwen
```

Gereksinimler: Apple Silicon Mac, [Homebrew](https://brew.sh), [Claude Code](https://code.claude.com), `~/.local/bin` dizininin PATH'te olması.

### Hangi quant?

27B'lik yoğun (dense) bir model için yaklaşık rakamlar:

| RAM | Önerilen | Dosya |
|---|---|---|
| 24 GB | IQ4_XS / IQ3_M | 13–14 GB |
| 32 GB | Q4_K_XL / Q5_K_M | 17–20 GB |
| 48 GB+ | Q6_K / Q8_0 | 22–30 GB |

Bağlam (KV önbelleği) ve sistem için ~6 GB pay bırak.

## Kullanım

```bash
claude-qwen                     # model seç ve Claude Code'u aç
claude-qwen -p "soru"           # diğer argümanlar claude'a aynen geçer
claude-qwen download [repo]     # GGUF indir
claude-qwen models              # indirilmiş modeller
claude-qwen status | stop | log
```

| Değişken | Varsayılan | Açıklama |
|---|---|---|
| `QWEN_MODEL` | (menü) | Menüyü atla: numara ya da dosya adı |
| `QWEN_CTX` | `102400` | Bağlam boyutu. Claude Code otomatik özetleme için en az 100K ister |
| `QWEN_EFFORT` | `medium` | Düşünme seviyesi: `xhigh`, `medium`, `low`, `none` |
| `QWEN_MTP` | `auto` | Modelin kendi MTP katmanıyla spekülatif çözme. `auto`: GPU sınırı yeterliyse açılır. `0`/`1`: zorla kapat/aç |
| `QWEN_KEEP` | `0` | `1`: oturum kapanınca sunucu açık kalır |
| `QWEN_GPU_MB` | RAM × 7/8 | Sunucu açıkken kullanılacak GPU bellek sınırı |
| `QWEN_EXTRA` | | Ek `llama-server` argümanları |

### GPU bellek sınırı (isteğe bağlı)

macOS, GPU'ya varsayılan olarak RAM'in ~%75'ini verir. 24 GB'lık bir Mac'te bu sınır 27B model + 100K bağlam + MTP için yetmez. `claude-qwen setup` komutu (geçici dosyayı doğrudan root olarak `/etc/sudoers.d` içine yazar, `visudo` ile doğrular), yalnızca iki komuta şifresiz sudo izni veren dar bir kural yazdırır: `sysctl iogpu.wired_limit_mb=<N>` ve `sysctl iogpu.wired_limit_mb=0`. Bu kuralı kurarsan betik, sunucu açıkken sınırı yükseltir ve kapanınca varsayılana döndürür. Kurmazsan her şey normal çalışır, yalnızca MTP kapalı kalır.

## Performans (Apple M6, 24 GB, 100K bağlam)

Yazma hızı (saniyede token):

| Model | MTP kapalı | MTP açık |
|---|---|---|
| Qwen3.8-27B UD-IQ4_XS (14.3 GB) | 9.2 | 14.9 (Claude Code içinde ~16) |
| Qwen3.8-27B Uncensored IQ3_M (12.8 GB) | 10.1 | 15.5–18.5 (Claude Code içinde ~16) |

- MTP, modelin içinde gömülü olan çoklu token tahmin katmanını kullanır. Ayrı bir dosya gerekmez ve kaliteyi düşürmez. `--spec-draft-n-max 3` ayarı, 2'ye göre biraz daha hızlı çıktı.
- İlk okuma (prefill): saniyede ~210–300 token. Claude Code'un ilk isteği ~14K token olduğu için oturumun ilk cevabı ~1 dakika sürer; sonraki istekler önbellekten gelir.
- Daha da hızlandırmak için: `QWEN_EFFORT=low` ve claude-qwen oturumlarında kullanmadığın MCP sunucularını ya da eklentileri kapatmak.

## Uyarılar

- Yerel model, bulut Claude kadar yetenekli değil. Özellikle `--dangerously-skip-permissions` ile çalıştırırken dikkatli ol.
- Açılışta çıkan "unrecognized model" ve "connectors disabled" uyarıları beklenen davranış.

## Lisans

MIT. `chat_template.jinja`, Qwen3.8 sohbet şablonundan türetilmiştir (Apache 2.0).
