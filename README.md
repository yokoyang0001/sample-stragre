from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
import re

# --- モデル設定 -------------------------------------------------
llm_o1 = ChatOpenAI(
    model            = "o1",
    temperature      = 0.2,
    max_tokens       = 2048,
    reasoning_effort = "high",   # ★推論型専用オプション
    stream           = True
)

# --- プロンプト -------------------------------------------------
prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "あなたは熟練研究者です。必ず「Step 1:」「Step 2:」… と推論し、最後に「## 結論 ##」で 2 行以内にまとめてください。"),
        ("user",   "{question}")
    ]
)

# --- 実行 ------------------------------------------------------
chain = prompt | llm_o1
full_text = "".join(chunk.content for chunk in chain.stream({"question": "火星で持続的に生活するため最優先の技術課題は？"}))

# --- “最終的な回答”だけ抽出 ------------------------------
match = re.search(r"## 結論 ##\s*(.*)", full_text, re.DOTALL)
final_answer = match.group(1).strip() if match else full_text

print("--- 推論過程 ---")
print(full_text[:500], "...")
print("\n--- 最終的な回答 ---")
print(final_answer)
