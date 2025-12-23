# gemini-chat-log

from google import genai
import streamlit as st
from datetime import datetime
import json
from pathlib import Path
import time

# =====================
# 設定
# =====================
client = genai.Client(api_key=st.secrets["GEMINI_API_KEY"])

LOG_FILE = Path("gemini_chat_log.jsonl")

def save_log(entry: dict):
    """1行1JSONでログ保存"""
    with LOG_FILE.open("a", encoding="utf-8") as f:
        f.write(json.dumps(entry, ensure_ascii=False) + "\n")

# =====================
# Streamlit UI設定
# =====================
st.set_page_config(
    page_title="My Gemini Log",
    page_icon="🤖",
    layout="centered"
)

st.title("Gemini Chat (Personal Log)")

# =====================
# セッション初期化
# =====================
if "messages" not in st.session_state:
    st.session_state.messages = []

if "busy" not in st.session_state:
    st.session_state.busy = False

# =====================
# チャット履歴表示
# =====================
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# =====================
# 入力欄
# =====================
prompt = st.chat_input(
    "Geminiに聞く…",
    disabled=st.session_state.busy
)

# =====================
# 送信処理
# =====================
if prompt and not st.session_state.busy:
    st.session_state.busy = True

    # ユーザー発話
    user_entry = {
        "timestamp": datetime.now().isoformat(),
        "role": "user",
        "content": prompt
    }
    st.session_state.messages.append(user_entry)
    save_log(user_entry)

    with st.chat_message("user"):
        st.markdown(prompt)

    # Gemini応答
    with st.spinner("Geminiが考えています…"):
        for attempt in range(3):
            try:
                response = client.models.generate_content(
                    model="gemini-2.0-flash",
                    contents=prompt
                )
                answer = response.text
                break
            except Exception as e:
                if "429" in str(e) and attempt < 2:
                    time.sleep(10)
                else:
                    answer = f"エラーが発生しました: {e}"
                    break

    assistant_entry = {
        "timestamp": datetime.now().isoformat(),
        "role": "assistant",
        "content": answer
    }
    st.session_state.messages.append(assistant_entry)
    save_log(assistant_entry)

    with st.chat_message("assistant"):
        st.markdown(answer)

    st.session_state.busy = False
