# Amazon Photos 整理スキル

ダウンロードフォルダ内のAmazonPhotosフォルダをSeagateに移動し、日付修正・重複除去・年月フォルダ整理を行う。

## 設定（変更が必要な場合はここを編集）

```
SOURCE_DIR     : ~/Downloads/
SOURCE_PATTERN : AmazonPhotos または AmazonPhotos-N（N は数字）
SEAGATE_BASE   : /Volumes/Seagate Hub
DEST_BASE      : /Volumes/Seagate Hub/整理済み_Amazon
LOG_FILE       : /Volumes/Seagate Hub/amazon_pipeline_log.txt
```

## 処理フロー

以下の Python スクリプトを生成して実行すること。スクリプトは一時ファイル `/tmp/organize_amazon_photos.py` に書き出し、実行後に削除する。

### Step 1 — 事前確認

実行前に以下を確認し、ユーザーに報告する。

1. `~/Downloads/` 内の `AmazonPhotos*.zip` の数と `AmazonPhotos*` フォルダの数・ファイル数
2. Seagate がマウントされているか（`/Volumes/Seagate Hub` が存在するか）
3. `整理済み_Amazon` の既存ファイル数

問題があれば作業を中断してユーザーに伝える。

### Step 1.5 — zip 自動解凍

`~/Downloads/` に `AmazonPhotos*.zip` が存在する場合、以下を実行する。

```bash
cd ~/Downloads
for f in AmazonPhotos*.zip; do
    unzip -q "$f" -d "${f%.zip}"
done
```

解凍完了後、zipファイルのリストを記録しておく（Step 4 で削除するため）。

### Step 2 — Python スクリプト生成・実行

以下のロジックで Python スクリプトを生成して実行する。

```python
#!/usr/bin/env python3
"""
Amazon Photos 整理スクリプト
- ~/Downloads/AmazonPhotos* → /Volumes/Seagate Hub/整理済み_Amazon/YYYY/YYYY-MM/
- ファイルタイムスタンプを撮影日（ファイル名から取得）に修正
- MD5ハッシュで重複検出・削除（既存ファイルとの比較も含む）
- 対応ファイル名形式:
    通常形式    : YYYY-MM-DD_HH-MM-SS_XXX.ext
    編集済み形式: YYYY-MM-DD_HH-MM-SS_XXX (タイムスタンプ).ext
"""

import os, re, hashlib, shutil, subprocess
from datetime import datetime

DOWNLOADS_DIR = os.path.expanduser("~/Downloads")
SEAGATE_BASE  = "/Volumes/Seagate Hub"
DEST_BASE     = os.path.join(SEAGATE_BASE, "整理済み_Amazon")
LOG_FILE      = os.path.join(SEAGATE_BASE, "amazon_pipeline_log.txt")

stats = {"processed": 0, "moved": 0, "duplicates": 0, "errors": 0}

def log(msg):
    ts   = datetime.now().strftime("[%Y-%m-%d %H:%M:%S]")
    line = f"{ts} {msg}"
    print(line)
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(line + "\n")

def md5_file(path):
    h = hashlib.md5()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            h.update(chunk)
    return h.hexdigest()

def set_timestamp(fpath, date_part, time_part):
    m = re.match(r"(\d{4})-(\d{2})-(\d{2})", date_part)
    n = re.match(r"(\d{2})-(\d{2})-(\d{2})", time_part)
    if m and n:
        Y, Mo, D = m.groups()
        H, Mi, S = n.groups()
        subprocess.run(
            ["touch", "-t", f"{Y}{Mo}{D}{H}{Mi}.{S}", fpath],
            check=True, capture_output=True
        )

def get_exif_date(fpath):
    """exiftoolでDateTimeOriginal → CreateDate → MetadataDate の順に取得（秒なし形式にも対応）"""
    for tag in ["-DateTimeOriginal", "-CreateDate", "-MetadataDate"]:
        result = subprocess.run(
            ["exiftool", tag, "-s3", fpath],
            capture_output=True, text=True
        )
        val = result.stdout.strip()
        if val:
            m = re.match(r"(\d{4}):(\d{2}):(\d{2}) (\d{2}):(\d{2}):?(\d{2})?", val)
            if m:
                Y, Mo, D, H, Mi, S = m.groups()
                return Y, Mo, D, H, Mi, S or "00"
    return None

# ファイル名パターン（通常形式・編集済み形式の両方）
PATTERN = re.compile(
    r"^(\d{4}-\d{2}-\d{2})_(\d{2}-\d{2}-\d{2})_(\d+)( \([^)]+\))?\.(\w+)$"
)
# (1)サフィックス付きパターン
PATTERN_DUP = re.compile(
    r"^(\d{4})-(\d{2})-(\d{2})_(\d{2})-(\d{2})-(\d{2})_\d+\(\d+\)\.\w+$"
)

# ── Step A: 既存ハッシュDB構築 ──────────────────────────────
log("===== Amazon Photos 整理パイプライン開始 =====")
log("Step A: 既存ファイルのハッシュDB構築中...")

existing_hashes = set()
for root, _, files in os.walk(DEST_BASE):
    for fname in files:
        if fname.startswith("."): continue
        try:
            existing_hashes.add(md5_file(os.path.join(root, fname)))
        except Exception:
            pass

log(f"  既存ファイル数: {len(existing_hashes)}件")

# ── Step B: ソースファイル収集 ─────────────────────────────
log("Step B: ソースファイル収集中...")

source_dirs = sorted([
    os.path.join(DOWNLOADS_DIR, d)
    for d in os.listdir(DOWNLOADS_DIR)
    if re.match(r"^AmazonPhotos(-\d+)?$", d)
    and os.path.isdir(os.path.join(DOWNLOADS_DIR, d))
])

source_files = []
for sdir in source_dirs:
    for fname in sorted(os.listdir(sdir)):
        fpath = os.path.join(sdir, fname)
        if os.path.isfile(fpath) and not fname.startswith("."):
            source_files.append(fpath)

log(f"  ソースフォルダ数: {len(source_dirs)}件")
log(f"  ソースファイル数: {len(source_files)}件")

# ── Step C: 処理（日付修正・重複除去・移動）──────────────
log("Step C: ファイル処理中...")
batch_hashes = {}

for fpath in source_files:
    fname = os.path.basename(fpath)
    stats["processed"] += 1

    if stats["processed"] % 500 == 0:
        log(f"  ... {stats['processed']}/{len(source_files)}件処理中")

    m = PATTERN.match(fname)
    if not m:
        # (1)サフィックス付きファイル名から日付抽出
        md = PATTERN_DUP.match(fname)
        if md:
            Y, Mo, D, H, Mi, S = md.groups()
        else:
            # EXIFフォールバック（MOAxxx.jpg / IMG_xxxx.JPG など）
            exif = get_exif_date(fpath)
            if exif:
                Y, Mo, D, H, Mi, S = exif
            else:
                log(f"  [SKIP] 日付取得不可: {fname}")
                stats["errors"] += 1
                continue
        year, month = Y, Mo
        # タイムスタンプ設定・重複チェック・移動（以下共通処理へ）
        try:
            h = md5_file(fpath)
        except Exception as e:
            log(f"  [ERROR] ハッシュエラー: {fname}: {e}")
            stats["errors"] += 1
            continue
        if h in existing_hashes or h in batch_hashes:
            log(f"  [DUP] {fname} → 重複削除")
            stats["duplicates"] += 1
            os.remove(fpath)
            continue
        batch_hashes[h] = fpath
        try:
            subprocess.run(["touch", "-t", f"{Y}{Mo}{D}{H}{Mi}.{S}", fpath], check=True, capture_output=True)
        except Exception:
            pass
        dest_dir  = os.path.join(DEST_BASE, year, f"{year}-{month}")
        os.makedirs(dest_dir, exist_ok=True)
        dest_path = os.path.join(dest_dir, fname)
        if os.path.exists(dest_path) and md5_file(dest_path) == h:
            log(f"  [DUP] {fname} → 移動先と同一")
            stats["duplicates"] += 1
            os.remove(fpath)
            existing_hashes.add(h)
            continue
        try:
            shutil.copy2(fpath, dest_path)
            os.remove(fpath)
            stats["moved"] += 1
            existing_hashes.add(h)
            log(f"  [EXIF] {fname} → {year}/{year}-{month}/")
        except Exception as e:
            log(f"  [ERROR] {fname}: {e}")
            stats["errors"] += 1
        continue

    date_part, time_part, _, suffix, ext = m.groups()
    year  = date_part[:4]
    month = date_part[5:7]

    try:
        h = md5_file(fpath)
    except Exception as e:
        log(f"  [ERROR] ハッシュエラー: {fname}: {e}")
        stats["errors"] += 1
        continue

    # 重複チェック（既存 or バッチ内）
    if h in existing_hashes or h in batch_hashes:
        log(f"  [DUP] {fname} → 重複削除")
        stats["duplicates"] += 1
        os.remove(fpath)
        continue

    batch_hashes[h] = fpath

    # タイムスタンプを撮影日に修正
    try:
        set_timestamp(fpath, date_part, time_part)
    except Exception as e:
        log(f"  [WARN] 日付設定失敗: {fname}: {e}")

    # 移動先
    dest_dir  = os.path.join(DEST_BASE, year, f"{year}-{month}")
    os.makedirs(dest_dir, exist_ok=True)
    dest_path = os.path.join(dest_dir, fname)

    # 同名ファイル衝突チェック
    if os.path.exists(dest_path):
        if md5_file(dest_path) == h:
            log(f"  [DUP] {fname} → 移動先と同一")
            stats["duplicates"] += 1
            os.remove(fpath)
            existing_hashes.add(h)
            continue
        base, ex = os.path.splitext(fname)
        dest_path = os.path.join(dest_dir, f"{base}_alt{ex}")

    # コピー後ソース削除（安全なMOVE）
    try:
        shutil.copy2(fpath, dest_path)
        os.remove(fpath)
        stats["moved"] += 1
        existing_hashes.add(h)
    except Exception as e:
        log(f"  [ERROR] 移動失敗: {fname}: {e}")
        stats["errors"] += 1

# ── Step D: 空になったソースフォルダを削除 ──────────────
log("Step D: ソースフォルダ後処理...")
for sdir in source_dirs:
    try:
        remaining = [f for f in os.listdir(sdir) if not f.startswith(".")]
        if len(remaining) == 0:
            shutil.rmtree(sdir)
            log(f"  削除済み: {os.path.basename(sdir)}/")
        else:
            log(f"  [残存] {os.path.basename(sdir)}/ に {len(remaining)}件")
    except Exception as e:
        log(f"  [ERROR] フォルダ削除エラー: {e}")

# ── 完了サマリー ────────────────────────────────────────
total = sum(len(files) for _, _, files in os.walk(DEST_BASE))
log("===== 処理完了 =====")
log(
    f"処理済み: {stats['processed']}件 / "
    f"移動: {stats['moved']}件 / "
    f"重複削除: {stats['duplicates']}件 / "
    f"エラー: {stats['errors']}件"
)
log(f"整理済み_Amazon 累計: {total}件")
```

### Step 3 — zip ファイル削除

Step 1.5 で解凍した zip ファイルを削除する。

```bash
cd ~/Downloads && rm AmazonPhotos*.zip
```

### Step 4 — 結果報告

処理完了後、以下をユーザーに報告する。

- 解凍した zip ファイル数（あった場合）
- 移動件数・重複削除件数・エラー件数
- 整理済み_Amazon の累計ファイル数
- エラーや残存ファイルがある場合はその詳細

---

## 実行方法

```
/organize-amazon-photos
```
