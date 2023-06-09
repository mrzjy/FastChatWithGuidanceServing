# OpenAI-Compatible RESTful APIs & SDK

FastChat provides OpenAI-compatible APIs for its supported models, so you can use FastChat as a local drop-in replacement for OpenAI APIs.
The FastChat server is compatible with both [openai-python](https://github.com/openai/openai-python) library and cURL commands.

The following OpenAI APIs are supported:
- Chat Completions. (Reference: https://platform.openai.com/docs/api-reference/chat)
- Completions. (Reference: https://platform.openai.com/docs/api-reference/completions)
- Embeddings. (Reference: https://platform.openai.com/docs/api-reference/embeddings)

The following Guidance API is supported:
- Guidance Completions. (Similar to Completions API, but with [guidance](https://github.com/microsoft/guidance) support) 

## RESTful API Server
First, launch the controller

```bash
python3 -m fastchat.serve.controller
```

Then, launch the model worker(s)

```bash
python3 -m fastchat.serve.model_worker --model-name 'vicuna-7b-v1.1' --model-path /path/to/vicuna/weights
```

Finally, launch the RESTful API server

```bash
python3 -m fastchat.serve.openai_api_server --host localhost --port 8000
```

Now, let us test the API server.

### Guidance Usage for Local models

```python
import requests
import json
url = "http://localhost:8000/v1/guidance/completions"
headers = {'Content-Type': 'application/json'}

prompt = """USER: On the scale of 1 to 10, where 1 is purely mundane (e.g., brushing teeth, making bed) and 10 is extremely poignant (e.g., a break up, college acceptance), rate the likely poignancy of the following pieces of memory
ASSISTANT: Got it. I will answer with an integer rating</s>
USER: Memory: John's best friend got accepted by New York University
ASSISTANT: Rating: {{#select 'rating'}}1{{or}}2{{or}}3{{or}}4{{or}}5{{or}}6{{or}}7{{or}}8{{or}}9{{or}}10{{/select}}"""
input_dict = {
    "prompt": prompt,
    "arguments": {}
}
data = {
    "model": "vicuna-7b-v1.1",
    "prompt": json.dumps(input_dict, ensure_ascii=False)
}
res = requests.post(url, headers=headers, json=data)
text = res.json()["choices"][0]["text"]
print(json.loads(text))
```

One should be able to get something like the following from the above code. Note how guidance output variables are saved in the choice's "text" field

```json
{
   "id":"cmpl-cujWKUP8jHJ6mh6GZWS4Gg",
   "object":"text_completion",
   "created":1686294422,
   "model":"vicuna-7b-v1.1",
   "choices":[
      {
         "index":0,
         "text":"{\"rating\": \" 8\"}",
         "logprobs":"None",
         "finish_reason":"stop"
      }
   ],
   "usage":{
      "prompt_tokens":158,
      "total_tokens":160,
      "completion_tokens":2
   }
}
```

### OpenAI Official SDK
The goal of `openai_api_server.py` is to implement a fully OpenAI-compatible API server, so the models can be used directly with [openai-python](https://github.com/openai/openai-python) library.

First, install openai-python:
```bash
pip install --upgrade openai
```

Then, interact with model vicuna:
```python
import openai
openai.api_key = "EMPTY" # Not support yet
openai.api_base = "http://localhost:8000/v1"

model = "vicuna-7b-v1.1"
prompt = "Once upon a time"

# create a completion
completion = openai.Completion.create(model=model, prompt=prompt, max_tokens=64)
# print the completion
print(prompt + completion.choices[0].text)

# create a chat completion
completion = openai.ChatCompletion.create(
  model=model,
  messages=[{"role": "user", "content": "Hello! What is your name?"}]
)
# print the completion
print(completion.choices[0].message.content)
```

Streaming is also supported. See [test_openai_sdk.py](../tests/test_openai_sdk.py).

### cURL
cURL is another good tool for observing the output of the api.

List Models:
```bash
curl http://localhost:8000/v1/models
```

Chat Completions:
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vicuna-7b-v1.1",
    "messages": [{"role": "user", "content": "Hello! What is your name?"}]
  }'
```

Text Completions:
```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vicuna-7b-v1.1",
    "prompt": "Once upon a time",
    "max_tokens": 41,
    "temperature": 0.5
  }'
```

Embeddings:
```bash
curl http://localhost:8000/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "vicuna-7b-v1.1",
    "input": "Hello world!"
  }'
```

## LangChain Support
This OpenAI-compatible API server supports LangChain. See [LangChain Integration](langchain_integration.md) for details.

## Adjusting Timeout
By default, a timeout error will occur if a model worker does not response within 20 seconds. If your model/hardware is slower, you can change this timeout through an environment variable: `export FASTCHAT_WORKER_API_TIMEOUT=<larger timeout in seconds>`

## Todos
Some features to be implemented:

- [ ] Support more parameters like `logprobs`, `logit_bias`, `user`, `presence_penalty` and `frequency_penalty`
- [ ] Model details (permissions, owner and create time)
- [ ] Edits API
- [ ] Authentication and API key
- [ ] Rate Limitation Settings
