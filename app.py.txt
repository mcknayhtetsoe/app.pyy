import streamlit as st
import pandas as pd
import pandas_ta as ta
from binance.client import Client
import plotly.graph_objects as go

# Layout configuration
st.set_page_config(page_title="P'Ta AI Trading Dashboard", layout="wide")

# Binance Client setup (API Key မလိုပါ - Public Data သုံးထားသည်)
client = Client()

st.title("🚀 BTC/USD AI Live Market Analyzer")
st.write("Real-time ဈေးကွက်ကို AI နဲ့ သုံးသပ်ပြီး ဝယ်သင့်/ရောင်းသင့် အကြံပြုပေးသော Website")

# Sidebar - ရွေးချယ်စရာများ
st.sidebar.header("Settings")
symbol = st.sidebar.selectbox("Choose Crypto", ["BTCUSDT", "ETHUSDT", "SOLUSDT"])
timeframe = st.sidebar.selectbox("Timeframe", ["1h", "4h", "1d", "15m"])

# --- ၁။ Data ဆွဲယူခြင်း ---
@st.cache_data(ttl=60) # 60 စက္ကန့်တိုင်း refresh လုပ်မည်
def get_data(symbol, interval):
    bars = client.get_klines(symbol=symbol, interval=interval, limit=100)
    df = pd.DataFrame(bars, columns=['Time', 'Open', 'High', 'Low', 'Close', 'Volume', 'CloseTime', 'QuoteAssetVol', 'Trades', 'TakerBuyBase', 'TakerBuyQuote', 'Ignore'])
    df['Time'] = pd.to_datetime(df['Time'], unit='ms')
    for col in ['Open', 'High', 'Low', 'Close', 'Volume']:
        df[col] = df[col].astype(float)
    return df

df = get_data(symbol, timeframe)

# --- ၂။ AI Logic (Technical Analysis) ---
df['RSI'] = ta.rsi(df['Close'], length=14)
df['EMA_20'] = ta.ema(df['Close'], length=20)
df['EMA_50'] = ta.ema(df['Close'], length=50)

last_price = df['Close'].iloc[-1]
last_rsi = df['RSI'].iloc[-1]
ema_20 = df['EMA_20'].iloc[-1]
ema_50 = df['EMA_50'].iloc[-1]

# --- ၃။ AI Decision Engine ---
score = 0
reasons = []

if last_rsi < 35: 
    score += 40
    reasons.append("RSI က ဈေးအရမ်းကျနေလို့ ဝယ်ဖို့ကောင်းနေပြီ (Oversold)")
elif last_rsi > 65:
    score -= 40
    reasons.append("RSI က ဈေးအရမ်းတက်နေလို့ ပြန်ကျနိုင်တယ် (Overbought)")

if last_price > ema_20:
    score += 20
    reasons.append("ဈေးနှုန်းက EMA 20 အထက်မှာရှိလို့ Trend ကောင်းနေတယ်")
else:
    score -= 20
    reasons.append("ဈေးနှုန်းက EMA 20 အောက်ရောက်နေလို့ သတိထားပါ")

# Decision Logic
if score >= 30:
    decision = "✅ BUY (ဝယ်ရန် အချက်ပြနေသည်)"
    color = "#00ff88"
elif score <= -30:
    decision = "❌ SELL (ရောင်းရန်/ရှောင်ရန် အချက်ပြနေသည်)"
    color = "#ff3366"
else:
    decision = "⚖️ NEUTRAL (စောင့်ကြည့်ပါ)"
    color = "#f59e0b"

# --- ၄။ Dashboard UI ပြသခြင်း ---
col1, col2, col3 = st.columns(3)
col1.metric("Current Price", f"${last_price:,.2f}")
col2.metric("RSI (14)", f"{last_rsi:.2f}")
col3.metric("AI Sentiment Score", f"{score}%")

st.markdown(f"### AI Recommendation: <span style='color:{color}'>{decision}</span>", unsafe_allow_html=True)
st.info("AI Analysis: " + " | ".join(reasons))

# Chart ပြသခြင်း
fig = go.Figure(data=[go.Candlestick(x=df['Time'], open=df['Open'], high=df['High'], low=df['Low'], close=df['Close'], name='Market Data')])
fig.add_trace(go.Scatter(x=df['Time'], y=df['EMA_20'], line=dict(color='orange', width=1), name='EMA 20'))
fig.update_layout(template="plotly_dark", xaxis_rangeslider_visible=False)
st.plotly_chart(fig, use_container_width=True)

