# PL-Repo.

114-1 師大科技系程式語言
 - 授課教師：蔡芸琤老師
 - 姓名：趙嘉鋐
 - 系級：科技系二年級
 - 課程筆記區
   ---
 - 作業連結區
   ---
   - [HW1](https://github.com/icecat14159/PL-Repo./blob/main/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80HW01_%E6%97%A5%E5%B8%B8%E6%94%AF%E5%87%BA%E9%80%9F%E7%AE%97%E8%88%87%E5%88%86%E6%94%A4.ipynb)
   - [HW2](https://github.com/icecat14159/PL-Repo./blob/main/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80HW02_%E6%88%90%E7%B8%BE%E4%B8%80%E6%9C%AC%E9%80%9A.ipynb)(10/5 晚20:00測試時原模型gemini-1.5-flash-latest仍可用(第四段執行正常)，10/6早9:30再次測試時已不可用(第四段出現錯誤)，故改為gemini-flash-latest，但仍然超時)
   - [HW3](https://github.com/icecat14159/PL-Repo./blob/main/HW03_%E6%9B%B8%E7%B1%8D%E6%B8%85%E5%96%AE.ipynb)，[匯出的csv檔案](https://github.com/icecat14159/PL-Repo./blob/main/books.csv)
   - [HW4](https://github.com/icecat14159/PL-Repo./blob/main/HW04_%E7%B6%B2%E9%A0%81%E7%88%AC%E8%9F%B2.ipynb)(原先抓取成功，試算表也有內容。後來第一段再次測試時出現狀態碼429，故顯示為抓取0筆文章)
   - [HW5](https://github.com/icecat14159/PL-Repo./blob/main/HW05%E5%B0%8B%E6%89%BE%E8%94%AC%E9%A3%9F%E9%A4%90%E5%BB%B3.ipynb)
   - [HW6](https://github.com/icecat14159/PL-Repo./blob/main/HW06_%E6%9B%B8%E7%B1%8D%E6%B8%85%E5%96%AE2.ipynb)
 - 專題連結區
   ---
   - [提案簡報](https://www.canva.com/design/DAG5JeIAvPs/LlJdgvb4HQ7s0yHEz_eHrw/edit?utm_content=DAG5JeIAvPs&utm_campaign=designshare&utm_medium=link2&utm_source=sharebutton)
   - [提案影片](https://youtu.be/LoHRQU6lJt4)
   - [專案程式](追番清單.ipynb)

  📺 追番清單管理系統（Gradio + Google Sheet）
  
  本專案提供一個基於 Gradio 的網頁介面，用於統一管理
  小說、漫畫、動畫的追蹤清單，並將資料儲存於 Google Sheet。
  
  --------------------------------------------------
  🚀 功能特色
  --------------------------------------------------
  - 小說／漫畫／動畫三種媒體分開管理
  - 可直接在表格中編輯作品資訊
  - 一鍵同步至 Google Sheet
  - 長時間未更新作品自動示警
  - 跨媒體同名作品 ID 對應顯示
  - 依更新網址自動檢查進度
  - 跨媒體搜尋與篩選功能
  --------------------------------------------------
  程式片段說明
  --------------------------------------------------
### 第一步:安装依附元件
```bash
!pip install gspread gradio pandas beautifulsoup4 requests
from google.colab import auth #身分驗證
from google.auth import default #憑證
import gspread
import gradio as gr
import pandas as pd
import datetime #日期
import io #io
import requests
from bs4 import BeautifulSoup
import re
!pip install feedparser
import feedparser
```
### 第二步:授權連線、連結試算表
```bash
# 授權連線 Google Sheet
auth.authenticate_user()  #要求授權
creds, _ = default()  #獲取憑證
gc = gspread.authorize(creds) #授權
# 連結試算表
sheet_url = "https://docs.google.com/spreadsheets/d/1fGjPVqPHt3flo-LxBNU9EZl_ZMvg-I8UXfMhIY-jDNA/edit?usp=sharing"  #GoogleSheet連結
sh = gc.open_by_url(sheet_url)  #進入試算表
WORKSHEETS = {
    "novel": sh.worksheet("小說清單"),
    "comic": sh.worksheet("漫畫清單"),
    "anime": sh.worksheet("動畫清單")
}
SNAPSHOT_SHEET = sh.worksheet("update_snapshot")
```
### 第三步:讀取、寫回試算表
```bash
# 讀取試算表
COLUMNS = ["ID", "作品名稱", "作者", "評級", "進度", "狀態", "最後紀錄日期", "更新網址"]  #資料庫8欄
DISPLAY_COLUMNS = COLUMNS + ["距離上次紀錄(天)"] #顯示用8欄

def R_D(media):
    ws = WORKSHEETS[media]
    records = ws.get_all_records()
    df = pd.DataFrame(records)

    if df.empty:
        df = pd.DataFrame(columns=COLUMNS)

    df["ID"] = pd.to_numeric(df["ID"], errors="coerce")

    def calc_days(d):
        try:
            return (datetime.date.today() - datetime.datetime.strptime(d, "%Y/%m/%d").date()).days
        except:
            return None

    df["距離上次紀錄(天)"] = df["最後紀錄日期"].apply(calc_days)
    return df

def read_data(media):
  worksheet = WORKSHEETS[media]
  records = worksheet.get_all_records()
  df = pd.DataFrame(records)

  if df.empty:
    df = pd.DataFrame(columns=COLUMNS)

  #確保ID
  if "ID" in df.columns:
    df["ID"] = pd.to_numeric(df["ID"], errors="coerce")

  #計算距離上次紀錄時間
  today = datetime.date.today()

  def calc_days(date_str):
    try:
      last_date = datetime.datetime.strptime(date_str, "%Y/%m/%d").date()
      return (today - last_date).days
    except:
      return None
  df["距離上次紀錄(天)"] = df["最後紀錄日期"].apply(calc_days)

  return df
# 寫回試算表
def write_data(df, media):
  worksheet = WORKSHEETS[media]
  df = df[COLUMNS] #強制只留下資料庫欄位
  worksheet.clear()
  worksheet.append_row(COLUMNS)
  worksheet.append_rows(df.values.tolist())
```
### 第四步:新增、修改、刪除紀錄
```bash
# 新增紀錄
def add_record(name, author, rating, progress, condition, date, media):
  worksheet = WORKSHEETS[media]
  df = read_data(media)
  if not date:
      date = datetime.date.today().strftime("%Y/%m/%d")
  if df.empty or df["ID"].dropna().empty: #產生ID
    new_id = 1
  else:
    new_id = int(df["ID"].max()) + 1
  new_entry = {"ID": new_id, "作品名稱": name, "作者": author, "評級": rating, "進度": progress, "狀態": condition, "最後紀錄日期": date}
  df = pd.concat([df, pd.DataFrame([new_entry])], ignore_index=True)
  write_data(df, media)
  return df
# 修改紀錄
def edit_record(record_id, name, author, rating, progress, condition, date, media):
  worksheet = WORKSHEETS[media]
  df = read_data(media)
  if record_id not in df["ID"].values:
    return "找不到此 ID"
  if not date:
    date = datetime.date.today().strftime("%Y/%m/%d")

  df.loc[df["ID"] == record_id, ["作品名稱", "作者", "評級", "進度", "狀態", "最後紀錄日期"]] = \
      [name, author, rating, progress, condition, date]
  write_data(df, media)
  return df
# 刪除紀錄
def delete_record(record_id, media):
  worksheet = WORKSHEETS[media]
  df = read_data()
  if record_id not in df["ID"].values:
    return "找不到此 ID"
  df = df[df["ID"] != record_id]
  write_data(df, media)
  return df
```
### 第五步:以Gradio改動表單
```bash
# gradio用 改動表單
def save_table(table_df, media):
  worksheet = WORKSHEETS[media]
  df = pd.DataFrame(table_df, columns=DISPLAY_COLUMNS) # 轉成DataFrame並強制欄位順序
  df = df[COLUMNS] #丟掉第8欄
  df["ID"] = range(1, len(df) + 1) #修正ID（防止使用者亂改/刪列）

  today = datetime.date.today().strftime("%Y/%m/%d")
  df["最後紀錄日期"] = df["最後紀錄日期"].replace("", today)
  write_data(df, media)

  return read_data(media)
# gradio用 新增資料列
def add_empty_row(table_df, media):
  worksheet = WORKSHEETS[media]
  df = pd.DataFrame(table_df, columns=DISPLAY_COLUMNS)

  new_row = {
    "ID": "",
    "作品名稱": "",
    "作者": "",
    "評級": "",
    "進度": "",
    "狀態": "",
    "最後紀錄日期": "",
    "距離上次紀錄(天)": ""
  }

  df = pd.concat([df, pd.DataFrame([new_row])], ignore_index=True)
  return df
def delete_and_refresh(record_id, media):
  if record_id is None:
    return read_data(media)
  try:
    record_id = int(record_id)
  except:
    return read_data(media)
  df = read_data(media)
  if record_id not in df["ID"].values:
    return df
  # 執行刪除
  df = df[df["ID"] != record_id].reset_index(drop=True)
  if df.empty:
    return df
  df["ID"] = range(1, len(df) + 1)
  write_data(df, media)
  return read_data(media)
```
### 第六步:示警清單
```bash
ALERT_RULES = {
  "novel": 90, #小說3個月
  "comic": 30, #漫畫1個月
  "anime": 14 #動畫14天
}
ALERT_COLUMNS = [
    "ID",
    "作品名稱",
    "作者",
    "進度",
    "狀態",
    "最後紀錄日期",
    "距離上次紀錄(天)"
]

def get_alert_table(media):
  df = read_data(media)
  if df.empty:
      return df
  threshold = ALERT_RULES[media]
  alert_df = df[
      (df["狀態"] == "未完結") &
      (df["距離上次紀錄(天)"].notna()) &
      (df["距離上次紀錄(天)"] >= threshold)
  ]
  return alert_df.reset_index(drop=True)
```
### 第七步:跨媒體偵測
```bash
CROSS_MEDIA_CONFIG = {
    "novel": {
        "targets": ["comic", "anime"],
        "columns": ["對應漫畫ID", "對應動畫ID"]
    },
    "comic": {
        "targets": ["novel", "anime"],
        "columns": ["對應小說ID", "對應動畫ID"]
    },
    "anime": {
        "targets": ["novel", "comic"],
        "columns": ["對應小說ID", "對應漫畫ID"]
    }
}
def get_cross_columns(media):
  return ["ID", "作品名稱"] + CROSS_MEDIA_CONFIG[media]["columns"]

def build_cross_media_table(media):
  base_df = read_data(media)

  if base_df.empty:
      return pd.DataFrame(columns=get_cross_columns(media))

  targets = CROSS_MEDIA_CONFIG[media]["targets"]
  target_dfs = {t: read_data(t) for t in targets}

  rows = []

  for _, row in base_df.iterrows():
      title = row["作品名稱"]
      base_id = int(row["ID"])

      cross_ids = {}

      for t in targets:
          match = target_dfs[t][target_dfs[t]["作品名稱"] == title]
          cross_ids[t] = int(match.iloc[0]["ID"]) if not match.empty else ""

      # 至少有一個對應才顯示
      if any(cross_ids.values()):
          entry = {
              "ID": base_id,
              "作品名稱": title
          }

          for t, col_name in zip(targets, CROSS_MEDIA_CONFIG[media]["columns"]):
              entry[col_name] = cross_ids[t]

          rows.append(entry)

  return pd.DataFrame(rows, columns=get_cross_columns(media))
```
### 第八步:查詢系統
```bash
SEARCH_COLUMNS = [
    "ID", "作品名稱", "作者", "評級", "進度",
    "狀態", "最後紀錄日期", "距離上次紀錄(天)", "類型"
]
def get_all_media_data():
  dfs = []

  for media, label in [
      ("novel", "小說"),
      ("comic", "漫畫"),
      ("anime", "動畫")
  ]:
    df = read_data(media)
    if df.empty:
        continue

    df = df.copy()
    df["類型"] = label
    dfs.append(df)

  if not dfs:
    return pd.DataFrame(columns=SEARCH_COLUMNS)

  return pd.concat(dfs, ignore_index=True)

def search_works(keyword, rating, status, min_days):
  df = get_all_media_data()

  if df.empty:
      return df

  # 關鍵字搜尋（ID / 作品 / 作者）
  if keyword:
    keyword = str(keyword).strip()
    df = df[
      df["作品名稱"].astype(str).str.contains(keyword, case=False, na=False) |
      df["作者"].astype(str).str.contains(keyword, case=False, na=False) |
      df["ID"].astype(str).str.contains(keyword, na=False)
    ]

  # 評級篩選
  if rating != "全部":
    df = df[df["評級"] == rating]

  # 狀態篩選
  if status != "全部":
    df = df[df["狀態"] == status]

  # 距離上次紀錄
  if min_days is not None:
    df = df[df["距離上次紀錄(天)"] >= min_days]

  return df[SEARCH_COLUMNS]
```
### 第九步:構建紀錄更新用的snapshot
```bash
# ---------- Snapshot ----------
def load_snapshot():
    records = SNAPSHOT_SHEET.get_all_records()
    if not records:
        return pd.DataFrame(columns=["媒體", "ID", "作品名稱", "進度"])
    return pd.DataFrame(records)


def save_snapshot(df):
    SNAPSHOT_SHEET.clear()
    SNAPSHOT_SHEET.append_row(["媒體", "ID", "作品名稱", "進度"])
    if not df.empty:
        SNAPSHOT_SHEET.append_rows(df.values.tolist())
```
### 第十步:構建用來蒐集資料的parsers
```bash

def parse_lnovel(url):
    try:
        headers = {"User-Agent": "Mozilla/5.0"}
        resp = requests.get(url, headers=headers, timeout=10)
        resp.encoding = "utf-8"
        soup = BeautifulSoup(resp.text, "html.parser")

        volumes = []

        for text in soup.stripped_strings:
            text = text.strip()

            # ✅ 核心：抓「第X卷」，X 可為中文或數字
            m = re.search(r"第\s*([零一二三四五六七八九十\d]+)\s*卷", text)
            if m:
                raw = m.group(1)
                num = chinese_to_int(raw)
                if num > 0:
                    volumes.append(num)

        if not volumes:
            return "未知"

        latest = max(volumes)
        return f"第{latest}卷"

    except Exception:
        return "未知"

def parse_manhuagui(url):
    try:
        soup = BeautifulSoup(requests.get(url, timeout=10).text, "html.parser")
        chapters = soup.select(".chapter-list a")
        if chapters:
            return chapters[0].text.strip()
        return "未知"
    except:
        return "未知"

def parse_bahamut(url):
    return "請手動更新"

def parse_update_date(url):
    try:
        headers = {
            "User-Agent": "Mozilla/5.0"
        }
        html = requests.get(url, headers=headers, timeout=10).text
        soup = BeautifulSoup(html, "html.parser")

        # 常見更新日期文字
        for text in soup.stripped_strings:
            if "更新" in text and any(c.isdigit() for c in text):
                return text.strip()

        return "未知"
    except:
        return "未知"

def parse_generic(url):
    try:
        soup = BeautifulSoup(requests.get(url, timeout=10).text, "html.parser")
        for t in soup.stripped_strings:
            if any(k in t.lower() for k in ["話", "集", "章", "ep", "episode"]) and len(t) <= 15:
                return t.strip()
        return "未知"
    except:
        return "未知"

PARSERS = {
    "manhuagui.com": parse_manhuagui,
    "www.manhuagui.com": parse_manhuagui,
    "ani.gamer.com.tw": parse_bahamut,
    "lnovel.tw": parse_lnovel,
    "www.lnovel.tw": parse_lnovel,
}


def get_domain(url):
    try:
        return url.split("//")[1].split("/")[0]
    except:
        return ""


def get_latest_from_url(url, debug=True):
    domain = get_domain(url)

    if debug:
        print(f"[DEBUG] URL = {url}")
        print(f"[DEBUG] domain = {domain}")

    parser = PARSERS.get(domain)

    if not parser:
        if debug:
            print("[DEBUG] ❌ No parser found for this domain")
        return "未知"

    if debug:
        print(f"[DEBUG] ✅ Using parser: {parser.__name__}")

    try:
        result = parser(url)

        if debug:
            print(f"[DEBUG] 📤 Parser result = {result}")

        return result
    except Exception as e:
        if debug:
            print(f"[DEBUG] 💥 Parser exception: {e}")
        return "未知"

```
### 第十一步:建立檢測更新的機制
```bash
# ---------- 更新檢查 ----------
def detect_updates(media):
    df = R_D(media)
    snapshot = load_snapshot()
    alerts = []
    new_rows = []

    for _, r in df.iterrows():
        cid = str(r["ID"])
        title = r["作品名稱"]
        latest = get_latest_from_url(r.get("更新網址", ""))

        snap = snapshot[(snapshot["媒體"] == media) & (snapshot["ID"] == cid)]
        old = r["進度"]

        if latest != "未知" and latest != old:
            alerts.append(f"{title}：{old} → {latest}")
            df.loc[df["ID"] == int(cid), "進度"] = latest
            df.loc[df["ID"] == int(cid), "最後紀錄日期"] = TD()
            new_rows.append([media, cid, title, latest])
        else:
            new_rows.append([media, cid, title, old])

    #W_D(df, media)
    save_snapshot(pd.concat([
        snapshot[snapshot["媒體"] != media],
        pd.DataFrame(new_rows, columns=["媒體", "ID", "作品名稱", "進度"])
    ], ignore_index=True))

    return alerts
def get_gradio_alerts():
    msgs = []
    for m, label in [("novel", "小說"), ("comic", "漫畫"), ("anime", "動畫")]:
        for a in detect_updates(m):
            msgs.append(f"- {label}｜{a}")
    return "✅ 目前沒有更新" if not msgs else "📢 **更新檢查結果**\n\n" + "\n".join(msgs)
```
### 第十二步:Gradio操作介面
```bash
import gradio as gr

#gradio操作介面
with gr.Blocks() as app:
  gr.Markdown("追番清單")
  with gr.Tab("小說清單"): #分頁
    gr.Markdown("直接編輯清單後點擊儲存更變即可儲存!")
    refresh_novel = gr.Button("重新整理") #刷新按鈕
    with gr.Row():
      add_novel = gr.Button("新增一筆")
      del_novel_id = gr.Number(label="刪除 ID", precision=0)
      del_novel = gr.Button("刪除該筆", variant="stop")
    save_novel = gr.Button("儲存變更", variant="secondary")

    table_novel = gr.Dataframe( #表單
      value=read_data("novel"),
      headers=COLUMNS,
      interactive=True
    )

    gr.Markdown("系統偵測到以下作品存在跨媒體")
    cross_table_novel = gr.Dataframe(
        value=build_cross_media_table("novel"),
        headers=get_cross_columns("novel"),
        interactive=False
    )

    refresh_novel.click(fn=lambda: read_data("novel"), outputs=table_novel)
    add_novel.click(
        fn=lambda table: add_empty_row(table, "novel"),
        inputs=table_novel,
        outputs=table_novel
    )
    del_novel.click(
      fn=lambda rid: delete_and_refresh(rid, "novel"),
      inputs=del_novel_id,
      outputs=table_novel
    )

    save_novel.click(
        fn=lambda table: save_table(table, "novel"),
        inputs=table_novel,
        outputs=table_novel
    )

  with gr.Tab("漫畫清單"):
    gr.Markdown("直接編輯清單後點擊儲存更變即可儲存!")
    refresh_comic = gr.Button("重新整理") #刷新按鈕
    with gr.Row():
      add_comic = gr.Button("新增一筆")
      del_comic_id = gr.Number(label="刪除 ID", precision=0)
      del_comic = gr.Button("刪除該筆", variant="stop")
    save_comic = gr.Button("儲存變更", variant="secondary")

    table_comic = gr.Dataframe( #表單
      value=read_data("comic"),
      headers=COLUMNS,
      interactive=True
    )

    gr.Markdown("系統偵測到以下作品存在跨媒體")
    cross_table_comic = gr.Dataframe(
        value=build_cross_media_table("comic"),
        headers=get_cross_columns("comic"),
        interactive=False
    )

    refresh_comic.click(fn=lambda: read_data("comic"), outputs=table_comic)
    add_comic.click(
        fn=lambda table: add_empty_row(table, "comic"),
        inputs=table_comic,
        outputs=table_comic
    )
    del_comic.click(
      fn=lambda rid: delete_and_refresh(rid, "comic"),
      inputs=del_comic_id,
      outputs=table_comic
    )

    save_comic.click(
        fn=lambda table: save_table(table, "comic"),
        inputs=table_comic,
        outputs=table_comic
    )

  with gr.Tab("動畫清單"):
    gr.Markdown("直接編輯清單後點擊儲存更變即可儲存!")
    refresh_anime = gr.Button("重新整理") #刷新按鈕
    with gr.Row():
      add_anime = gr.Button("新增一筆")
      del_anime_id = gr.Number(label="刪除 ID", precision=0)
      del_anime = gr.Button("刪除該筆", variant="stop")
    save_anime = gr.Button("儲存變更", variant="secondary")

    table_anime = gr.Dataframe( #表單
      value=read_data("anime"),
      headers=COLUMNS,
      interactive=True
    )

    gr.Markdown("系統偵測到以下作品存在跨媒體")
    cross_table_anime = gr.Dataframe(
        value=build_cross_media_table("anime"),
        headers=get_cross_columns("anime"),
        interactive=False
    )

    refresh_anime.click(fn=lambda: read_data("anime"), outputs=table_anime)
    add_anime.click(
        fn=lambda table: add_empty_row(table, "anime"),
        inputs=table_anime,
        outputs=table_anime
    )
    del_anime.click(
      fn=lambda rid: delete_and_refresh(rid, "anime"),
      inputs=del_anime_id,
      outputs=table_anime
    )

    save_anime.click(
        fn=lambda table: save_table(table, "anime"),
        inputs=table_anime,
        outputs=table_anime
    )

  with gr.Tab("示警清單"):
    gr.Markdown("僅顯示「未完結」且長時間未更新的作品")

    alert_refresh = gr.Button("重新整理示警清單")

    gr.Markdown("小說示警（超過 3 個月）")
    alert_novel = gr.Dataframe(
        value=get_alert_table("novel"),
        headers=ALERT_COLUMNS,
        interactive=False
    )

    gr.Markdown("漫畫示警（超過 1 個月）")
    alert_comic = gr.Dataframe(
        value=get_alert_table("comic"),
        headers=ALERT_COLUMNS,
        interactive=False
    )

    gr.Markdown("動畫示警（超過 14 天）")
    alert_anime = gr.Dataframe(
        value=get_alert_table("anime"),
        headers=ALERT_COLUMNS,
        interactive=False
    )

    alert_refresh.click(
      fn=lambda: (
          get_alert_table("novel"),
          get_alert_table("comic"),
          get_alert_table("anime")
      ),
      outputs=[alert_novel, alert_comic, alert_anime]
    )

  with gr.Tab("查詢系統"):
    gr.Markdown("### 跨媒體作品查詢")

    with gr.Row():
      keyword = gr.Textbox(label="關鍵字（ID / 作品名稱 / 作者）")
      rating = gr.Dropdown(
        choices=["全部", "S", "A", "B", "C"],
        value="全部",
        label="評級"
      )
      status = gr.Dropdown(
        choices=["全部", "未完結", "完結"],
        value="全部",
        label="狀態"
      )
      min_days = gr.Number(
        label="距離上次紀錄 ≥ N 天",
        precision=0
      )

    search_btn = gr.Button("查詢")

    search_table = gr.Dataframe(
      headers=SEARCH_COLUMNS,
      interactive=False
    )

    search_btn.click(
      fn=search_works,
      inputs=[keyword, rating, status, min_days],
      outputs=search_table
    )
  with gr.Tab("更新提醒"):
      out = gr.Markdown()
      gr.Button("檢查更新").click(get_gradio_alerts, outputs=out)

app.launch()
```
  --------------------------------------------------
  🧭 介面說明
  --------------------------------------------------
  
  【1】小說／漫畫／動畫清單
  - 以分頁顯示三種媒體
  - 表格可直接編輯以下欄位：
    - 作品名稱、作者、評級、進度、狀態
    - 最後紀錄日期、更新網址
  
  功能按鈕：
  - 重新整理：從 Google Sheet 重新載入資料
  - 新增一筆：在表格底部新增空白列
  - 刪除該筆：輸入 ID 後刪除指定作品
  - 儲存變更：將目前表格內容寫回 Google Sheet
  
  ※ 修改後務必點擊「儲存變更」才會生效
  
  --------------------------------------------------
  
  【2】跨媒體清單
  - 自動偵測同名作品
  - 顯示其在其他媒體中的對應 ID
  - 僅供查看，不可直接編輯
  
  --------------------------------------------------
  
  【3】示警清單
  顯示「未完結」且超過指定天數未更新的作品：
  - 小說：90 天
  - 漫畫：30 天
  - 動畫：14 天
  
  點擊「重新整理示警清單」可更新結果
  
  --------------------------------------------------
  
  【4】查詢系統
  支援跨媒體搜尋，條件包含：
  - 關鍵字（ID／作品名稱／作者）
  - 評級（S / A / B / C）
  - 狀態（完結 / 未完結）
  - 距離上次紀錄天數
  
  --------------------------------------------------
  
  【5】更新提醒
  - 依據「更新網址」自動抓取最新進度(使用者必須手動輸入)
  - 若進度有變化，會顯示更新提示
  - 自動更新進度與最後紀錄日期
  
  --------------------------------------------------
  🗂 資料欄位說明
  --------------------------------------------------
  - ID：系統自動編號（請勿手動調整）
  - 作品名稱：必填，跨媒體比對依據
  - 作者：作者或原作
  - 評級：S / A / B / C
  - 進度：目前觀看／閱讀進度
  - 狀態：未完結 / 完結
  - 最後紀錄日期：最後一次更新時間
  - 更新網址：用於自動更新偵測
  
  --------------------------------------------------
  ⚠ 注意事項
  --------------------------------------------------
  - 請勿手動更動 ID 順序
  - 清空表格後儲存會同步清空 Google Sheet
  - 更新偵測僅支援已實作 parser 的網站
  
  --------------------------------------------------
  🛠 技術架構
  --------------------------------------------------
  - 前端介面：Gradio
  - 資料儲存：Google Sheet
  - 資料處理：Pandas
  - 網頁解析：Requests / BeautifulSoup
  """
