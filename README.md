# README.md








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
  }' \


