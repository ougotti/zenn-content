---
title: "方眼紙Excel→Markdown変換、結局LLMしか勝たん話【全手法比較】"
emoji: "📐"
type: "tech"
topics: ["python", "excel", "markdown", "docling", "llm"]
published: true
---

## はじめに

:::message
本記事は「方眼紙ExcelをAI/RAGに投入したいエンジニア」を対象としています。
:::

日本の現場でよく見かける「方眼紙Excel」——全セルを正方形に統一し、セル結合を多用してレイアウトを作るあのスタイルです。工事仕様書・工程表・設計図・申請書類など、建設・製造・行政の現場で今も現役です。

今回使ったサンプルはこんなものです。

![工事仕様書シート — セル結合を多用した典型的な方眼紙スタイル](/images/houganshi-excel-to-markdown/sheet_kouji_shiyousho.png)
*工事仕様書：基本情報・材料仕様・施工注意事項をセル結合で構造化*

![工程表シート — ガントチャート形式。色付きセルで工期を表現](/images/houganshi-excel-to-markdown/sheet_koutei_hyo.png)
*工程表：横軸が月、縦軸が工種のガントチャート。塗りつぶしセルで期間を表現*

![数量集計表シート — 数値データと埋め込みグラフ](/images/houganshi-excel-to-markdown/sheet_suryo_shukei.png)
*数量集計表：設計・計画・実施数量の比較表と埋め込み棒グラフ*

これをMarkdownに変換したい、というニーズが増えています。ドキュメント管理のモダン化、AIへの入力、RAGのインデックス化……いずれも「テキスト形式で意味が伝わること」が前提です。

**結論を先に言います。方眼紙Excel → Markdown変換は、LLMを使うのが現実解です。** 理由は後述しますが、従来ツールは「セル構造」を変換するのに対し、LLMは「意味構造」を再構成するため、そもそもアプローチが違います。

本記事では以下のツールを実際に試し、**インストール・実装・変換結果・処理時間**を比較します。

| # | ツール | 特徴 |
|---|--------|------|
| 1 | openpyxl | セルを直接走査・Pythonの定番 |
| 2 | pandas | DataFrame経由で変換 |
| 3 | markitdown | Microsoft製、マルチフォーマット対応 |
| 3b | markitdown（改善版） | 結合セルを事前展開する前処理を追加 |
| 4 | docling | IBM製、AI文書解析エンジン |
| 4b | docling（改善版） | 方眼紙向けオプションをチューニング |
| 5 | GitHub Models API | LLM（gpt-4o-mini / gpt-4o）で意味的に変換 |
| 6 | GitHub Copilot SDK | Copilot CLIをエンジンにした非同期変換 |

> **コード全体はこちら：** https://github.com/ougotti/houganshi-excel-to-markdown

---

## テスト環境

```
OS      : Windows 11
Python  : 3.12
Excel   : 自作サンプルデータ
```

### 仮想環境のセットアップ

```bash
python -m venv .venv
.venv\Scripts\activate   # Windows
# source .venv/bin/activate  # Mac/Linux

pip install openpyxl pandas tabulate markitdown docling pillow
pip install azure-ai-inference  # GitHub Models API 用
```

---

## テストデータについて

テストデータは `create_test_data.py` で自動生成しています。3シート構成で、方眼紙Excelの典型的な要素を含みます。

| シート | 内容 |
|--------|------|
| 工事仕様書 | タイトル・基本情報・材料表・施工注意事項・配置概要図（画像） |
| 工程表 | ガントチャート風（月別バー表示） |
| 数量集計表 | 集計テーブル・棒グラフ・断面図画像 |

**方眼紙の特徴：**
- 全列の幅・全行の高さがほぼ均一（約2〜3px）
- タイトル行は横50列を結合
- テーブルの各セルも複数行・複数列にまたがる

---

## 01. openpyxl — セルを直接走査

### インストール

```bash
pip install openpyxl
```

### 実装のポイント

openpyxlでは結合セルの値は左上セルにしか格納されていません。そのため**結合セルマップ**を先に構築してから走査します。

```python
import openpyxl
from openpyxl.utils import get_column_letter

def sheet_to_markdown(ws) -> str:
    # 結合セルマップを構築
    merged_map = {}
    for rng in ws.merged_cells.ranges:
        val = ws.cell(rng.min_row, rng.min_col).value
        for r in range(rng.min_row, rng.max_row + 1):
            for c in range(rng.min_col, rng.max_col + 1):
                merged_map[(r, c)] = val

    rows = []
    for r in range(1, (ws.max_row or 0) + 1):
        row = []
        for c in range(1, (ws.max_column or 0) + 1):
            v = merged_map.get((r, c), ws.cell(r, c).value)
            row.append(str(v).strip() if v is not None else "")
        rows.append(row)
    # ... Markdown テーブルに変換
```

### 変換結果（抜粋）

```markdown
| 工事名称 | 工事名称 | 工事名称 | ○○地区 新築工事 | ... | 工事場所 | 東京都○○区○○町1-2-3 | ... |
| 工事名称 | 工事名称 | 工事名称 | ○○地区 新築工事 | ... |
```

### 問題点：「50列繰り返し」問題

方眼紙の結合セルを展開すると、同じ値が50列以上に繰り返されます。Markdownに`colspan`の概念がないため、これは構造的に避けられません。

これは単なる実装の問題ではなく、**表現モデルの不一致**です。方眼紙Excelは「視覚レイアウト」で意味を表現しますが、Markdownは「論理構造」で表現します。この2つは根本的に異なるモデルであり、機械的な変換には限界があります。

---

## 02. pandas — DataFrame経由で変換

### インストール

```bash
pip install pandas tabulate openpyxl
```

### 実装

```python
import pandas as pd

def convert_sheet(ws_name, excel_path):
    df = pd.read_excel(excel_path, sheet_name=ws_name, header=0)
    return df.to_markdown(index=False)
```

### 変換結果（抜粋）

```markdown
| 工事名称 |            | ○○地区 新築工事 | ...  | 工事場所 | 東京都○○区○○町1-2-3 |
|:---------|:-----------|:----------------|:-----|:---------|:--------------------|
| 発注者   |            | 株式会社 ○○建設 | ...  | 施工者   | 株式会社 サンプル建設 |
```

### 問題点

- 結合セルの2行目以降が `NaN` になる
- 列名が `Unnamed: 1`, `Unnamed: 2` ... と自動付与される
- openpyxlより**見やすいが情報落ちがある**

---

## 03. markitdown — Microsoft製マルチフォーマット変換

### インストール

```bash
pip install markitdown
```

### 実装

```python
from markitdown import MarkItDown
from pathlib import Path

def convert(input_file: Path, output_dir: Path):
    md = MarkItDown()
    result = md.convert(str(input_file))
    md_text = result.text_content
    # ...
```

### 変換結果（抜粋）

```markdown
| 工 事 仕 様 書（サンプル） | Unnamed: 1 | Unnamed: 2 | ... |
| NaN | NaN | NaN | ... |
| 工事名称 | NaN | NaN | NaN | NaN | NaN | ○○地区 新築工事 | NaN | ... |
```

### 問題点

markitdownは内部で `pandas.read_excel()` を使っているため、**pandasと同じ NaN 問題**が発生します（v0.1.5時点）。

---

## 03b. markitdown 改善版 — 結合セルを事前展開

### 改善策

openpyxlで結合セルをすべて展開してから、BytesIOとしてmarkitdownに渡します。一時ファイルを作らずにメモリ内で処理できます。

```python
import io
import openpyxl
from markitdown import MarkItDown

def expand_merged_cells_to_stream(input_path) -> io.BytesIO:
    wb = openpyxl.load_workbook(input_path, data_only=True)
    for ws in wb.worksheets:
        for merge_range in list(ws.merged_cells.ranges):
            top_left_value = ws.cell(
                row=merge_range.min_row,
                column=merge_range.min_col
            ).value
            ws.unmerge_cells(str(merge_range))
            for row in ws.iter_rows(
                min_row=merge_range.min_row, max_row=merge_range.max_row,
                min_col=merge_range.min_col, max_col=merge_range.max_col,
            ):
                for cell in row:
                    cell.value = top_left_value
    buf = io.BytesIO()
    wb.save(buf)
    buf.seek(0)
    return buf

# markitdown に渡す
expanded_buf = expand_merged_cells_to_stream(input_file)
result = MarkItDown().convert(expanded_buf, file_extension=".xlsx")
```

### 改善後の結果

NaNは消えますが、50列の繰り返しは残ります。Markdownの構造的な限界です。

---

## 04. docling — IBM製AI文書解析エンジン

### インストール

```bash
pip install docling
```

:::message
初回実行時にHuggingFaceからモデルをダウンロードします（数百MB）。2回目以降はキャッシュから読み込まれます。
:::

### 実装

```python
from docling.document_converter import DocumentConverter

def convert(input_file, output_dir):
    converter = DocumentConverter()
    doc_result = converter.convert(str(input_file))
    doc = doc_result.document

    md_text = doc.export_to_markdown()
    # ...
```

### 変換結果（抜粋）

openpyxlと同様に50列繰り返し問題が発生します。ただし `treat_singleton_as_text` オプションを持つなど、方眼紙向けのチューニング余地があります（→04b参照）。

```markdown
| 工 事 仕 様 書（サンプル） | 工 事 仕 様 書（サンプル） | ... |（50列）
| 工事名称 | 工事名称 | ... | ○○地区 新築工事 | ... | 工事場所 | 東京都○○区○○町1-2-3 | ... |
| 発注者   | 発注者   | ... | 株式会社 ○○建設 | ... | 施工者   | 株式会社 サンプル建設 | ... |
```

デフォルトでは `gap_tolerance=0` のため、空白行/列で細かくテーブルが分割されます。画像は `<!-- image -->` として出力されます。

---

## 04b. docling 改善版 — 方眼紙向けオプション

### `MsExcelBackendOptions` でチューニング

```python
from docling.datamodel.backend_options import MsExcelBackendOptions
from docling.document_converter import DocumentConverter, ExcelFormatOption

backend_options = MsExcelBackendOptions(
    gap_tolerance=1,              # 空白1行/列まで同一テーブルとみなす
    treat_singleton_as_text=True, # 孤立セルをテキスト扱い（タイトル行など）
)

converter = DocumentConverter(
    allowed_formats=[InputFormat.XLSX],
    format_options={
        InputFormat.XLSX: ExcelFormatOption(
            backend_options=backend_options
        )
    },
)
```

| オプション | 効果 |
|------------|------|
| `gap_tolerance=1` | 空白行/列で細かく分割されすぎるのを防ぐ |
| `treat_singleton_as_text=True` | 単独セルをTableではなくTextとして扱い、ノイズ削減 |

### 変換結果

**Markdown出力**はデフォルト版と同様に50列繰り返し問題が残ります。

```markdown
| 工 事 仕 様 書（サンプル） | 工 事 仕 様 書（サンプル） | ... |（50列）
| 工事名称 | 工事名称 | ... | ○○地区 新築工事 | ... |
| 項目 | 項目 | ... | 材料名 | ... | 規格・品番 | ... | 数量 | ... |
| 1    | 1    | ... | コンクリート | ... | Fc=24N/mm² | ... | 120 | ... |
| 2. 施工注意事項 | 2. 施工注意事項 | ... |
| ① 本仕様書は設計図書と合わせて使用すること。 | ① ... |
```

ただし `gap_tolerance=1` + `treat_singleton_as_text=True` の効果で、今回のサンプルでは **3テーブル**にまとめて検出されました（工事仕様書: 27行×50列、工程表: 13行×55列、数量集計表: 9行×44列）。デフォルト版（`gap_tolerance=0`）では空白行/列ごとに細かく分割されます。

**HTML出力では rowspan/colspan が完全に保持されます。** これが04bの最大の強みです。

```html
<!-- 工事仕様書テーブルの一部 -->
<tr>
  <th rowspan="3" colspan="50">工 事 仕 様 書（サンプル）</th>
</tr>
<tr>
  <td rowspan="2" colspan="6">工事名称</td>
  <td rowspan="2" colspan="14">○○地区 新築工事</td>
  <td rowspan="2" colspan="6">工事場所</td>
  <td rowspan="2" colspan="24">東京都○○区○○町1-2-3</td>
</tr>
<tr>
  <td rowspan="2" colspan="6">発注者</td>
  <td rowspan="2" colspan="14">株式会社 ○○建設</td>
  <td rowspan="2" colspan="6">施工者</td>
  <td rowspan="2" colspan="24">株式会社 サンプル建設</td>
</tr>
```

Markdownとしては「50列繰り返し」が残りますが、HTMLとしては方眼紙のセル結合構造を正確に再現しています。後続処理でHTMLを扱える場合（ブラウザ表示・HTMLパーサーへの入力など）は、このアプローチが最も情報を保持します。

### HTML出力のコード例

doclingの大きな強みは **HTML出力でrowspan/colspanが保持される**点です。

```python
from docling_core.types.doc import TableItem  # docling_core からインポート

# Markdown（rowspan/colspanは失われる）
md_text = doc.export_to_markdown()

# HTML（rowspan/colspanが保持される）
html_text = doc.export_to_html()

# 個別のテーブルをDataFrameやHTMLで取り出す
for item, level in doc.iterate_items():
    if isinstance(item, TableItem):
        df = item.export_to_dataframe()   # pandas DataFrame
        html = item.export_to_html()      # rowspan/colspan付きHTML
```

---

## 05. GitHub Models API（LLM変換）— 意味を理解して変換

ここからが本命です。従来ツールは「セル構造」をそのままMarkdownに写しますが、LLMは「意味構造」を読み取って再構成します。方眼紙Excelの「視覚レイアウト→論理構造」変換は、LLMが唯一まともにこなせるアプローチです。

### セットアップ

GitHub Personal Access Token（`models:read` 権限）を取得し、環境変数に設定します。

```bash
# Windows
set GITHUB_TOKEN=ghp_xxxx

# Mac/Linux
export GITHUB_TOKEN=ghp_xxxx
```

```bash
pip install azure-ai-inference
```

### 実装：2つのアプローチ

**アプローチA: テキストモード（gpt-4o-mini）**

openpyxlでセルを抽出 → タブ区切りテキストとしてLLMに渡す

````python
from azure.ai.inference import ChatCompletionsClient
from azure.ai.inference.models import SystemMessage, UserMessage
from azure.core.credentials import AzureKeyCredential
import os

client = ChatCompletionsClient(
    endpoint="https://models.github.ai/inference",
    credential=AzureKeyCredential(os.environ["GITHUB_TOKEN"]),
)

prompt = f"""以下は「{ws.title}」というExcelシートのセルデータです（タブ区切り）。
方眼紙スタイルのExcelで、セル結合を多用したレイアウトになっています。
このデータを、内容が伝わるMarkdown形式に変換してください。

シートデータ：
```
{grid_text}
```

Markdownのみを出力してください。"""

response = client.complete(
    model="openai/gpt-4o-mini",
    messages=[
        SystemMessage("あなたはExcelドキュメントをMarkdownに変換する専門家です。"),
        UserMessage(prompt),
    ],
    max_tokens=4096,
)
md = response.choices[0].message.content
````

**アプローチB: ビジョンモード（gpt-4o）**

PILでシートをグリッド画像にレンダリング → Vision LLMに渡す

```python
from azure.ai.inference.models import (
    ImageContentItem, ImageUrl, TextContentItem, UserMessage,
)
import base64
from PIL import Image, ImageDraw, ImageFont

# シートをPNG化
img = Image.new("RGB", (img_w, img_h), "white")
draw = ImageDraw.Draw(img)
# ... セルをグリッド描画 ...
buf = io.BytesIO()
img.save(buf, format="PNG")
b64 = base64.b64encode(buf.getvalue()).decode()

# Vision LLMに送信
response = client.complete(
    model="openai/gpt-4o",
    messages=[
        SystemMessage("あなたはExcel画像からMarkdownを生成する専門家です。"),
        UserMessage([
            TextContentItem(text=prompt),
            ImageContentItem(
                image_url=ImageUrl(
                    url=f"data:image/png;base64,{b64}",
                    detail="high",
                )
            ),
        ]),
    ],
    max_tokens=4096,
)
```

### LLMの変換結果（テキストモード）

50列繰り返し問題が**完全に解決**されました。LLMが意味を理解して整形してくれます。

```markdown
# 工事仕様書（サンプル）

## 基本情報

| 項目     | 内容                           |
|----------|-------------------------------|
| 工事名称 | ○○地区 新築工事               |
| 工事場所 | 東京都○○区○○町1-2-3           |
| 発注者   | 株式会社 ○○建設               |
| 施工者   | 株式会社 サンプル建設         |
| 工期     | 2025年4月1日 ～ 2026年3月31日 |
| 図面番号 | S-001                         |
| 改訂     | Rev.0                         |

## 主要材料仕様

| 項目 | 材料名               | 規格・品番               | 数量 | 単位 | 備考     |
|------|----------------------|--------------------------|------|------|----------|
| 1    | コンクリート         | Fc=24N/mm²              | 120  | m³   | 基礎・床 |
| 2    | 鉄筋                 | SD345 D16                | 8.5  | t    | 主筋     |
| 3    | 構造用合板           | JAS特類 t=12mm           | 340  | 枚   | 床・壁下地 |
| 4    | 断熱材（グラスウール）| HG16-105mm              | 280  | m²   | 外壁充填 |
| 5    | アルミサッシ         | 複層ガラス Low-E          | 45   | 箇所 | 断熱仕様 |
| 6    | 屋根材               | カラーガルバリウム鋼板 t=0.4mm | 180 | m² |         |
| 7    | 外壁材               | 窯業系サイディング t=14mm | 310  | m²   | 塗装品   |

## 施工注意事項

1. 本仕様書は設計図書と合わせて使用すること。
2. 材料の搬入前に監督員の承認を得ること。
3. 各工程完了時に写真記録を行い、施工管理台帳に添付すること。
4. 寸法は原則として現場実測を優先し、疑義が生じた場合は監督員に確認すること。
5. 廃材の処理は廃棄物処理法に従い適正に行うこと。

## 配置概要図

[図: 配置概要図]
```

---

## 06. GitHub Copilot SDK — Copilot CLIをエンジンに変換

### GitHub Models API（05）との違い

| | 05: GitHub Models API | 06: GitHub Copilot SDK |
|--|----------------------|------------------------|
| SDK | `azure-ai-inference` | `github-copilot-sdk` |
| 通信先 | GitHub Models APIエンドポイント | **ローカルのGitHub Copilot CLIプロセス**（JSON-RPC） |
| 認証 | GitHub PAT（`models:read`） | GitHub Copilot **サブスクリプション**が必要 |
| API形式 | 同期・RESTライク | **非同期・イベント駆動** |
| ステータス | 一般提供 | テクニカルプレビュー（2026年1月発表） |

### インストール

```bash
# GitHub Copilot CLI（gh extension）が必要
gh extension install github/gh-copilot

pip install github-copilot-sdk   # Python 3.11 以上
```

### 実装

```python
import asyncio
from copilot import CopilotClient, PermissionHandler

async def convert_sheet(sheet_name: str, grid_text: str, model: str) -> str:
    client = CopilotClient()
    await client.start()

    session = await client.create_session({
        "on_permission_request": PermissionHandler.approve_all,
        "model": model,          # モデルを指定
    })

    result_parts = []
    done = asyncio.Event()

    def on_event(event):
        event_type = event.type.value if hasattr(event.type, "value") else str(event.type)
        if event_type == "assistant.message":
            content = getattr(event.data, "content", "")
            if content:
                result_parts.append(content)
        elif event_type in ("session.idle", "session.stop"):
            done.set()

    session.on(on_event)
    await session.send({"prompt": f"以下のExcelシートをMarkdownに変換してください:\n{grid_text}"})
    await asyncio.wait_for(done.wait(), timeout=120)

    await session.disconnect()
    await client.stop()
    return "".join(result_parts)
```

`CopilotClient()` がローカルの GitHub Copilot CLI プロセスを起動し、JSON-RPC で通信します。レスポンスはイベント駆動で受信します。

`create_session()` の `"model"` キーにモデルIDを渡すと、利用するLLMを切り替えられます。`client.list_models()` で利用可能なモデル一覧を取得できます。

### 利用可能モデル（抜粋）

```python
import asyncio
from copilot import CopilotClient

async def list():
    client = CopilotClient()
    await client.start()
    models = await client.list_models()
    for m in models:
        print(m["id"], m.get("billing_multiplier"))
    await client.stop()

asyncio.run(list())
```

実行結果（2026年3月時点）：

| モデルID | 概要 |
|----------|------|
| `gpt-4.1` | GPT-4系最新 |
| `gpt-5-mini` | GPT-5軽量版 |
| `claude-sonnet-4.6` | Claude Sonnet 最新 |
| `claude-haiku-4.5` | Claude 軽量高速 |
| `claude-opus-4.6` | Claude 最上位 |
| `gemini-3-pro-preview` | Gemini Pro プレビュー |

### モデル比較：gpt-4.1 vs claude-sonnet-4.6

同じ3シートを2モデルで変換し、処理時間と出力品質を比較しました。

#### 処理時間

| シート | gpt-4.1 | claude-sonnet-4.6 |
|--------|:-------:|:-----------------:|
| 工事仕様書 | 53.0秒 | 22.0秒 |
| 工程表 | 38.3秒 | 13.8秒 |
| 数量集計表 | 15.2秒 | 13.7秒 |
| **合計** | **112.7秒** | **55.6秒** |

Claude Sonnet 4.6 は gpt-4.1 の**約半分**の時間で変換を完了しました。

#### 出力品質

両モデルとも50列繰り返し問題は**完全に解決**されました。細かい点で差が見られます。

**工事仕様書（gpt-4.1）** — 列幅をスペースで揃えた整形スタイル：

```markdown
## 基本情報

| 工事名称         | ○○地区 新築工事         |
|------------------|------------------------|
| 工事場所         | 東京都○○区○○町1-2-3    |
| 発注者           | 株式会社 ○○建設         |
| 施工者           | 株式会社 サンプル建設   |
| 工期             | 2025年4月1日 ～ 2026年3月31日 |
...

## 2. 施工注意事項

1. 本仕様書は設計図書と合わせて使用すること。
2. 材料の搬入前に監督員の承認を得ること。
```

**工事仕様書（claude-sonnet-4.6）** — コンパクト、元データの①②③表記を保持：

```markdown
## 基本情報

| 項目 | 内容 |
|------|------|
| 工事名称 | ○○地区 新築工事 |
| 工事場所 | 東京都○○区○○町1-2-3 |
| 発注者 | 株式会社 ○○建設 |
| 施工者 | 株式会社 サンプル建設 |
| 工期 | 2025年4月1日 ～ 2026年3月31日 |
...

## 2. 施工注意事項

① 本仕様書は設計図書と合わせて使用すること。
② 材料の搬入前に監督員の承認を得ること。
```

**数量集計表（claude-sonnet-4.6）** — 数値列を右寄せ指定：

```markdown
| 工種 | 単位 | 設計数量 | 計画数量 | 実施数量 | 達成率(%) | 備考 |
|------|------|--------:|--------:|--------:|--------:|------|
| コンクリート打設 | m³ | 120 | 122 | 115 | 95.8 | |
```

#### 比較まとめ

| 観点 | gpt-4.1 | claude-sonnet-4.6 |
|------|---------|-------------------|
| 処理時間（3シート） | 約113秒 | **約56秒（約2倍速）** |
| テーブル整形 | 列幅スペース揃え | コンパクト |
| 数値列の右寄せ | なし | **あり（意味を理解）** |
| 元データの表記保持 | 変換（①→1.） | **保持（①のまま）** |
| 出力の安定性 | コードフェンス二重が発生 | 安定 |

処理速度・出力品質ともに **claude-sonnet-4.6 がやや優位** でした。どちらも50列繰り返し問題は解決しており、構造把握の正確さは同等です。

---

## ベンチマーク結果

5回計測（初回コールド＋4回ウォーム）の結果です。

| ツール | コールド（秒） | ウォーム平均（秒） | 備考 |
|--------|:---:|:---:|------|
| pandas | 0.139 | 0.096 | 最速 |
| markitdown | 0.300 | 0.265 | pandasベース |
| openpyxl | 0.367 | 0.348 | 結合セル展開あり |
| docling | 8.923 | 1.912 | 初回はモデルロードで遅い |
| GitHub Models (text) | — | — | 3〜5秒程度（3シート・参考値） |
| GitHub Copilot SDK (gpt-4.1) | — | — | 約113秒（3シート・参考値） |
| GitHub Copilot SDK (claude-sonnet-4.6) | — | — | 約56秒（3シート・参考値） |

:::message
doclingのコールドが遅いのは、初回実行時にHuggingFaceのモデルをローカルキャッシュに展開するためです。2回目以降はキャッシュから読まれ約4.7倍高速になります。

LLM系（05・06）の処理時間はネットワーク状態・トークン量・モデルの混雑状況によって大きく変動します。表の値は参考値としてご覧ください。
:::

---

## 各ツールの総合比較

| ツール | 方眼紙対応 | 50列問題 | 画像・図 | 処理速度 | 導入難度 | コスト |
|--------|:---:|:---:|:---:|:---:|:---:|:---:|
| openpyxl | △ | ❌ 残る | ❌ | ⚡ 速い | 低 | 無料 |
| pandas | ✗ NaN残 | ❌ 残る | ❌ | ⚡ 最速 | 低 | 無料 |
| markitdown | △ | ❌ 残る | ❌ | ⚡ 速い | 低 | 無料 |
| markitdown改善版 | ○ NaN解消 | ❌ 残る | ❌ | ⚡ 速い | 低 | 無料 |
| docling | △ | ❌ 残る | △ | 🐢 遅い | 高 | 無料 |
| docling改善版 | ○ | ❌ Markdown | ✅ HTML | 🐢 遅い | 高 | 無料 |
| GitHub Models (text) | ✅ | ✅ **解決** | △ 認識可 | ○ API依存 | 中 | 無料枠あり |
| GitHub Models (vision) | ✅ | ✅ **解決** | ✅ 認識可 | ○ API依存 | 中 | 無料枠あり |
| GitHub Copilot SDK (gpt-4.1) | ✅ | ✅ **解決** | △ 認識可 | 🐢 CLI起動込み | 中 | Copilotサブスク必要 |
| GitHub Copilot SDK (claude-sonnet-4.6) | ✅ | ✅ **解決** | △ 認識可 | 🐢 速め（gptの半分） | 中 | Copilotサブスク必要 |

---

## 結論

**方眼紙Excel → Markdown変換は、LLMを使うのが現実解。**

方眼紙Excelは「視覚レイアウト」で意味を表現する形式です。Markdownは「論理構造」で表現します。この2つはそもそも表現モデルが違うため、セル構造を機械的に変換しても意味のあるMarkdownにはなりません。LLMだけが意味構造を読み取って再構成できます。

### 用途別の最短ルート

| 目的 | 選択肢 |
|------|--------|
| とりあえず試したい | **pandas** または **markitdown**（秒単位・無料） |
| NaN / 空文字を減らしたい | **markitdown改善版**（openpyxlで事前展開） |
| セル結合を正確に保持したい | **docling改善版（HTML出力）** |
| 実務で使える品質のMarkdownが欲しい | **GitHub Models API（テキストモード）** |
| 画像・図面も含めて変換したい | **GitHub Models API（ビジョンモード）** |
| GitHub Copilotサブスクがある | **Copilot SDK（claude-sonnet-4.6）** |

### Copilot SDKの実用性について

Copilot SDKは複数モデルを切り替えられる柔軟さが魅力ですが、**処理時間（56〜113秒 / 3シート）と実装コスト**を考えると、現時点では GitHub Models API の方が実用的です。Copilotサブスクを既に持っており、エージェント的な処理を組み込みたい場合に有効な選択肢です。

### なぜLLMが勝つのか

従来ツールは「セル構造 → Markdownのセル構造」という変換をします。LLMは「セルの配置から意味を読み取り → 論理的なMarkdown構造として再構成」します。この違いがすべてです。方眼紙Excelにはセル結合で「タイトル」「見出し」「データ行」を視覚的に区別する情報が埋め込まれています。それを理解できるのは、今のところLLMだけです。

---

## 参考リンク

- [openpyxl ドキュメント](https://openpyxl.readthedocs.io/)
- [markitdown GitHub](https://github.com/microsoft/markitdown)
- [docling GitHub](https://github.com/DS4SD/docling)
- [GitHub Models](https://github.com/marketplace/models)
- [azure-ai-inference PyPI](https://pypi.org/project/azure-ai-inference/)
- [GitHub Copilot SDK 発表ブログ](https://github.blog/jp/2026-01-23-build-an-agent-into-any-app-with-the-github-copilot-sdk/)
- [github-copilot-sdk PyPI](https://pypi.org/project/github-copilot-sdk/)
- [Excel 方眼紙を GitHub Copilot に食わせてみた（参考）](https://zenn.dev/microsoft/articles/github-copilot-excel-hoganshi)
- [本記事のソースコード](https://github.com/ougotti/houganshi-excel-to-markdown)
