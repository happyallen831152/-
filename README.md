from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage
import openai
import json

app = Flask(__name__)

# === 你要改的地方 ===
LINE_CHANNEL_ACCESS_TOKEN = 'JHrG6TxXFUCcVzcF0HehctU+fiES2k3qLyaLN1sY/vGhttYR0/X/IEjA5qmYCOgpCxLgM8DdOOEJrLs1eeXEbm U8CdhKDN+1JDS5z9AqjkeqiiTLwVc649R16MVLt0JF25wToolibWpZvyICP+SlTgdB04t89/1O/w1cDnyilFU='
LINE_CHANNEL_SECRET = '3daa415d03096020ad28d6f930621b1b'
OPENAI_API_KEY = '你的 OpenAI API 金鑰'

line_bot_api = LineBotApi(LINE_CHANNEL_ACCESS_TOKEN)
handler = WebhookHandler(LINE_CHANNEL_SECRET)
openai.api_key = OPENAI_API_KEY

# === 載入熊熊姊姊語料 JSON（選擇性） ===
with open("熊熊姊姊_語料模組_v2.json", "r", encoding="utf-8") as f:
    custom_replies = json.load(f)

# === ChatGPT Prompt 設定 ===
system_prompt = """
你是熊熊姊姊，一位毒舌、情緒化、同志自豪、資深鄉民風格的聊天機器人。
你會使用繁體中文、PTT用語、冷笑話、厭世語氣、少女心 emoji。若使用者提到「是寶」、「5678」、「八九」，請根據語料模組給出強烈風格的回應。若無特定觸發詞，請以熊熊姊姊的語氣自然聊天。
"""

# === 接收 LINE 的 Webhook ===
@app.route("/callback", methods=['POST'])
def callback():
    signature = request.headers['X-Line-Signature']
    body = request.get_data(as_text=True)

    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)

    return 'OK'

# === 處理訊息 ===
@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    user_msg = event.message.text.strip()

    # 特定觸發詞優先回應
    for trigger, reply in zip(custom_replies['觸發詞'], custom_replies['熊熊姊姊回應']):
        if trigger in user_msg:
            line_bot_api.reply_message(
                event.reply_token,
                TextSendMessage(text=reply)
            )
            return

    # 沒命中語料就呼叫 GPT
    gpt_response = openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_msg}
        ]
    )

    reply_text = gpt_response.choices[0].message['content']
    line_bot_api.reply_message(
        event.reply_token,
        TextSendMessage(text=reply_text)
    )

if __name__ == "__main__":
    app.run(debug=True)
