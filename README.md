# Quick start

Use Docker to download and start an API server for the llama model. The API server is written in Python and runs the llama model when API requests come in. Its source code is in this repo.

```
docker pull divinaventi/llama-web-server
docker run -p 8000:8000 divinaventi/llama-web-server
```

Send in an HTTP API request to the server to see the model information.

```
curl -X GET http://localhost:8000/v1/models \
  -H 'accept: application/json'
```

The response is

```
{
  "object": "list",
  "data": [
    {
      "id": "/llama-cpp-python/vendor/llama.cpp/models/7B/ggml-model-q4_0.bin",
      "object": "model",
      "owned_by": "me",
      "permissions": []
    }
  ]
}
```

Send in an HTTP API request to prompt the model and ask it to answer a question.

```
curl -X POST http://localhost:8000/v1/chat/completions \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{"messages":[{"role":"system", "content": "You are a helpful assistance. Provide helpful and informative responses in a concise and complete manner. Please avoid using conversational tags and only reply in full sentences. Ensure that your answers are presented directly and without the human of '\''Human:'\'' or '\''###'\''. Thank you for your cooperation"}, {"role":"user", "content": "What was the significance of Joseph Weizenbaum'\''s ElIZA program?"}], "max_tokens":64}'
```

The response is

```
{
  "id": "chatcmpl-0cfe6cdf-28be-4bb0-a39e-9572e678c0d1",
  "object": "chat.completion",
  "created": 1690825931,
  "model": "/llama-cpp-python/vendor/llama.cpp/models/7B/ggml-model-q4_0.bin",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The eliza program is an artificial intelligence program. It simulates a Rogerian therapist and was created by Weizanbam in 1964. The user can ask it questions or give it statements to respond with replies that are designed to make them feel better. However, the computer"
      },
      "finish_reason": "length"
    }
  ],
  "usage": {
    "prompt_tokens": 92,
    "completion_tokens": 64,
    "total_tokens": 156
  }
}
```

## Other relevant docker images

```bash
docker pull divinaventi/openllama-web-server
docker pull divinaventi/llama-web-server
docker pull divinaventi/panda-web-server
docker pull divinaventi/llama-frontend
docker compose up
```

# Local build instructions for the llama API server

Python3 can be installed through apt (simultaneously with a few other necessary dependencies like gcc, etc) with the command:

```bash
# optional but recommended
sudo apt-get upgrade
# the actual install for python3 and some other required libraries for building the project successfully as well as its functionality
sudo apt-get install python3 python3-pip python-is-python3 libopenblas-dev build-essential
```

After installing python with the above commands, the following required python libraries can also be installed with: 

```bash
python -m pip install --upgrade pip pytest cmake scikit-build setuptools fastapi uvicorn sse-starlette pydantic-settings
```

The web server is built from abetlan's llama-cpp-python repo and will need to be initialized. You need to build a shared library file of `llama.cpp` for the server's use by navigating to `vendor/llama.cpp` and running the command `make libllama.so` then moving it to the `llama_cpp` folder located in the `llama-cpp-python` directory. All of these setup steps are accomplished with:

```bash
cd ByteLlama
# initialize llama-cpp-python and llama.cpp with a submodule pull (path automatically configured assuming run from the ByteLlama main directory)
git submodule update --init --recursive
# navigate to llama.cpp repo
cd llama-web-server/llama-cpp-python/vendor/llama.cpp/
# increase the context length
sed -i 's/uint32_t n_ctx   = 512;/uint32_t n_ctx   = 2048;/' llama.cpp
# build the shared library file and move it to the right location
make libllama.so
mv libllama.so ../../llama_cpp
# return to ByteLlama main directory
cd ../../../..
```

Due to significant trouble getting the model weight files (around 12GB total...) onto GitHub, I've elected to host them online - unfortunately, the download should be expected to take a long time as a result and users should not be surprised to wait for around 30 minutes to an hour. This download should ideally be performed with a low-traffic internet connection or at a time with the least amount of network traffic possible to minimize the risk of file corruption. Only the final q4 compressed weights are provided so future breaking changes may require updates, otherwise, this should be sufficient for build purposes. These files may be downloaded with the following commands:

```bash
# silent download files into their appropriate folders with curl
curl -s -L --remote-name-all https://7b-llm-models-1302315972.cos.ap-beijing.myqcloud.com/7B.zip https://7b-llm-models-1302315972.cos.ap-beijing.myqcloud.com/OpenLlama7B.zip https://7b-llm-models-1302315972.cos.ap-beijing.myqcloud.com/Panda7BInstr.zip

# Unzip and copy into the right folder
unzip -q 7B.zip
mv 7B llama-web-server/llama-cpp-python/vendor/llama.cpp/models/
rm -rf 7B.zip
```

Next, install the API server with the model.

```bash
cd llama-web-server/llama-cpp-python
LLAMA_OPENBLAS=1 pip install llama_cpp_python
```

Finally, start the API server:

```bash
python3 -m llama_cpp.server --model vendor/llama.cpp/models/7B/ggml-model-q4_0.bin
```

Try the CLI command in Quick start to test the API server.

```
curl -X GET http://localhost:8000/v1/models \
  -H 'accept: application/json'

{"object":"list","data":[{"id":"vendor/llama.cpp/models/7B/ggml-model-q4_0.bin","object":"model","owned_by":"me","permissions":[]}]}
```

# Install and run the web frontend

Install node.js

```bash
# optional but recommended
sudo apt-get upgrade
# the actual install
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - &&\
sudo apt-get install -y nodejs
```

The node package dependencies required for the frontend can be installed by navigating into the `llama-frontend` subdirectory and then automatically loading the packages from the provided `package.json` file:

```bash
cd ByteLlama
cd llama-frontend
# install the relevant node dependencies with node package manager using package.json saved in llama-frontend
npm install
# return to the ByteLlama main directory
cd ..
```

Start the frontend server at port 80.

```bash
sudo npm run dev
```

With both the frontend and the API server running, you can access the chat UI via your browser at:

```
http://localhost/
```
