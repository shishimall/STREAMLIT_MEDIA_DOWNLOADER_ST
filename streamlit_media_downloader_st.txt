# streamlit media downloader

import streamlit as st
import tempfile
from yt_dlp import YoutubeDL
from pathlib import Path

st.set_page_config(page_title="Media Downloader (Permissioned)", layout="centered")

st.title("📥 Media Downloader (Permissioned)")
st.caption("※ ご自身にダウンロード権限のあるURLのみ使用してください。")

# ---- 入力UI ----
agree = st.checkbox("私はこのURLのコンテンツをダウンロードする権利があります。")
url = st.text_input("動画/音声のURL", placeholder="https://...")
rename_on = st.checkbox("日本語表記のかな置換（朝→あさ / 昼→ひる / 夜→よる）", value=True)

# ---- 実行ボタンをページ上部に固定 ----
run = st.button("▶ 処理開始", type="primary", disabled=not (agree and url))

# ---- 進捗・ログ ----
log_box = st.empty()
progress = st.progress(0, text="待機中…")

def find_ffmpeg():
    from shutil import which
    return which("ffmpeg") or ""

def build_ydl_opts(tmpdir: str, fmt: str):
    ffmpeg_path = find_ffmpeg()
    outtmpl = str(Path(tmpdir) / "%(title)s.%(ext)s")
    return {
        "format": fmt,
        "outtmpl": outtmpl,
        "ffmpeg_location": ffmpeg_path if ffmpeg_path else None,
        "noplaylist": True,
        "merge_output_format": "mp4",
        "progress_hooks": [],
        "quiet": True,
        "nocheckcertificate": True,
    }

def hook_factory():
    state = {"last_percent": 0}
    def _hook(d):
        if d["status"] == "downloading":
            p = d.get("_percent_str", "").strip().replace("%", "")
            try:
                pct = int(float(p))
            except:
                pct = state["last_percent"]
            state["last_percent"] = max(state["last_percent"], min(pct, 100))
            progress.progress(state["last_percent"], text=f"ダウンロード中… {state['last_percent']}%")
            log_box.write(d.get("filename", "") + "  " + d.get("_eta_str", ""))
        elif d["status"] == "finished":
            progress.progress(100, text="変換/結合処理中…")
            log_box.write("ダウンロード完了。処理中…")
    return _hook

def kana_rename(name: str) -> str:
    return name.replace("朝","あさ").replace("昼","ひる").replace("夜","よる")

def pick_single_file(folder: str):
    exts = [".mp4",".m4a",".webm",".mp3"]
    files = [f for f in Path(folder).iterdir() if f.is_file()]
    files.sort(key=lambda p: p.stat().st_mtime, reverse=True)
    for e in exts:
        for f in files:
            if f.suffix.lower() == e:
                return f
    return files[0] if files else None

# ---- 実行処理 ----
if run:
    if not agree:
        st.error("権利確認にチェックしてください。")
    elif not url.strip():
        st.error("URLを入力してください。")
    else:
        try:
            with tempfile.TemporaryDirectory() as tmpdir:
                formats = ["bestvideo+bestaudio/best","best","bestaudio","worst"]
                f, last_err = None, None

                for fmt in formats:
                    try:
                        ydl_opts = build_ydl_opts(tmpdir, fmt)
                        ydl_opts["progress_hooks"].append(hook_factory())
                        log_box.write(f"試行: {fmt}")
                        with YoutubeDL(ydl_opts) as ydl:
                            ydl.download([url])
                        f = pick_single_file(tmpdir)
                        if f: break
                    except Exception as e:
                        last_err = e
                        log_box.write(f"⚠ {fmt} 失敗: {e}")

                if not f:
                    raise Exception(f"全フォーマット失敗: {last_err}")

                # ---- 成功処理 ----
                final_name = f.name
                if rename_on:
                    new = kana_rename(f.stem) + f.suffix
                    new_path = f.parent / new
                    if not new_path.exists():
                        f.rename(new_path)
                        f = new_path
                        final_name = new

                with open(f,"rb") as r:
                    data = r.read()

                progress.progress(100, text="完了！ダウンロードできます。")
                st.success("✅ ダウンロード準備ができました。")

                # 🎯 完了メッセージの直下にのみ出現
                st.download_button(
                    label="💾 結果ファイルを保存",
                    data=data,
                    file_name=final_name,
                    mime="application/octet-stream",
                )

        except Exception as e:
            st.error(f"エラー: {e}")
