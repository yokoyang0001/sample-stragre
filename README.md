# README.md




```python
import requests
import google.auth
from google.auth.transport.requests import Request
import json
import time

# 認証トークンの取得
credentials, _ = google.auth.default()
credentials.refresh(Request())
access_token = credentials.token

# Vertex AI エンドポイント
PROJECT_ID = "your-project-id"
LOCATION = "us-central1"
MODEL_ID = "gemini-pro"
ENDPOINT = f"https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/{LOCATION}/publishers/google/models/{MODEL_ID}:streamGenerateContent"

# ヘッダー
headers = {
    "Authorization": f"Bearer {access_token}",
    "Content-Type": "application/json",
    "Accept": "text/event-stream"
}

# リクエストボディ
data = {
    "contents": [
        {"role": "user", "parts": [{"text": "Tell me about machine learning."}]}
    ],
    "generationConfig": {
        "temperature": 0.7,
        "top_p": 0.9,
        "max_output_tokens": 1024
    }
}

# 最大リトライ回数
MAX_RETRIES = 5

def convert_messages_to_vertexai(messages):
    converted = []
    for message in messages:
        if isinstance(message, SystemMessage):
            converted.append({"role": "user", "parts": [{"text": message.content}]})  # SystemMessage も "user"
        elif isinstance(message, HumanMessage):
            converted.append({"role": "user", "parts": [{"text": message.content}]})
        elif isinstance(message, AIMessage):
            converted.append({"role": "model", "parts": [{"text": message.content}]})
    return converted

def call_gemini(retries=0):
    try:
        response = requests.post(ENDPOINT, headers=headers, json=data, stream=True, timeout=15)

        # HTTP ステータスコードによるエラーハンドリング
        if response.status_code == 400:
            raise ValueError("400 Bad Request: リクエストデータを確認してください")
        elif response.status_code == 401:
            raise PermissionError("401 Unauthorized: 認証トークンが無効または期限切れです")
        elif response.status_code == 403:
            raise PermissionError("403 Forbidden: 権限が不足しています")
        elif response.status_code == 404:
            raise FileNotFoundError("404 Not Found: エンドポイント URL が間違っています")
        elif response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 5))
            print(f"429 Too Many Requests: {retry_after}秒後に再試行します")
            time.sleep(retry_after)
            return call_gemini(retries + 1)  # 再試行
        elif response.status_code in [500, 503, 504]:
            if retries < MAX_RETRIES:
                wait_time = 2 ** retries  # 指数バックオフ
                print(f"{response.status_code} エラー: {wait_time}秒後に再試行（{retries+1}/{MAX_RETRIES}）")
                time.sleep(wait_time)
                return call_gemini(retries + 1)  # 再試行
            else:
                raise ConnectionError(f"{response.status_code} サーバーエラー: 最大リトライ回数を超えました")

        # ストリーミングレスポンスの処理
        for line in response.iter_lines():
            if line:
                try:
                    event_data = json.loads(line.decode("utf-8").replace("data: ", ""))
                    print(event_data)  # ストリーミングデータ
                except json.JSONDecodeError:
                    print("JSON デコードエラー:", line.decode("utf-8"))

    except requests.exceptions.Timeout:
        if retries < MAX_RETRIES:
            wait_time = 2 ** retries
            print(f"タイムアウト発生。{wait_time}秒後に再試行（{retries+1}/{MAX_RETRIES}）")
            time.sleep(wait_time)
            return call_gemini(retries + 1)  # 再試行
        else:
            print("タイムアウトが続いたため、処理を中断します")

    except requests.exceptions.ConnectionError:
        print("ネットワークエラー: サーバーに接続できません")
    except requests.exceptions.RequestException as e:
        print(f"リクエストエラー: {e}")
    except Exception as e:
        print(f"予期しないエラー: {e}")

# 実行
call_gemini()

```



```
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{
    "contents": [
      {"role": "user", "parts": [{"text": "Tell me a long story."}]}
    ],
    "generationConfig": {
      "temperature": 0.7,
      "top_p": 0.9,
      "max_output_tokens": 1024
    }
  }' 
```


```
import json
import time
import requests
import google.auth
from google.auth.transport.requests import Request

# 🔹 Google 認証情報の取得
credentials, project = google.auth.default()
credentials.refresh(Request())

# 🔹 Vertex AI のエンドポイント & モデル
LOCATION = "asia-northeast1"
MODEL_ID = "gemini-pro"
ENDPOINT = f"https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{project}/locations/{LOCATION}/publishers/google/models/{MODEL_ID}:streamGenerateContent"

# 🔹 API ヘッダー
headers = {
    "Authorization": f"Bearer {credentials.token}",
    "Content-Type": "application/json"
}

# 🔹 リクエストデータ
data = {
    "model": MODEL_ID,
    "contents": [{"role": "user", "parts": [{"text": "Geminiについて500字で教えて"}]}]
}

# 🔹 `CallbackManager` クラス（イベントごとに専用の `execute_*` メソッドを作成）
class CallbackManager:
    def execute_start(self):
        """リクエスト開始時の処理"""
        print("[INFO] リクエスト開始")

    def execute_new_token(self, chunk, elapsed_time):
        """LLM から新しいトークンを受信したときの処理"""
        try:
            decoded_chunk = json.loads(chunk.decode("utf-8"))
            token = decoded_chunk.get("text", "")  # 受信したテキスト部分
            print(f"[{elapsed_time:.2f}s] 受信トークン: {token}")
        except json.JSONDecodeError:
            self.execute_llm_error("無効なトークンを受信しました")

    def execute_end(self):
        """リクエスト完了時の処理"""
        print("[INFO] リクエスト終了")

    def execute_llm_error(self, error):
        """LLM からのレスポンスエラー"""
        print(f"[LLM ERROR] {error}")

    def execute_chain_error(self, error):
        """API 呼び出し全体のエラー"""
        print(f"[CHAIN ERROR] {error}")

# 🔹 `CallbackManager` のインスタンス作成
callback_manager = CallbackManager()

# 🔹 API リクエスト送信（ストリーミング対応）
start_time = time.time()

try:
    callback_manager.execute_start()  # リクエスト開始

    with requests.post(ENDPOINT, headers=headers, json=data, stream=True) as response:
        if response.status_code != 200:
            raise Exception(f"APIエラー: {response.status_code} {response.text}")

        for chunk in response.iter_lines():
            if chunk:
                elapsed_time = time.time() - start_time
                callback_manager.execute_new_token(chunk, elapsed_time)  # トークン受信

    callback_manager.execute_end()  # リクエスト終了

except requests.exceptions.RequestException as e:
    callback_manager.execute_chain_error(str(e))
except Exception as e:
    callback_manager.execute_chain_error(str(e))

```
