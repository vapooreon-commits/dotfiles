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

**確認なしで自動実行する。** Seagate がマウントされていない場合のみ中断してユーザーに伝える。それ以外の問題（ソースなし・既存ファイルあり等）は自動処理して最後に結果報告する。

以下の Python スクリプトを `/tmp/organize_amazon_photos.py` に書き出し、実行し、実行後に削除する。

```python
#!/usr/bin/env python3
import os, re, hashlib, shutil, subprocess, glob
from datetime import datetime

DOWNLOADS_DIR = os.path.expanduser("~/Downloads")
SEAGATE_BASE  = "/Volumes/Seagate Hub"
DEST_BASE     = os.path.join(SEAGATE_BASE, "整理済み_Amazon")
LOG_FILE      = os.path.join(SEAGATE_BASE, "amazon_pipeline_log.txt")

# ── 事前チェック ────────────────────────────────────────
if not os.path.isdir(SEAGATE_BASE):
    print("ERROR: Seagate がマウントされていません。/Volumes/Seagate Hub が見つかりません。")
    exit(1)

stats = {"processed": 0, "moved": 0, "duplicates": 0, "errors": 0, "zips": 0}

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
        subprocess.run(["touch", "-t", f"{Y}{Mo}{D}{H}{Mi}.{S}", fpath],
                       check=True, capture_output=True)

def get_exif_date(fpath):
    for tag in ["-DateTimeOriginal", "-CreateDate", "-MetadataDate"]:
        result = subprocess.run(["exiftool", tag, "-s3", fpath],
                                capture_output=True, text=True)
        val = result.stdout.strip()
        if val:
            m = re.match(r"(\d{4}):(\d{2}):(\d{2}) (\d{2}):(\d{2}):?(\d{2})?", val)
            if m:
                Y, Mo, D, H, Mi, S = m.groups()
                return Y, Mo, D, H, Mi, S or "00"
    return None

PATTERN = re.compile(
    r"^(\d{4}-\d{2}-\d{2})_(\d{2}-\d{2}-\d{2})_(\d+)( \([^)]+\))?\.(\w+)$"
)
PATTERN_DUP = re.compile(
    r"^(\d{4})-(\d{2})-(\d{2})_(\d{2})-(\d{2})-(\d{2})_\d+\(\d+\)\.\w+$"
)

log("===== Amazon Photos 整理パイプライン開始 =====")

# ── Step 1: zip 自動解凍 ────────────────────────────────
zip_files = sorted(glob.glob(os.path.join(DOWNLOADS_DIR, "AmazonPhotos*.zip")))
if zip_files:
    log(f"Step 1: zip解凍中 ({len(zip_files)}件)...")
    for zf in zip_files:
        dest_dir = zf[:-4]  # .zip を除いたフォルダ名
        subprocess.run(["unzip", "-q", zf, "-d", dest_dir], check=True)
        stats["zips"] += 1
    log(f"  解凍完了: {stats['zips']}件")

# ── Step 2: 既存ハッシュDB構築 ─────────────────────────
log("Step 2: 既存ファイルのハッシュDB構築中...")
existing_hashes = set()
if os.path.isdir(DEST_BASE):
    for root, _, files in os.walk(DEST_BASE):
        for fname in files:
            if fname.startswith("."): continue
            try:
                existing_hashes.add(md5_file(os.path.join(root, fname)))
            except Exception:
                pass
log(f"  既存ファイル数: {len(existing_hashes)}件")

# ── Step 3: ソースファイル収集 ─────────────────────────
log("Step 3: ソースファイル収集中...")
source_dirs = sorted([
    os.path.join(DOWNLOADS_DIR, d)
    for d in os.listdir(DOWNLOADS_DIR)
    if re.match(r"^AmazonPhotos(-\d+)?$", d)
    and os.path.isdir(os.path.join(DOWNLOADS_DIR, d))
])
source_files = []
for sdir in source_dirs:
    for root, dirs, files in os.walk(sdir):
        for fname in sorted(files):
            if not fname.startswith("."):
                source_files.append(os.path.join(root, fname))
log(f"  ソースフォルダ数: {len(source_dirs)}件 / ソースファイル数: {len(source_files)}件")

# ── Step 4: 処理（日付修正・重複除去・移動）──────────
log("Step 4: ファイル処理中...")
batch_hashes = {}

for fpath in source_files:
    fname = os.path.basename(fpath)
    stats["processed"] += 1

    if stats["processed"] % 500 == 0:
        log(f"  ... {stats['processed']}/{len(source_files)}件処理中")

    m = PATTERN.match(fname)
    if not m:
        md = PATTERN_DUP.match(fname)
        if md:
            Y, Mo, D, H, Mi, S = md.groups()
        else:
            exif = get_exif_date(fpath)
            if exif:
                Y, Mo, D, H, Mi, S = exif
            else:
                log(f"  [SKIP] 日付取得不可: {fname}")
                stats["errors"] += 1
                continue
        year, month = Y, Mo
        try:
            h = md5_file(fpath)
        except Exception as e:
            log(f"  [ERROR] ハッシュエラー: {fname}: {e}")
            stats["errors"] += 1
            continue
        if h in existing_hashes or h in batch_hashes:
            stats["duplicates"] += 1
            os.remove(fpath)
            continue
        batch_hashes[h] = fpath
        try:
            subprocess.run(["touch", "-t", f"{Y}{Mo}{D}{H}{Mi}.{S}", fpath],
                           check=True, capture_output=True)
        except Exception:
            pass
        dest_dir  = os.path.join(DEST_BASE, year, f"{year}-{month}")
        os.makedirs(dest_dir, exist_ok=True)
        dest_path = os.path.join(dest_dir, fname)
        if os.path.exists(dest_path) and md5_file(dest_path) == h:
            stats["duplicates"] += 1
            os.remove(fpath)
            existing_hashes.add(h)
            continue
        try:
            shutil.copy2(fpath, dest_path)
            os.remove(fpath)
            stats["moved"] += 1
            existing_hashes.add(h)
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

    if h in existing_hashes or h in batch_hashes:
        stats["duplicates"] += 1
        os.remove(fpath)
        continue

    batch_hashes[h] = fpath

    try:
        set_timestamp(fpath, date_part, time_part)
    except Exception as e:
        log(f"  [WARN] 日付設定失敗: {fname}: {e}")

    dest_dir  = os.path.join(DEST_BASE, year, f"{year}-{month}")
    os.makedirs(dest_dir, exist_ok=True)
    dest_path = os.path.join(dest_dir, fname)

    if os.path.exists(dest_path):
        if md5_file(dest_path) == h:
            stats["duplicates"] += 1
            os.remove(fpath)
            existing_hashes.add(h)
            continue
        base, ex = os.path.splitext(fname)
        dest_path = os.path.join(dest_dir, f"{base}_alt{ex}")

    try:
        shutil.copy2(fpath, dest_path)
        os.remove(fpath)
        stats["moved"] += 1
        existing_hashes.add(h)
    except Exception as e:
        log(f"  [ERROR] 移動失敗: {fname}: {e}")
        stats["errors"] += 1

# ── Step 5: ソースフォルダ後処理 ──────────────────────
for sdir in source_dirs:
    try:
        remaining = [f for f in os.listdir(sdir) if not f.startswith(".")]
        if len(remaining) == 0:
            shutil.rmtree(sdir)
        else:
            log(f"  [残存] {os.path.basename(sdir)}/ に {len(remaining)}件")
    except Exception as e:
        log(f"  [ERROR] フォルダ削除エラー: {e}")

# ── Step 6: zip削除 ────────────────────────────────────
for zf in zip_files:
    try:
        os.remove(zf)
    except Exception:
        pass

# ── 完了サマリー ────────────────────────────────────────
total = sum(len(files) for _, _, files in os.walk(DEST_BASE))
log("===== 処理完了 =====")
log(
    f"zip解凍: {stats['zips']}件 / "
    f"処理済み: {stats['processed']}件 / "
    f"移動: {stats['moved']}件 / "
    f"重複削除: {stats['duplicates']}件 / "
    f"エラー: {stats['errors']}件"
)
log(f"整理済み_Amazon 累計: {total}件")
```

## 結果報告

スクリプト完了後、以下をユーザーに報告する。

- zip解凍数・移動件数・重複削除件数・エラー件数
- 整理済み_Amazon の累計ファイル数
- エラーや残存ファイルがある場合はその詳細

---

## 実行方法

```
/organize-amazon-photos
```
