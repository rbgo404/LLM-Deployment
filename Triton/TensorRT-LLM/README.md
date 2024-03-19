# Triton TensorRT-LLM
TensorRT-LLM can be used through the docker container. There are multiple ways you can do it:
TensorRT-LLM can be accessed via Docker container, offering two approaches:
1. Clone the [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM) GitHub repository and build the container.
2. Alternatively, clone the [Tensorrtllm_backend](https://github.com/triton-inference-server/tensorrtllm_backend) GitHub repository, which includes TensorRT-LLM. This option is preferred as it avoids potential TensorRT version mismatches during model inference.

## FOLLOW THE FOLLOWING STEP TO BUILD THE CONTAINER:

### STEP 1. Clone [tensorrtllm_backend](https://github.com/triton-inference-server/tensorrtllm_backend.git)
```
git clone https://github.com/triton-inference-server/tensorrtllm_backend.git -b v0.8.0
cd tensorrtllm_backend
git lfs install
git submodule update --init --recursive
```

### STEP 2. Build the Triton TRT-LLM backend container

```
DOCKER_BUILDKIT=1 docker build -t triton_trt_llm -f dockerfile/Dockerfile.trt_llm_backend .
```
Note: Building the container will take some time < 2 hours.

### STEP 3: Model Preperation
Before lunching the container, create a directory `model_input` and download the huggingface model into it. Then create `model_output` directory, where we will store the model engine. Make sure that you are on your home directory.

```bash
mkdir model_input
cd model_input
git lfs install
# For gated model use this -> git clone https://<user_name>:<hf_token>@huggingface.co/meta-llama/Llama-2-7b-hf
git clone https://huggingface.co/meta-llama/Llama-2-7b-hf
# Move to the home directory
cd ..
mkdir model_output
cd model_output
mkdir model_checkpoint
```
### STEP 4. Launch the docker container
```
docker run --gpus all --network host -it --shm-size=1g -v $(pwd)/tensorrtllm_backend/all_models:/all_models -v $(pwd)/tensorrtllm_backend/scripts:/opt/scripts -v ${PWD}/model_input:/model_input -v ${PWD}/model_output:/model_output triton_trt_llm bash
```

### STEP 5. Build engines
Once you are inside the docker container, you can now move to `tensorrt_llm/examples` to convert the model into tensorrt-llm checkpoint format and build the engine.
```
cd tensorrt_llm/examples/llama
# Convert weights from HF Tranformers to tensorrt-llm checkpoint format
python convert_checkpoint.py --model_dir /model_input/Llama-2-7b-hf/ \
                             --output_dir /model_input/model_checkpoint/ \
                             --dtype float16

# Build TensorRT engines
trtllm-build --checkpoint_dir /model_input/model_checkpoint/ \
            --output_dir /model_output/ \
            --gemm_plugin float16 \
            --max_input_len 4000 \
            --max_output_len 4000
```

### STEP 6. Create model repository
```
# Create the model repository that will be used by the Triton server
cd /workspace/tensorrtllm_backend
mkdir triton_model_repo

# Copy the example models to the model repository
cp -r all_models/inflight_batcher_llm/ensemble triton_model_repo/
cp -r all_models/inflight_batcher_llm/preprocessing triton_model_repo/
cp -r all_models/inflight_batcher_llm/postprocessing triton_model_repo/
cp -r all_models/inflight_batcher_llm/tensorrt_llm triton_model_repo/

# Copy the TRT engine to triton_model_repo/tensorrt_llm/1/
cp tensorrt_llm/examples/gpt/engines/fp16/1-gpu/* triton_model_repo/tensorrt_llm/1
```

### 6. Modify the model configuration
The following table shows the fields that need to be modified before deployment:

*triton_model_repo/preprocessing/config.pbtxt*

Mandatory ones are
| Name | Description
| :----------------------: | :-----------------------------: |
| `triton_max_batch_size` | Here setting to 4 |
| `tokenizer_dir` | The path to the tokenizer for the model. In this example, the path should be set to `/workspace/tensorrtllm_backend/tensorrt_llm/examples/gpt/gpt2`|
| `tokenizer_type` | The type of the tokenizer for the model, `t5`, `auto` and `llama` are supported. In this example, the type should be set to `auto` |
| `preprocessing_instance_count` | Here setting to 1 |

#### Run the following command to prepare preprocessing config.pbtxt
```
export HF_GPT_MODEL=/workspace/tensorrtllm_backend/tensorrt_llm/examples/gpt/gpt2
python3 tools/fill_template.py -i triton_model_repo/preprocessing/config.pbtxt "triton_max_batch_size:4,tokenizer_dir:${HF_GPT_MODEL},tokenizer_type:auto,preprocessing_instance_count:1"
```


*triton_model_repo/tensorrt_llm/config.pbtxt*

Mandatory ones are
| Name | Description
| :----------------------: | :-----------------------------: |
| `triton_max_batch_size` | Here setting to 4 |
| `max_queue_delay_microseconds` | Here setting to 100 |
| `decoupled` | Controls streaming. Decoupled mode must be set to `True` if using the streaming option from the client. |
| `gpt_model_type` | Set to `inflight_fused_batching` when enabling in-flight batching support. To disable in-flight batching, set to `V1` |
| `gpt_model_path` | Path to the TensorRT-LLM engines for deployment. In this example, the path should be set to `/workspace/tensorrtllm_backend/triton_model_repo/tensorrt_llm/1` as the tensorrtllm_backend directory will be mounted to `/tensorrtllm_backend` within the container |

Optional
| Name | Description
| :----------------------: | :-----------------------------: |
| `max_beam_width` | The maximum beam width that any request may ask for when using beam search |
| `max_tokens_in_paged_kv_cache` | The maximum size of the KV cache in number of tokens |
| `max_attention_window_size` | When using techniques like sliding window attention, the maximum number of tokens that are attended to generate one token. Defaults to maximum sequence length |
| `batch_scheduler_policy` | Set to `max_utilization` to greedily pack as many requests as possible in each current in-flight batching iteration. This maximizes the throughput but may result in overheads due to request pause/resume if KV cache limits are reached during execution. Set to `guaranteed_no_evict` to guarantee that a started request is never paused.|
| `kv_cache_free_gpu_mem_fraction` | Set to a number between 0 and 1 to indicate the maximum fraction of GPU memory (after loading the model) that may be used for KV cache|
| `max_num_sequences` | Maximum number of sequences that the in-flight batching scheme can maintain state for. Defaults to `max_batch_size` if `enable_trt_overlap` is `false` and to `2 * max_batch_size` if `enable_trt_overlap` is `true`, where `max_batch_size` is the TRT engine maximum batch size.
| `enable_trt_overlap` | Set to `true` to partition available requests into 2 'microbatches' that can be run concurrently to hide exposed CPU runtime |
| `exclude_input_in_output` | Set to `true` to only return completion tokens in a response. Set to `false` to return the prompt tokens concatenated with the generated tokens  |

#### Run the following command to prepare tensorrt_llm model config.pbtxt

```
python3 tools/fill_template.py -i triton_model_repo/tensorrt_llm/config.pbtxt "triton_max_batch_size:4,decoupled_mode:False,engine_dir:/workspace/tensorrtllm_backend/triton_model_repo/tensorrt_llm/1,batching_strategy:V1,max_queue_delay_microseconds:100"
```

*triton_model_repo/postprocessing/config.pbtxt*

Mandatory ones are
| Name | Description
| :----------------------: | :-----------------------------: |
| `triton_max_batch_size` | Here setting to 4 |
| `tokenizer_dir` | The path to the tokenizer for the model. In this example, the path should be set to `/workspace/tensorrtllm_backend/tensorrt_llm/examples/gpt/gpt2`|
| `tokenizer_type` | The type of the tokenizer for the model, `t5`, `auto` and `llama` are supported. In this example, the type should be set to `auto` |
| `postprocessing_instance_count` | Here setting to 1 |

#### Run the following command to prepare postprocessing config.pbtxt
```
export HF_GPT_MODEL=/workspace/tensorrtllm_backend/tensorrt_llm/examples/gpt/gpt2
python3 tools/fill_template.py -i triton_model_repo/postprocessing/config.pbtxt "triton_max_batch_size:4,tokenizer_dir:${HF_GPT_MODEL},tokenizer_type:auto,postprocessing_instance_count:1"
```

*triton_model_repo/ensemble/config.pbtxt*

| Name | Description
| :----------------------: | :-----------------------------: |
| `triton_max_batch_size` | Here setting to 4 |

#### Run the following command to prepare ensemble config.pbtxt
```
python3 tools/fill_template.py -i triton_model_repo/ensemble/config.pbtxt "triton_max_batch_size:4"
```

### 7. Launch Triton server

Run this command to launch Triton server. Set `--world_size` = number of GPUs aka TP degree.

```
python3 scripts/launch_triton_server.py --world_size=1 --model_repo=/workspace/tensorrtllm_backend/triton_model_repo
```

### 8. Query the server with the Triton generate endpoint
[Query the server with the Triton generate endpoint](https://github.com/triton-inference-server/tensorrtllm_backend#query-the-server-with-the-triton-generate-endpoint)

```
curl -X POST localhost:8000/v2/models/ensemble/generate -d '{"text_input": "What is machine learning?", "max_tokens": 20, "bad_words": "", "stop_words": ""}'
```
