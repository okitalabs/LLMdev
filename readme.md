# LLM開発環境の構築
さまざまなLLMの実装、検証を行うためのDocker環境の構築。

<hr>

## 基本設定

### User情報
Dockerで作成するアカウント情報。Host側と同じにしておく。ホームディレクトリはHostとは別の前提。

|Host|Docker|Desk|
|:----|:----|:----|
|USERNAME|llmdev|ユーザ名|
|USER_UID|10001|ユーザID|
|GROUPNAME|llmdev|グループ名|
|USER_GID|10000|グループID|
|HOME_DIR|/home/users/llmdev|ホームディレクトリ|
|Docker名|llmdev||

### Dir情報
HostでもDockerでも同じ扱いに出来るようにするため、ディレクトリ配置は同じにしておく。

|Host|Docker|Desk|
|:----|:----|:----|
|/home/users/llmdev|/home/users/llmdev|HOMEディレクトリ|
|/home/users/model|/home/users/model|モデルファイル置き場|

### Port情報
イロイロなサーバを起動できるようにあらかじめポートを割り当てる。コンテナごとに10000番台、20000番台と割り当てると判りやすい。
|Host|Docker|Desc|
|:----|:----|:----|
|18000|8000|FastChat|
|18010|8010|llama-cpp HTTP Server|
|18020|8020|llama-cpp-python Server|
|18030|8030| |
|18040|8040| |
|18050|8050| |
|18060|8060| |
|18070|8070| |
|18080|8080|LiteLLM Proxy|
|18090|8090| |
|18888|8888|Jupyter|

<hr>

## HOMEの作成

Docker用のHOMEとModelディレクトリを作成しておく。ユーザ、グループはすでにある前提。
```
$ mkdir /home/users/llmdev
$ mkdir /home/users/model
$ cp /etc/skel/.bashrc /home/users/llmdev
$ chown -R llmdev:llmdev /home/users/llmdev
$ chown -R llmdev:llmdev /home/users/model
```

`.bashrc`に追記しておく。
```
export HUGGINGFACE_HUB_CACHE=/home/users/model/
```


## Dockerの作成

huggingfaceのGPU用Docker Imageをベースにする。  

`Jupyter`、`llama-cpp-python`、`llama-cpp HTTP Server`、`LiteLLM Proxy`を入れる例。入れたいパッケージを適宜追加する。

`Dockerfile`
```
FROM huggingface/transformers-all-latest-gpu

ENV HOST 0.0.0.0

RUN apt update && apt install -y vim curl wget w3m git sl

## 必要なライブラリを追加する
## Jupyter
RUN pip install jupyterlab ipywidgets iprogress ## numpy scikit-learn scipy pandas matplotlib seaborn plotly gensim statsmodels Pillow joblib flask


## llama-cpp-python
ENV CUDA_DOCKER_ARCH=all
ENV LLAMA_CUBLAS=1
RUN python3 -m pip install --upgrade pip pytest cmake scikit-build setuptools fastapi uvicorn sse-starlette pydantic-settings starlette-context
RUN CMAKE_ARGS="-DLLAMA_CUBLAS=on" pip install llama-cpp-python

## LiteLLM Proxy
RUN pip install 'litellm[proxy]'

## llamacpp HTTP Server
WORKDIR /tmp
ENV LLAMA_CUDA=1
ENV LLAMA_CURL=1
RUN apt-get install -y build-essential git libcurl4-openssl-dev
RUN git clone https://github.com/ggerganov/llama.cpp.git && cd llama.cpp/ && LLAMA_CUDA=1 LLAMA_CURL=1 make && cp ./server /usr/local/bin/llamacpp-server

## Create User
ARG USERNAME=okita ## <--ユーザ名に変更
ARG USER_UID=10001 ## <--ユーザIDに変更
ARG GROUPNAME=hp   ## <--グループ名に変更
ARG USER_GID=10000 ## <--グループIDに変更
ARG HOME_DIR=/home/users/llmdev ## <--ホームディレクトリに変更
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID --no-create-home --home-dir $HOME_DIR $USERNAME \
    && rm /etc/apt/sources.list.d/cuda.list \
    && apt-get clean \
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

WORKDIR $HOME_DIR
USER $USERNAME
```

### ビルド
```
docker build -t llmdev .
```

### 実行
```
docker run -itd --gpus all --cap-add SYS_RESOURCE -e USE_MLOCK=0 \
--add-host=host.docker.internal:host-gateway \
-p 18000:8000 \
-p 18010:8010 \
-p 18020:8020 \
-p 18030:8030 \
-p 18040:8040 \
-p 18050:8050 \
-p 18060:8060 \
-p 18070:8070 \
-p 18080:8080 \
-p 18090:8090 \
-p 18888:8888 \
-v /home/users/llmdev:/home/users/llmdev/ \
-v /home/users/model:/home/users/model/ \
-h llmdev --name llmdev llmdev \
/bin/bash
```

### Dockerログイン
Docker内で作業する場合、以下でコンテナに入る。
```
docker exec -it llmdev /bin/bash
```

## アプリ実行
以下は、Docker内で実行する想定。

### Jupyter
Hostからはブラウザで、`http://localhost:18888`、token`hoge`で接続。
```
jupyter-lab --no-browser --port=8888 --ip=0.0.0.0 --allow-root --NotebookApp.token="hoge"
```

### llama-cpp HTTP Server
`vicuna-13b-v1.5.Q8_0.gguf`を`Port:8010`で起動する例。モデルはあらかじめ`/home/users/model`にダウンロードしておく。

```
llamacpp-server  \
--chat-template vicuna \
--threads-batch 8 \
--threads-http 8 \
--model /home/users/model/vicuna-13b-v1.5.Q8_0.gguf \
--ctx-size 4095 \
--embeddings \
--parallel 8 \
--cont-batching \
--n-gpu-layers 96 \
--host 0.0.0.0 \
--port 8010
```

### llama-cpp-python Server
設定ファイル  
` llamacpp-python.json`
```
{
  "host": "0.0.0.0",
  "port": 8020,
  "models": [
    {
      "model": "/home/users/model/vicuna-13b-v1.5.Q8_0.gguf",
      "model_alias": "vicuna-7b-v1.5-q8",
      "chat_format": "vicuna",
      "n_gpu_layers": -1,
      "n_ctx": 4096
    }
  ]
}
```

起動
```
python3 -m llama_cpp.server --config_file llamacpp-python.json
```


### LiteLLM Proxy
llama-cpp HTTP Serverにアクセスする場合。
設定ファイル  
` litellm.yaml`

```
model_list:
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: openai/vicuna-13b
      api_base: http://localhost:8010/v1
      api_key: None
  - model_name: text-embedding-ada-002
    litellm_params:
      model: openai/vicuna-13b
      api_base: http://localhost:8010/v1
      api_key: None
```

起動
```
litellm --config litellm.yaml --detailed_debug --port 8000 --
```

## 確認
### Chat Completion
HostからLiteLLMにアクセスする場合。Docker内からアクセスする場合、Portは`8080`になる。
```
$ curl http://localhost:18080/v1/chat/completions \
-H "Content-Type: application/json" \
-H "Authorization: Bearer None" \
-d '{
  "model": "gpt-3.5-turbo",
  "templature": 0.1,
  "top_p": 0.1,
  "messages": [
    {"role": "system", "content": "あなたは優秀な観光ガイドです。"},
    {"role": "user", "content": "日本の都道府県名をランダムに１つ回答してください。その都道府県名の魅力を3つ答えてください。"}
  ]
}'
```

### Embedding
```
$ curl http://localhost:18080/v1/embeddings \
-H "Content-Type: application/json" \
-H "Authorization: Bearer None" \
-d '{
  "model": "text-embedding-ada-002",
  "input": "query: 夕飯はお肉です。"
}' 
```

<hr>

LLM実行委員会