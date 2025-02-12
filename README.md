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
