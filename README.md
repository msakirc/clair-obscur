# Clair Obscur — Local image-server lifecycle

> **Standalone.** Split from the [KutAI](https://github.com/msakirc/kutai) monorepo (`packages/clair_obscur/`), where it is developed. It does not need the KutAI app; its test suite passes with only its declared dependencies installed.

[English](#english) · [Türkçe](#türkçe)

<a id="english"></a>

## Purpose

Clair Obscur is the image-side counterpart of DaLLaMa: it owns the **process** of a
local image server (ComfyUI by default, or AUTOMATIC1111) on the shared GPU. It
launches it on demand, waits for it to become healthy, records a PID lock, reports
its resident VRAM to Nerd Herd, kills its own orphan left by a crash (only the PID
it wrote into its lock, and only if the command line matches its backend), and
releases the GPU after an idle window when General Beckman switches lanes back to
text work.

**It does NOT** generate images (`paintress` does), pick a model or checkpoint (the
checkpoint name is read by `paintress` from `CLAIR_OBSCUR_MODEL`), or touch
`llama-server` — orphan reconciliation only ever matches its own backend process.

## Install

```bash
pip install "clair_obscur @ git+https://github.com/msakirc/clair-obscur.git"
```

Requires Python 3.10+. Sibling KutAI packages are installed from their own repositories.

## Public API

```python
import clair_obscur

if clair_obscur.available():          # False when no backend exe is configured
    url = await clair_obscur.start()  # idempotent; -> base URL once healthy
clair_obscur.status()                 # -> {resident, pid, backend, base_url, release_hint_at}
clair_obscur.base_url()               # -> http://127.0.0.1:8188 (ComfyUI) / :7860 (A1111) by default
clair_obscur.record_release_hint()    # lane switch: stop after the idle window
await clair_obscur.stop()             # forced / emergency stop only
```

## Dependencies

- `dallama` — reuses its `PlatformHelper` (Windows Job Object so the child dies with
  KutAI, no console window, stderr capture). `nerd_herd` — resident-VRAM push.
  `httpx`, `psutil`, `yazbunu`.
- Env (all optional; missing exe → `available()` is `False` and Fatih Hoca filters
  the local image model out):

| variable | default | meaning |
|---|---|---|
| `CLAIR_OBSCUR_BACKEND` | `comfyui` | `comfyui` or `a1111` |
| `CLAIR_OBSCUR_EXE` | — | launcher executable / script |
| `CLAIR_OBSCUR_DIR` | — | working directory for the launcher |
| `CLAIR_OBSCUR_HOST` / `CLAIR_OBSCUR_PORT` | `127.0.0.1` / `8188` (ComfyUI), `7860` (A1111) | where to probe |
| `CLAIR_OBSCUR_URL` | `http://HOST:PORT` | full base URL override |
| `CLAIR_OBSCUR_IDLE_RELEASE_SECONDS` | `60` | idle time after a release hint before stop |

## Gotchas

- `start()` is serialised by a lock: a concurrent caller waits, then sees the server
  resident and gets the URL — never a second launch.
- Normal lane switches must go through `record_release_hint()`; the idle backstop
  performs the actual stop. A direct `stop()` while an image task is queued forces a
  cold start.
- If the child exits during the health wait, `start()` aborts immediately with the
  stderr tail in the log instead of waiting for the full timeout.

## Tests

```powershell
python -m pytest packages/clair_obscur/tests
```

## In KutAI

Fatih Hoca drops the local image model when `available()` is false; [paintress](https://github.com/msakirc/paintress) gets the server URL from it, and Hüsam's image lane starts it on demand. Source: [`packages/clair_obscur/`](https://github.com/msakirc/kutai/tree/main/packages/clair_obscur) in the [KutAI monorepo](https://github.com/msakirc/kutai), where this package is developed.

## Status and limitations

- Standalone: it does not import the KutAI app, and its tests pass with only its declared dependencies.
- Version 0.1.0, part of a personal research project. Developed and tested on Windows 11 with Python 3.10; other platforms are untested.
- Needs a ComfyUI or AUTOMATIC1111 install that you point at with `CLAIR_OBSCUR_EXE`.

## License

MIT. See [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE).

---
<a id="türkçe"></a>
## Türkçe

### Amaç

Clair Obscur, DaLLaMa'nın görüntü tarafındaki karşılığıdır: paylaşılan GPU üzerindeki
yerel görüntü sunucusunun (varsayılan ComfyUI ya da AUTOMATIC1111) **sürecinin**
sahibidir. Gerektiğinde başlatır, sağlıklı olmasını bekler, PID kilidi tutar, kullandığı
VRAM'i Nerd Herd'e bildirir, çökmeden kalan kendi yetim sürecini öldürür (yalnızca
kilidine yazdığı PID'i ve komut satırı arka ucuyla eşleşiyorsa) ve General Beckman metin işine döndüğünde boşta bekleme süresinden sonra
GPU'yu serbest bırakır.

Görüntü **üretmez** (`paintress` üretir), model ya da checkpoint seçmez ve
`llama-server`'a asla dokunmaz — yetim süreç uzlaştırması yalnızca kendi arka uç
sürecini eşler.

### Kurulum

```bash
pip install "clair_obscur @ git+https://github.com/msakirc/clair-obscur.git"
```

Python 3.10+ gerekir. Kardeş KutAI paketleri kendi depolarından kurulur.

### Genel API

```python
import clair_obscur

if clair_obscur.available():          # arka uç exe ayarlı değilse False
    url = await clair_obscur.start()  # eş güçlü; sağlıklı olunca temel URL döner
clair_obscur.record_release_hint()    # şerit değişimi: boşta süreden sonra durdur
await clair_obscur.stop()             # yalnızca zorunlu / acil durdurma
```

### Dikkat Edilecekler

- Normal şerit değişimleri `record_release_hint()` üzerinden yapılmalı; asıl durdurmayı
  boşta bekleme denetimi yapar.
- Ayarlar ortam değişkenlerinden okunur (`CLAIR_OBSCUR_*`, tablo yukarıda).

### Testler

```powershell
python -m pytest packages/clair_obscur/tests
```

### KutAI içindeki yeri

`available()` yanlışsa Fatih Hoca yerel görüntü modelini eler; [paintress](https://github.com/msakirc/paintress) sunucu adresini buradan alır, Hüsam'ın görüntü hattı onu gerektiğinde başlatır. Kaynak: [KutAI monorepo](https://github.com/msakirc/kutai) içinde [`packages/clair_obscur/`](https://github.com/msakirc/kutai/tree/main/packages/clair_obscur); paket orada geliştirilir.

### Durum ve sınırlamalar

- Bağımsız: KutAI uygulamasını import etmez; testleri yalnızca bildirilen bağımlılıklarla geçer.
- Sürüm 0.1.0, kişisel bir araştırma projesinin parçası. Windows 11 ve Python 3.10 üzerinde geliştirilip test edildi; diğer platformlar denenmedi.
- `CLAIR_OBSCUR_EXE` ile göstereceğiniz bir ComfyUI ya da AUTOMATIC1111 kurulumu gerekir.

### Lisans

MIT. [LICENSE](https://github.com/msakirc/kutai/blob/main/LICENSE) dosyasına bakın.
