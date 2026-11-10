"""
プライム市場「窓を開けて下落した銘柄」をDiscordに通知するスクリプト

判定ロジック（デフォルト）
  ・当日始値 < 前日安値  （＝ギャップダウン）
  ・窓幅 = (前日安値 - 当日始値) / 前日安値 が GAP_MIN_PCT 以上
  ・売買代金が MIN_TRADING_VALUE_OKU 億円以上（低流動性銘柄を除外）
  ・REQUIRE_UNFILLED=True なら「当日高値 < 前日安値（窓を埋めずに引けた）」のみ

必要なもの
  pip install yfinance pandas requests xlrd
  環境変数 DISCORD_WEBHOOK_URL に Webhook URL を設定
"""

import os
import sys
import time
from datetime import datetime, timedelta, timezone

import pandas as pd
import requests
import yfinance as yf

# ===================== 設定 =====================
WEBHOOK_URL = os.environ.get("DISCORD_WEBHOOK_URL", "")

GAP_MIN_PCT = 1.0            # 窓幅の下限(%)。前日安値に対して始値が何%下か
REQUIRE_UNFILLED = False     # True: 当日高値が前日安値に届かず「窓を開けたまま」の銘柄だけ
MIN_TRADING_VALUE_OKU = 1.0  # 当日売買代金の下限(億円)。0で無効
MAX_NOTIFY = 30              # 通知する最大件数（窓幅の大きい順）
CHUNK_SIZE = 200             # yfinance 一括取得の銘柄数
FORCE = os.environ.get("FORCE", "0") == "1"  # 1なら休場日チェックを無視

# 銘柄名タップ時のリンク先（{code} が銘柄コードに置き換わる）
LINK_TEMPLATE = "https://finance.yahoo.co.jp/quote/{code}.T"
# 株探にしたい場合 → "https://kabutan.jp/stock/chart?code={code}"

JPX_URL = ("https://www.jpx.co.jp/markets/statistics-equities/misc/"
           "tvdivq0000001vg2-att/data_j.xls")
EMBED_COLOR = 0xE74C3C       # 赤
JST = timezone(timedelta(hours=9))
# ================================================


def load_prime_list() -> pd.DataFrame:
    """JPXの上場銘柄一覧からプライム（内国株式）を取得"""
    df = pd.read_excel(JPX_URL, dtype={"コード": str})
    df = df[df["市場・商品区分"].astype(str).str.contains("プライム")]
    df = df.rename(columns={"コード": "code", "銘柄名": "name", "33業種区分": "sector"})
    df["code"] = df["code"].astype(str).str.strip()
    return df[["code", "name", "sector"]].reset_index(drop=True)


def download_prices(codes: list[str]) -> dict[str, pd.DataFrame]:
    """1年分の日足をまとめて取得 → {code: DataFrame}"""
    result = {}
    for i in range(0, len(codes), CHUNK_SIZE):
        chunk = codes[i:i + CHUNK_SIZE]
        tickers = [f"{c}.T" for c in chunk]
        print(f"  取得中 {i + 1}-{i + len(chunk)} / {len(codes)}")
        try:
            data = yf.download(tickers, period="1y", interval="1d",
                               group_by="ticker", auto_adjust=False,
                               threads=True, progress=False)
        except Exception as e:
            print(f"  ! 取得失敗: {e}")
            continue
        for c, t in zip(chunk, tickers):
            try:
                df = data[t] if len(tickers) > 1 else data
                df = df.dropna(subset=["Open", "High", "Low", "Close"])
                if len(df) >= 30:
                    result[c] = df
            except KeyError:
                pass
        time.sleep(1)
    return result


def rsi(close: pd.Series, period: int = 14) -> float:
    diff = close.diff()
    up = diff.clip(lower=0).ewm(alpha=1 / period, adjust=False).mean()
    down = (-diff.clip(upper=0)).ewm(alpha=1 / period, adjust=False).mean()
    rs = up.iloc[-1] / down.iloc[-1] if down.iloc[-1] != 0 else float("inf")
    return 100 - 100 / (1 + rs)


def analyze(code: str, df: pd.DataFrame) -> dict | None:
    today, prev = df.iloc[-1], df.iloc[-2]
    o, h, l, c, v = today["Open"], today["High"], today["Low"], today["Close"], today["Volume"]
    prev_low, prev_close = prev["Low"], prev["Close"]

    if o >= prev_low:
        return None
    gap_pct = (prev_low - o) / prev_low * 100
    if gap_pct < GAP_MIN_PCT:
        return None
    unfilled = h < prev_low
    if REQUIRE_UNFILLED and not unfilled:
        return None

    trading_value_oku = c * v / 1e8
    if trading_value_oku < MIN_TRADING_VALUE_OKU:
        return None

    vol_avg20 = df["Volume"].iloc[-21:-1].mean()
    ma25 = df["Close"].iloc[-25:].mean()
    high52 = df["High"].max()

    return {
        "code": code,
        "date": df.index[-1].strftime("%Y-%m-%d"),
        "close": c, "open": o, "high": h, "low": l,
        "prev_low": prev_low, "prev_close": prev_close,
        "change_pct": (c - prev_close) / prev_close * 100,
        "gap_pct": gap_pct,
        "unfilled": unfilled,
        "to_fill_pct": (prev_low - c) / c * 100,       # 終値から窓上限(前日安値)まで
        "vol_ratio": v / vol_avg20 if vol_avg20 else None,
        "trading_value_oku": trading_value_oku,
        "ma25_dev": (c - ma25) / ma25 * 100,
        "rsi": rsi(df["Close"]),
        "from_high52": (c - high52) / high52 * 100,
    }


def get_market_cap_oku(code: str) -> float | None:
    try:
        mc = yf.Ticker(f"{code}.T").fast_info["market_cap"]
        return mc / 1e8 if mc else None
    except Exception:
        return None


def fmt_price(x: float) -> str:
    return f"¥{x:,.1f}"


def build_embed(rank: int, r: dict) -> dict:
    mc = r.get("market_cap_oku")
    mc_txt = f"({mc:,.0f}億円)" if mc else ""
    vol_txt = f"{r['vol_ratio']:.2f}倍" if r["vol_ratio"] else "—"
    fill_txt = (f"🕳️ 窓 **未埋め**（窓上限まで +{r['to_fill_pct']:.1f}%）"
                if r["unfilled"] else "🩹 窓 ザラ場で一部/全部埋め")

    lines = [
        f"`{r['code']}` {r['sector']}",
        f"💰 **{fmt_price(r['close'])}** {mc_txt}  📉 前日比 **{r['change_pct']:+.2f}%**  "
        f"📈 出来高 **{vol_txt}**",
        f"🕳️ 窓幅 **-{r['gap_pct']:.2f}%**　前日安値 {fmt_price(r['prev_low'])} → 始値 {fmt_price(r['open'])}",
        f"📊 高値 {fmt_price(r['high'])} / 安値 {fmt_price(r['low'])}　💴 売買代金 {r['trading_value_oku']:,.1f}億円",
        f"📏 25日線乖離 {r['ma25_dev']:+.1f}%　RSI {r['rsi']:.1f}　52週高値比 {r['from_high52']:+.1f}%",
        fill_txt,
    ]
    return {
        "title": f"{rank}. {r['name']}",
        "url": LINK_TEMPLATE.format(code=r["code"]),
        "description": "\n".join(lines),
        "color": EMBED_COLOR,
    }


def post_discord(payload: dict) -> None:
    for _ in range(3):
        res = requests.post(WEBHOOK_URL, json=payload, timeout=30)
        if res.status_code == 429:
            time.sleep(float(res.json().get("retry_after", 2)))
            continue
        if res.status_code >= 400:
            print(f"  ! Discordエラー {res.status_code}: {res.text}")
        return


def notify(results: list[dict], date_str: str) -> None:
    if not results:
        post_discord({"content": f"📉 **{date_str} 窓開け下落銘柄**\n該当なし"})
        return

    header = (f"📉 **{date_str} 窓開け下落銘柄（プライム）** {len(results)}件\n"
              f"条件: 窓幅 {GAP_MIN_PCT}%以上 / 売買代金 {MIN_TRADING_VALUE_OKU}億円以上"
              f"{' / 窓未埋めのみ' if REQUIRE_UNFILLED else ''}")
    embeds = [build_embed(i + 1, r) for i, r in enumerate(results)]

    # Discordは1メッセージ10embedまで
    for i in range(0, len(embeds), 10):
        payload = {"embeds": embeds[i:i + 10]}
        if i == 0:
            payload["content"] = header
        post_discord(payload)
        time.sleep(1)


def main() -> None:
    if not WEBHOOK_URL:
        sys.exit("DISCORD_WEBHOOK_URL が設定されていません")

    today_jst = datetime.now(JST).strftime("%Y-%m-%d")
    print("プライム銘柄一覧を取得中...")
    prime = load_prime_list()
    print(f"  {len(prime)} 銘柄")
    info = prime.set_index("code").to_dict("index")

    print("株価を取得中...")
    prices = download_prices(prime["code"].tolist())
    if not prices:
        sys.exit("株価が取得できませんでした")

    # 休場日チェック（最新データが今日でなければ終了）
    latest = max(df.index[-1] for df in prices.values()).strftime("%Y-%m-%d")
    if latest != today_jst and not FORCE:
        print(f"最新データ {latest} が今日 {today_jst} ではないため終了（休場日？）")
        return

    print("判定中...")
    results = []
    for code, df in prices.items():
        if df.index[-1].strftime("%Y-%m-%d") != latest:
            continue  # 当日売買のない銘柄は除外
        r = analyze(code, df)
        if r:
            r.update(info.get(code, {"name": code, "sector": ""}))
            results.append(r)

    results.sort(key=lambda x: x["gap_pct"], reverse=True)
    results = results[:MAX_NOTIFY]
    print(f"  該当 {len(results)} 件 → 時価総額取得中...")
    for r in results:
        r["market_cap_oku"] = get_market_cap_oku(r["code"])

    notify(results, latest)
    print("完了")


if __name__ == "__main__":
    main()
