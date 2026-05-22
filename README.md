<h3>2. 完整 Python 程式碼 (app.py)</h3>
<p>請在你的電腦裡建立一個名為 <code>app.py</code> 的檔案，並將以下程式碼完整複製貼入：</p>

<pre style="background-color: #f4f4f4; border: 1px solid #ddd; border-left: 3px solid #f36d33; color: #333; page-break-inside: avoid; font-family: monospace; font-size: 15px; line-height: 1.6; margin-bottom: 1.6em; max-width: 100%; overflow: auto; padding: 1em 1.5em; display: block; word-wrap: break-word; white-space: pre;">
import streamlit as st
import tempfile
import os
from google import genai
from google.genai import types

# 網頁標題與設定
st.set_page_config(page_title="羽球賽後檢討 AI 教練", layout="centered")
st.title("🏸 羽球賽後檢討 AI 教練")
st.write("上傳你的比賽心得、數據檔案或影片，讓 AI 教練幫你精準分析！")

# 讓使用者在網頁上輸入自己的 Gemini API Key
api_key = st.sidebar.text_input("輸入你的 Gemini API Key", type="password")

if api_key:
    # 初始化 Gemini 客戶端
    client = genai.Client(api_key=api_key)

    # 設定 AI 的角色教練指令
    system_instruction = """
    你是一位擁有10年執教經驗的專業羽球總教練。
    請分析使用者提供的比賽心得、數據檔案或影片，並從以下三個方面給予結構化的賽後檢討：
    1. 【技術面分析】：擊球點、球路質量、動作問題（如果有影片）。
    2. 【戰術面分析】：多拍來回時的策略、防守或跑位盲點。
    3. 【下週練習計畫】：提供 2 個具體、可執行的自主訓練菜單。
    
    語氣請保持專業、嚴謹但充滿鼓勵。
    """

    # 1. 文字輸入區
    user_notes = st.text_area(
        "請輸入這場比賽的心得或比分（例如：21:18 輸了，一直被調動後場...）", 
        height=150
    )

    # 2. 檔案與影片上傳區
    uploaded_file = st.file_uploader(
        "選擇要上傳的檔案或影片（支援 .txt, .csv 或 .mp4 影片）", 
        type=["txt", "csv", "mp4"]
    )

    # 按下送出按鈕
    if st.button("讓教練開始檢討"):
        contents = [user_notes] if user_notes else []

        # 處理上傳的檔案/影片
        if uploaded_file is not None:
            with st.spinner("正在將檔案安全地提供給 AI 教練..."):
                
                # 如果是影片檔案，使用專用的連線傳輸（File API）
                if uploaded_file.type == "video/mp4":
                    with tempfile.NamedTemporaryFile(delete=False, suffix=".mp4") as tmp_file:
                        tmp_file.write(uploaded_file.read())
                        tmp_path = tmp_file.name

                    uploaded_video_file = client.files.upload(file=tmp_path)
                    contents.append(uploaded_video_file)
                    os.unlink(tmp_path)
                
                # 如果是純文字或 CSV 檔案
                else:
                    file_bytes = uploaded_file.read()
                    mime_type = uploaded_file.type
                    file_part = types.Part.from_bytes(
                        data=file_bytes,
                        mime_type=mime_type,
                    )
                    contents.append(file_part)

        if not contents:
            st.warning("請至少輸入心得或上傳一個檔案喔！")
        else:
            with st.spinner("AI 教練正在看影片和思考分析中，請稍候... (影片較大時需要稍等一下下)"):
                try:
                    # 呼叫 Gemini 2.5 Flash 模型
                    response = client.models.generate_content(
                        model='gemini-2.5-flash',
                        contents=contents,
                        config=types.GenerateContentConfig(
                            system_instruction=system_instruction,
                            temperature=0.4,
                        )
                    )
                    
                    # 顯示教練的檢討報告
                    st.success("📊 AI 教練檢討報告產出完畢！")
                    st.markdown(response.text)
                    
                except Exception as e:
                    st.error(f"發生錯誤：{e}")
else:
    st.info("請先在左側邊欄輸入你的 Gemini API Key 以啟動 AI 教練。")
</pre>
