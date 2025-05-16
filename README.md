from typing import Generator, Optional
import time
import logging
import os
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
# from vertexai.preview.tokenization import get_tokenizer_for_model
from langchain_google_vertexai import ChatVertexAI
import google.generativeai as genai
from promptflow.tracing import start_trace, trace


logging.basicConfig(level=logging.DEBUG)

start_trace()

DEFAULT_MESSAGE = "Geminiについて500字で教えて"
MODEL = "gemini-2.0-flash"

os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = "xxxx"

start_time = time.time()
model = ChatVertexAI(
    model_name=MODEL,
    api_transport="rest",
    location="asia-northeast1",
    streaming=True,
    api_endpoint="us-central1-aiplatform.googleapis.com"
)

def call_stream(_model):
    for i, chunk in enumerate(_model.stream("Geminiについて500字で教えて"), start=1):
        print(f"[チャンク1 {i}: {chunk.content}")


def compression_rate(model_name: str, text: str) -> float:
    model = genai.GenerativeModel(MODEL)
    token_count = model.count_tokens(text).total_tokens
    return len(text) / token_count

# call_stream(model)
try:
   sample_text = "昨日は渋谷で打ち合わせがありました。" + "来週のプロジェクトのスコープについて詳細を詰めています。"
   rate_15 = compression_rate("gemini1.5-pro-001", sample_text)
   rate_20 = compression_rate("gemini2.0-flash", sample_text)
   print(f"Gemini1.5-Pro の圧縮率: {rate_15:.2f} 文字/トークン")
   print(f"Gemini2.0-Flash の圧縮率: {rate_20:.2f} 文字/トークン")

except Exception as e:
    print(f"トークンカウント中にエラーが発生しました: {e}")
    print("サービスアカウントが正しく設定されているか、必要な権限があるか確認してください。")
    print("モデル名が正しいか、Vertex AI または Generative AI API で利用可能か確認してください。")

# tokenizer = get_tokenizer_for_model("gemini-1.5-pro-001")
# print(tokenizer.count_tokens(text).total_tokens)

# response = model.invoke("Geminiについて500字で教えて")
# print(response.content)

