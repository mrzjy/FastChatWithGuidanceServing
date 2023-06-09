# FastChatWithGuidanceServing

(See [README_original.md](README_original.md) for the original FastChat Readme.)

This project derives from [FastChat](https://github.com/lm-sys/FastChat) with **enhanced serving features**:

Besides the original Openai APIs (v1/completion, v1/chat/completion), we also support [Guidance](https://github.com/microsoft/guidance) for serving (v1/guidance/completion), making the LLM generation much more controllable than just prompt engineering.

See [openai_api.md](docs/openai_api.md) for more details

### Install from source

Remember to install fschat from source

1. Clone this repository and navigate to the FastChat folder
```bash
git clone https://github.com/lm-sys/FastChat.git
cd FastChat
```

If you are running on Mac:
```bash
brew install rust cmake
```

2. Install Package
```bash
pip3 install --upgrade pip  # enable PEP 660 support
pip3 install -e .
```

## Guidance Usage for Local models

### Sample 1

The following test prompt derives from [Generative Agents](https://arxiv.org/pdf/2304.03442.pdf)

```python
import requests
import json

url = "http://localhost:8000/v1/guidance/completions"
headers = {'Content-Type': 'application/json'}

prompt = """USER: On the scale of 1 to 10, where 1 is purely mundane (e.g., brushing teeth, making bed) and 10 is extremely poignant (e.g., a break up, college acceptance), rate the likely poignancy of the following pieces of memory
ASSISTANT: Got it. I will answer with an integer rating</s>
USER: Memory: {{memory}}
ASSISTANT: Rating: {{#select 'rating'}}1{{or}}2{{or}}3{{or}}4{{or}}5{{or}}6{{or}}7{{or}}8{{or}}9{{or}}10{{/select}}"""
input_dict = {
    "prompt": prompt,
    "arguments": {"memory": "John's best friend bought him a toothbrush"}
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

```{"memory": "John\'s best friend bought him a toothbrush", "rating": " 3"}```

### Sample 2

The following test prompt derives from [Guidance Example](https://github.com/microsoft/guidance#guidance-acceleration-notebook)

```python
import requests
import json

url = "http://localhost:8000/v1/guidance/completions"
headers = {'Content-Type': 'application/json'}

valid_weapons = ["sword", "axe", "mace", "spear", "bow", "crossbow"]
prompt = """The following is a character profile for an RPG game in JSON format.
```json
{
    "id": "{{id}}",
    "description": "{{description}}",
    "name": "{{gen 'name'}}",
    "age": {{gen 'age' pattern='[0-9]+' stop=','}},
    "armor": "{{#select 'armor'}}leather{{or}}chainmail{{or}}plate{{/select}}",
    "weapon": "{{select 'weapon' options=valid_weapons}}",
    "class": "{{gen 'class'}}",
    "mantra": "{{gen 'mantra' temperature=0.7}}",
    "strength": {{gen 'strength' pattern='[0-9]+' stop=','}},
    "items": [{{#geneach 'items' num_iterations=5 join=', '}}"{{gen 'this' temperature=0.7}}"{{/geneach}}]
}```"""
input_dict = {
    "prompt": prompt,
    "arguments": {
        "id": "e1f491f7-7ab8-4dac-8c20-c92b5e7d883d",
        "description": "A quick and nimble wizard.",
        "valid_weapons": valid_weapons,
    },
}
data = {
    "model": "vicuna-7b-v1.1",
    "prompt": json.dumps(input_dict, ensure_ascii=False)
}
res = requests.post(url, headers=headers, json=data)
text = res.json()["choices"][0]["text"]
print(json.loads(text))
```

The output should look like the following:

```json
{
   "id":"e1f491f7-7ab8-4dac-8c20-c92b5e7d883d",
   "description":"A quick and nimble wizard.",
   "valid_weapons":[
      "sword",
      "axe",
      "mace",
      "spear",
      "bow",
      "crossbow"
   ],
   "name":"Quickfingers",
   "age":"25",
   "armor":"leather",
   "weapon":"sword",
   "class":"wizard",
   "mantra":"Pact of the Quick",
   "strength":"10",
   "items":[
      "amulet of lightning",
      "magical staff",
      "mystic tome",
      "10 gold pieces",
      "2 healing potions"
   ]
}
```