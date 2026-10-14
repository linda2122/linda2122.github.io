---
title: LLM_deploy
date: 2026-10-14 15:35:02
tags: 
  - LLM
categories:
  - learning
---
使用fastapi和vllm 在AutoDL平台部署Qwen2.5_0.5b大模型
<!-- more -->
## 配置及环境
### 1、检查python环境 

执行命令：  
```bash
python --version
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```
运行结果：  
Python 3.12.3  
2.8.0+cu128  
True  

### 2、安装相关依赖  

执行命令  
```bash
python -m pip install \
  "transformers==4.46.3" \
  "huggingface_hub==0.36.2" \
  "tokenizers==0.20.3" \
  "accelerate==1.1.1" \
  "safetensors"
```
运行结果：  
Installing collected packages: safetensors, regex, hf-xet, huggingface_hub, tokenizers, transformers, accelerate  
Successfully installed accelerate-1.1.1 hf-xet-1.7.0 huggingface_hub-0.36.2 regex-2026.9.29 safetensors-0.8.0 tokenizers-0.20.3 transformers-4.46.3

### 3、检查安装情况  

执行命令  
`python -m pip check`  

```bash
python -c "import torch, transformers, huggingface_hub, tokenizers; print('torch:',torch.__version__); print('transformers:',transformers.__version__); print('hub:',huggingface_hub.__version__); print('tokenizers:',tokenizers.__version__); print('CUDA:',torch.cuda.is_available())"
```

运行结果：

orch: 2.8.0+cu128  
transformers: 4.46.3  
hub: 0.36.2  
tokenizers: 0.20.3  
CUDA: True  


## 下载模型文件
### 1.配置镜像源，下载json文件   

执行命令  
```bash
export HF_ENDPOINT=https://hf-mirror.com
curl -fL --connect-timeout 15 \
  https://hf-mirror.com/Qwen/Qwen2.5-0.5B-Instruct/resolve/main/config.json \
  -o /tmp/qwen_config.json
```
运行结果：

```text
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current  
                                 Dload  Upload   Total   Spent    Left  Speed  
100   256  100   256    0     0    144      0  0:00:01  0:00:01 --:--:--   144   
100   659    0   659    0     0    339      0 --:--:--  0:00:01 --:--:--     0  
```

执行命令  
```bash
head -c 200 /tmp/qwen_config.json
echo
```
运行结果：   
``` text
  "architectures": [  
    "Qwen2ForCausalLM"  
  ],  
  "attention_dropout": 0.0,  
  "bos_token_id": 151643,  
  "eos_token_id": 151645,  
  "hidden_act": "silu",  
  "hidden_size": 896,  
  "initializer_range":   
```
### 2.正式下载模型文件： 

执行命令  
```bash
python -c "from huggingface_hub import snapshot_download; print(snapshot_download(repo_id='Qwen/Qwen2.5-0.5B-Instruct', local_dir='/root/autodl-tmp/Qwen2.5-0.5B-Instruct'))"
```
运行结果： 

```text
gitattributes: 1.52kB [00:00, 126kB/s]                                                                          | 0/10 [00:00<?, ?it/s]  
config.json: 659B [00:00, 18.0kB/s]                                                                      | 1/10 [00:01<00:13,  1.52s/it]  
tokenizer_config.json: 7.30kB [00:00, 399kB/s]
LICENSE: 11.3kB [00:00, 92.5kB/s]                                                                            | 0.00/988M [00:00<?, ?B/s]
README.md: 4.92kB [00:00, 15.3kB/s]█████▏                                                                | 2/10 [00:02<00:08,  1.09s/it]
merges.txt: 1.67MB [00:01, 1.06MB/s]████████████▎                                                        | 3/10 [00:03<00:07,  1.12s/it]
generation_config.json: 242B [00:00, 1.25MB/s]                                                      | 10.5M/988M [00:01<02:14, 7.29MB/s]
tokenizer.json: 7.03MB [00:02, 2.90MB/s]████████████████████████▌                                        | 5/10 [00:04<00:03,  1.53it/s]
vocab.json: 2.78MB [00:04, 614kB/s]
model.safetensors: 100%|█████████████████████████████████████████████████████████████████████████████| 988M/988M [01:01<00:00, 16.1MB/s]
Fetching 10 files: 100%|████████████████████████████████████████████████████████████████████████████████| 10/10 [01:03<00:00,  6.31s/it]
/root/autodl-tmp/Qwen2.5-0.5B-Instruct███████████████████████████████████████████████████████████████| 988M/988M [01:01<00:00, 16.2MB/s]
```
**模型文件下载完成！**


## 部署
### Hugging Face Transformers + FastAPI
在 AutoDL 安装 API 服务
1. 安装fastapi、uvicorn  

执行命令  
```bash
python -m pip install fastapi uvicorn
```

2. 在 /root/autodl-tmp路径下新建api_server.py，并在py文件中写入服务代码。命令如下   

执行命令  
```bash
cat > /root/autodl-tmp/api_server.py <<'PY'
import os
import torch
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
from transformers import AutoTokenizer, AutoModelForCausalLM

MODEL_PATH = "/root/autodl-tmp/Qwen2.5-0.5B-Instruct"
API_KEY = os.environ.get("API_KEY", "")

app = FastAPI(title="My LLM API")

tokenizer = AutoTokenizer.from_pretrained(
    MODEL_PATH,
    local_files_only=True,
)

model = AutoModelForCausalLM.from_pretrained(
    MODEL_PATH,
    torch_dtype="auto",
    device_map="auto",
    local_files_only=True,
)
model.eval()


class Message(BaseModel):
    role: str
    content: str


class ChatRequest(BaseModel):
    model: str = "Qwen2.5-0.5B-Instruct"
    messages: list[Message]
    max_tokens: int = 256
    temperature: float = 0.7


@app.get("/health")
def health():
    return {
        "status": "ok",
        "device": str(model.device),
        "cuda_available": torch.cuda.is_available(),
    }


@app.post("/v1/chat/completions")
def chat_completions(
    req: ChatRequest,
    authorization: str | None = Header(default=None),
):
    if not API_KEY:
        raise HTTPException(status_code=503, detail="Server API key is not configured")

    if authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="Invalid API key")

    messages = [m.model_dump() for m in req.messages]

    prompt = tokenizer.apply_chat_template(
        messages,
        tokenize=False,
        add_generation_prompt=True,
    )
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

    generation_args = {
        "max_new_tokens": max(1, min(req.max_tokens, 2048)),
        "do_sample": req.temperature > 0,
    }
    if req.temperature > 0:
        generation_args["temperature"] = max(req.temperature, 0.01)
        generation_args["top_p"] = 0.9

    with torch.inference_mode():
        output = model.generate(**inputs, **generation_args)

    new_tokens = output[0][inputs["input_ids"].shape[1]:]
    answer = tokenizer.decode(new_tokens, skip_special_tokens=True)

    return {
        "id": "chatcmpl-local",
        "object": "chat.completion",
        "model": req.model,
        "choices": [{
            "index": 0,
            "message": {"role": "assistant", "content": answer},
            "finish_reason": "stop",
        }],
    }
PY
```

3. 生成API_KEY  

执行命令  
```bash
export API_KEY="$(python -c 'import secrets; print(secrets.token_hex(32))')"
echo "$API_KEY"
```
运行结果  d02fed83eaa1730eeb52ee4476f2ecc44ab27e75f35db54e42cc945f47726bbe

4. 启动  

执行命令  
```bash
cd /root/autodl-tmp
python -m uvicorn api_server:app --host 0.0.0.0 --port 8000
```

运行结果      
```text
INFO:     Started server process [2947]  
INFO:     Waiting for application startup.  
INFO:     Application startup complete.  
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)  
INFO:     127.0.0.1:52386 - "POST /v1/chat/completions HTTP/1.1" 200 OK  
```

5. 测试  

**autoDL另开一个终端**  

执行命令  
```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer d02fed83eaa1730eeb52ee4476f2ecc44ab27e75f35db54e42cc945f47726bbe" \
  -d '{
    "model": "Qwen2.5-0.5B-Instruct",
    "messages": [
      {"role": "user", "content": "你好，请用中文介绍一下自己"}
    ],
    "max_tokens": 128,
    "temperature": 0.7
  }'
```

**本地电脑**  
- 连接autodl  

执行命令   
```powershell
ssh -p 19734 -L 8000:127.0.0.1:8000 root@connect.nmb2.seetacloud.com
```
- 执行上面终端命令  

### vLLM
1. 构建单独部署环境  

安装uv:`python -m pip install -U uv`  
创建独立的 Python 3.12 环境: `uv venv --python 3.12 /root/autodl-tmp/vllm-env --seed`  
激活： `source /root/autodl-tmp/vllm-env/bin/activate`  
安装 vLLM：`uv pip install vllm --torch-backend=auto`  
安装完成后执行验证: `python -c "import vllm, torch; print('vLLM:', vllm.__version__); print('PyTorch:', torch.__version__); print('CUDA:', torch.cuda.is_available())"`  

2. 设置API_KEY  

执行命令  
```bash
export VLLM_API_KEY="$(python -c 'import secrets; print(secrets.token_hex(32))')"
echo "$VLLM_API_KEY"
```

3. 启动 vLLM  

执行命令  
```bash
vllm serve /root/autodl-tmp/Qwen2.5-0.5B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --api-key "$VLLM_API_KEY" \
  --served-model-name Qwen2.5-0.5B-Instruct \
  --max-model-len 4096
```

**报错：**  
> Free memory on device cuda:0 (10.08/11.63 GiB) is less than desired GPU memory utilization (0.92, 10.7 GiB)
意思是：
• GPU 总显存约 11.63 GiB
• 启动时可用显存约 10.08 GiB
• vLLM 默认想预留约 92%，即 10.7 GiB
• 可用显存不够，所以启动中止
vLLM 默认准备使用的显存比例太高，超过了当前可用显存。这是比较容易解决的问题，不需要重新安装 vLLM 或下载模型。
把 `--gpu-memory-utilization`设置为 0.75。这表示 vLLM 最多按 GPU 总显存的约 75% 规划使用量。

**更新命令**
```bash
vllm serve /root/autodl-tmp/Qwen2.5-0.5B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --api-key "$VLLM_API_KEY" \
  --served-model-name Qwen2.5-0.5B-Instruct \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.75
```
vLLM 会加载本地模型并提供 OpenAI 兼容接口。首次启动时请耐心等待模型加载完成。  
运行结果  
Application startup complete

4. 测试  

新开终端测试：  
执行命令  
```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer 这里替换成刚才生成的密钥"
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer 976f48b80837956b8169deeea9da54835d3f7af92383306a0f3f49f4c178dcc4"
```

运行结果  

```json
"object":"list","data":[{"id":"Qwen2.5-0.5B-Instruct","object":"model","created":1791793078,"owned_by":"vllm","root":"/root/autodl-tmp/Qwen2.5-0.5B-Instruct","parent":null,"max_model_len":4096,"permission":[{"id":"modelperm-a162dfd17aa62f78","object":"model_permission","created":1791793078,"allow_create_engine":false,"allow_sampling":true,"allow_logprobs":true,"allow_search_indices":false,"allow_view":true,"allow_fine_tuning":false,"organization":"*","group":null,"is_blocking":false}]}]}  
```

如果返回模型列表，就说明 vLLM API 已经启动成功。  

- 调用测试  

执行命令  
```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 976f48b80837956b8169deeea9da54835d3f7af92383306a0f3f49f4c178dcc4" \
  -d '{
    "model": "Qwen2.5-0.5B-Instruct",
    "messages": [
      {"role": "user", "content": "你好，请用中文介绍一下你自己。"}
    ],
    "temperature": 0.7,
    "max_tokens": 200
  }'
```
运行结果 

```json
{"id":"chatcmpl-9ccb019ceb5fe348","object":"chat.completion","created":1791793190,"model":"Qwen2.5-0.5B-Instruct","choices":[{"index":0,"message":{"role":"assistant","content":"我是Qwen，一个由阿里云开发的超大规模语言模型。我的名字是“Qwen”。我被设计为能够理解和生成人类语言的能力，并且在自然语言处理领域具有广泛的应用前景。\n\n我是通过深度学习和大规模预训练来构建的，旨在回答问题、创作文本以及进行对话。我可以理解多种语言，包括但不限于汉语、英语、法语等，还能处理各种主题和情境下的信息输入。我的目标是尽可能准确地提供有用的信息和支持用户的需求。\n\n此外，我还擅长于生成文学风格的文本，这让我能够在不同的上下文中创造独特的故事和内容。我对AI技术的发展非常感兴趣，并致力于推动其在各个领域的应用和发展。\n\n如果你有任何关于如何使用Qwen或想要了解更多信息的问题，随时欢迎提问！","refusal":null,"annotations":null,"audio":null,"function_call":null,"reasoning":null},"logprobs":null,"finish_reason":"stop","stop_reason":null,"token_ids":null,"routed_experts":null}],"service_tier":null,"system_fingerprint":"vllm-0.31.0-adcc3687","usage":{"prompt_tokens":36,"total_tokens":197,"completion_tokens":161,"prompt_tokens_details":null,"completion_tokens_details":null},"prompt_logprobs":null,"prompt_token_ids":null,"prompt_text":null,"kv_transfer_params":null,"ec_transfer_params":null,"metrics":null}
```

### 对比两种方案
当前方案：Transformers + FastAPI  
• 部署简单，适合学习和验证 API。  
• 自己编写接口逻辑，灵活性高。  
• 并发请求、批处理和显存调度需要额外处理。  
另一种方案：vLLM  
• 专门面向大语言模型推理服务。  
• 支持连续批处理、PagedAttention 等推理优化。  
• 可以直接提供兼容 OpenAI API 的接口，不必自己编写完整的 FastAPI 服务。  
• 更适合后续扩展为多用户共享的模型服务。  

## 注意
### autodl
模型文件下载路径在 /root/autodl-tmp，该路径为数据盘，将机器保存镜像时，数据盘不会保存到镜像内；不用的时候可以关机，但不要删除实例  
新开机器，使用 克隆实例 ，选择数据盘，重新选择。

[金山文档](https://www.kdocs.cn/l/cqHpRXhy0Gv2)
## 总结
无
