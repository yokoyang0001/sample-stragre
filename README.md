| 行                         | 役割                                   |
| ------------------------- | ------------------------------------ |
| `reasoning_effort="high"` | 内部 CoT を長めに確保し精度を上げる([note（ノート）][1]) |
| `stream=True`             | トークンを逐次 UI に流し“考え中”を可視化              |
| `## 結論 ##` ラベル            | 正規表現で**最終回答だけ**抜き出せる                 |

[1]: https://note.com/geeorgey/n/nb6c609a9efcb?utm_source=chatgpt.com "OpenAI API o1のReasoning Effort low/medium/highの違い ..."
