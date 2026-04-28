# 1 Reward全为“-1”

## 1.1 启动命令

```shell file:启动命令
export PLAN_REWARD_VERSION=v2
export PLAN_REWARD_PLUGIN_WORKERS=${MLP_WORKER_GPU}
export PLAN_REWARD_PLUGIN_LOG_EVERY=1
pip install swanlab

FORCE_TORCHRUN=1 \
NNODES=${MLP_WORKER_NUM} \
NODE_RANK=${MLP_ROLE_INDEX} \
NPROC_PER_NODE=${MLP_WORKER_GPU} \
MASTER_ADDR=${MLP_WORKER_0_PRIMARY_HOST} \
MASTER_PORT=29500 \
swift rlhf \
    --rlhf_type grpo \
    --model /dcar_ai_vepfs/yuhongjiang/models/qwen3-4b-struct_plan_v6/v2-20260419-173600-iter-2200-hf \
    --external_plugins /dcar_ai_vepfs/daixindi/script/plugins/plan_reward_decoupled/0424/PlanReward_plugin.py \
    --reward_funcs plan_xml_reward_decoupled \
    --dataset /dcar_ai_vepfs/daixindi/data/grpo/FinalPlan500_with_plan_prompt-1grpo.jsonl \
    --train_type lora \
    --torch_dtype bfloat16 \
    --max_completion_length 512 \
    --num_train_epochs 1 \
    --per_device_train_batch_size 2 \
    --per_device_eval_batch_size 2 \
    --learning_rate 1e-6 \
    --gradient_accumulation_steps 2 \
    --eval_steps 10 \
    --save_steps 10 \
    --logging_steps 5 \
    --save_total_limit 5 \
    --warmup_ratio 0.05 \
    --num_generations 4 \
    --steps_per_generation 4 \
    --temperature 1.6 \
    --max_length 8192 \
    --output_dir /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled \
    --dataloader_num_workers 1 \
    --dataset_num_proc 1 \
    --deepspeed zero3 \
    --report_to swanlab \
    --swanlab_token wihfZNZ9BnOuZWeePxDT3 \
    --swanlab_project grpo_qwen3-4b-struct_plan_v6-temp07 \
    --swanlab_exp_name grpo_qwen3-4b-struct_plan_v6 
```

## 1.2 错误现象

![[GRPO 踩坑记-2.png]]

## 1.3 训练监控

![[GRPO 踩坑记-3.png]]

![[GRPO 踩坑记-4.png]]

### 1.3.1 错误分析

**[训练监控](https://swanlab.cn/@dxindi14700/grpo_qwen3-4b-struct_plan_v6-temp07/runs/e2iafbxh8yl6p3wf43pog/chart)**显示，

- 大部分 step 的 `PlanReward` 耗时都非常低，几乎贴地
- 中间只有一次明显尖峰
- 支持一个判断：大部分样本在 hard gate 阶段就被快速判死了，没有真正走到耗时更高的 judge 逻辑
- 推测：模型输出**大多不合法，**可能是 4b SFT 模型的输出问题
    - 进一步看输出监控：`train/completions/max_length` 一直贴着 `512`：很可疑，说明输出经常顶到`--max_completion_length 512`，而我们 SFT 模型的输出一般达不到这个长度（？）
    - 加入控制台输出跑了一次，确实是没有按格式要求输出。
        - 模型：/dcar_ai_vepfs/yuhongjiang/models/qwen3-4b-struct_plan_v6/v2-20260419-173600-iter-2200-hf

        ![[GRPO 踩坑记-1.png]]

    - dcd_llm请求这个模型没问题，应该是 swift 框架数据格式问题

        ```Shell
        {"prompt": [{"role": "user", "content": "你是一个“选车语义解析模型”，。。。。\n\n========================\n【用户输入】\n========================\n雷克萨斯RX350h四驱版驾驶体验"}],"query": "雷克萨斯RX350h四驱版驾驶体验", "select_schema": "|xxxx|xx|","where_schema": "xx|xx", "task": "plan_xml"}
        ```


    ```file:log
        [2026-04-28 14:11:35] vllm_enable_lora=False,

[2026-04-28 14:11:35] vllm_enable_prefix_caching=True,

[2026-04-28 14:11:35] vllm_enforce_eager=False,

[2026-04-28 14:11:35] vllm_engine_kwargs={},

[2026-04-28 14:11:35] vllm_gpu_memory_utilization=0.9,

[2026-04-28 14:11:35] vllm_limit_mm_per_prompt={},

[2026-04-28 14:11:35] vllm_max_lora_rank=16,

[2026-04-28 14:11:35] vllm_max_model_len=None,

[2026-04-28 14:11:35] vllm_max_num_seqs=256,

[2026-04-28 14:11:35] vllm_mm_processor_cache_gb=None,

[2026-04-28 14:11:35] vllm_mode=colocate,

[2026-04-28 14:11:35] vllm_pipeline_parallel_size=1,

[2026-04-28 14:11:35] vllm_quantization=None,

[2026-04-28 14:11:35] vllm_reasoning_parser=None,

[2026-04-28 14:11:35] vllm_server_base_url=None,

[2026-04-28 14:11:35] vllm_server_host=None,

[2026-04-28 14:11:35] vllm_server_pass_dataset=False,

[2026-04-28 14:11:35] vllm_server_port=[8000],

[2026-04-28 14:11:35] vllm_server_timeout=240.0,

[2026-04-28 14:11:35] vllm_tensor_parallel_size=1,

[2026-04-28 14:11:35] vllm_use_async_engine=False,

[2026-04-28 14:11:35] wandb_log_unique_prompts=None,

[2026-04-28 14:11:35] warmup_ratio=0.05,

[2026-04-28 14:11:35] warmup_steps=0,

[2026-04-28 14:11:35] weight_decay=0.1,

[2026-04-28 14:11:35] whiten_rewards=False,

[2026-04-28 14:11:35] zero_hpz_partition_size=None,

[2026-04-28 14:11:35] )

[2026-04-28 14:11:35] [INFO:swift] model_kwargs: {'device_map': None, 'torch_dtype': torch.bfloat16}

[2026-04-28 14:11:35] [2026-04-28 14:11:35,852] [INFO] [partition_parameters.py:345:__exit__] finished initializing model - num_params = 399, num_elems = 4.41B

Loading checkpoint shards: 100%|██████████| 2/2 [00:04<00:00, 2.19s/it]

[2026-04-28 14:11:40] 

Loading checkpoint shards: 100%|██████████| 2/2 [00:04<00:00, 2.19s/it]

Loading checkpoint shards: 100%|██████████| 2/2 [00:04<00:00, 2.19s/it]

Loading checkpoint shards: 100%|██████████| 2/2 [00:04<00:00, 2.20s/it]

[2026-04-28 14:11:40] 

Loading checkpoint shards: 100%|██████████| 2/2 [00:04<00:00, 2.19s/it]

Loading checkpoint shards: 100%|██████████| 2/2 [00:05<00:00, 2.61s/it]

[2026-04-28 14:11:41] [INFO:swift] model_info: ModelInfo(model_type='qwen3', model_dir='/dcar_ai_vepfs/yuhongjiang/models/qwen3-4b-struct_plan_v6/v2-20260419-173600-iter-2200-hf', torch_dtype=torch.bfloat16, max_model_len=262144, quant_method=None, quant_bits=None, rope_scaling=None, is_moe_model=False, config=Qwen3Config {

[2026-04-28 14:11:41] "architectures": [

[2026-04-28 14:11:41] "Qwen3ForCausalLM"

[2026-04-28 14:11:41] ],

[2026-04-28 14:11:41] "attention_bias": false,

[2026-04-28 14:11:41] "attention_dropout": 0.0,

[2026-04-28 14:11:41] "bos_token_id": 151643,

[2026-04-28 14:11:41] "eos_token_id": 151645,

[2026-04-28 14:11:41] "head_dim": 128,

[2026-04-28 14:11:41] "hidden_act": "silu",

[2026-04-28 14:11:41] "hidden_size": 2560,

[2026-04-28 14:11:41] "initializer_range": 0.02,

[2026-04-28 14:11:41] "intermediate_size": 9728,

[2026-04-28 14:11:41] "layer_types": [

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention",

[2026-04-28 14:11:41] "full_attention"

[2026-04-28 14:11:41] ],

[2026-04-28 14:11:41] "max_position_embeddings": 262144,

[2026-04-28 14:11:41] "max_window_layers": 36,

[2026-04-28 14:11:41] "model_type": "qwen3",

[2026-04-28 14:11:41] "num_attention_heads": 32,

[2026-04-28 14:11:41] "num_hidden_layers": 36,

[2026-04-28 14:11:41] "num_key_value_heads": 8,

[2026-04-28 14:11:41] "pad_token_id": 151643,

[2026-04-28 14:11:41] "rms_norm_eps": 1e-06,

[2026-04-28 14:11:41] "rope_scaling": null,

[2026-04-28 14:11:41] "rope_theta": 5000000,

[2026-04-28 14:11:41] "sliding_window": null,

[2026-04-28 14:11:41] "tie_word_embeddings": true,

[2026-04-28 14:11:41] "torch_dtype": "bfloat16",

[2026-04-28 14:11:41] "transformers_version": "4.55.2",

[2026-04-28 14:11:41] "use_cache": false,

[2026-04-28 14:11:41] "use_sliding_window": false,

[2026-04-28 14:11:41] "vocab_size": 151936

[2026-04-28 14:11:41] }

[2026-04-28 14:11:41] , task_type='causal_lm', num_labels=None)

[2026-04-28 14:11:41] [INFO:swift] model.generation_config: GenerationConfig {

[2026-04-28 14:11:41] "bos_token_id": 151643,

[2026-04-28 14:11:41] "do_sample": true,

[2026-04-28 14:11:41] "eos_token_id": [

[2026-04-28 14:11:41] 151645,

[2026-04-28 14:11:41] 151643

[2026-04-28 14:11:41] ],

[2026-04-28 14:11:41] "max_new_tokens": 512,

[2026-04-28 14:11:41] "pad_token_id": 151643,

[2026-04-28 14:11:41] "temperature": 1.2,

[2026-04-28 14:11:41] "top_p": 0.9

[2026-04-28 14:11:41] }

[2026-04-28 14:11:41] 

[2026-04-28 14:11:41] [INFO:swift] default_system: None

[2026-04-28 14:11:41] [INFO:swift] max_length: 8192

[2026-04-28 14:11:41] [INFO:swift] response_prefix: ''

[2026-04-28 14:11:41] [INFO:swift] agent_template: hermes

[2026-04-28 14:11:41] [INFO:swift] Start time of running main: 2026-04-28 14:11:41.099379

[2026-04-28 14:11:41] [INFO:swift] swift.__version__: 3.11.0.dev0

Generating train split: 500 examples [00:00, 3913.67 examples/s]

Map: 100%|██████████| 500/500 [00:00<00:00, 5977.84 examples/s]

[2026-04-28 14:11:45] [INFO:swift] train_dataset: Dataset({

[2026-04-28 14:11:45] features: ['messages', 'query', 'select_schema', 'where_schema', 'task'],

[2026-04-28 14:11:45] num_rows: 500

[2026-04-28 14:11:45] })

[2026-04-28 14:11:45] [INFO:swift] val_dataset: None

[2026-04-28 14:11:45] [INFO:swift] The RLHFArguments will be saved in: /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled/v5-20260428-141125/args.json

[2026-04-28 14:11:45] [INFO:swift] lora_config: LoraConfig(task_type='CAUSAL_LM', peft_type=<PeftType.LORA: 'LORA'>, auto_mapping=None, base_model_name_or_path='/dcar_ai_vepfs/yuhongjiang/models/qwen3-4b-struct_plan_v6/v2-20260419-173600-iter-2200-hf', revision=None, inference_mode=False, r=8, target_modules={'v_proj', 'q_proj', 'k_proj', 'o_proj', 'gate_proj', 'up_proj', 'down_proj'}, exclude_modules=None, lora_alpha=32, lora_dropout=0.05, fan_in_fan_out=False, bias='none', use_rslora=False, modules_to_save=[], init_lora_weights=True, layers_to_transform=None, layers_pattern=None, rank_pattern={}, alpha_pattern={}, megatron_config=None, megatron_core='megatron.core', trainable_token_indices=None, loftq_config={}, eva_config=None, corda_config=None, use_dora=False, layer_replication=None, runtime_config=LoraRuntimeConfig(ephemeral_gpu_offload=False), lora_bias=False, lora_dtype=None, lorap_lr_ratio=None, lorap_emb_lr=1e-06)

[2026-04-28 14:11:45] [INFO:swift] model: PeftModelForCausalLM(

[2026-04-28 14:11:45] (base_model): LoraModel(

[2026-04-28 14:11:45] (model): Qwen3ForCausalLM(

[2026-04-28 14:11:45] (model): Qwen3Model(

[2026-04-28 14:11:45] (embed_tokens): Embedding(151936, 2560, padding_idx=151643)

[2026-04-28 14:11:45] (layers): ModuleList(

[2026-04-28 14:11:45] (0-35): 36 x Qwen3DecoderLayer(

[2026-04-28 14:11:45] (self_attn): Qwen3Attention(

[2026-04-28 14:11:45] (q_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=2560, out_features=4096, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=2560, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=4096, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (k_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=2560, out_features=1024, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=2560, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=1024, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (v_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=2560, out_features=1024, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=2560, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=1024, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (o_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=4096, out_features=2560, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=4096, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=2560, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (q_norm): Qwen3RMSNorm((0,), eps=1e-06)

[2026-04-28 14:11:45] (k_norm): Qwen3RMSNorm((0,), eps=1e-06)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (mlp): Qwen3MLP(

[2026-04-28 14:11:45] (gate_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=2560, out_features=9728, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=2560, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=9728, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (up_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=2560, out_features=9728, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=2560, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=9728, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (down_proj): lora.Linear(

[2026-04-28 14:11:45] (base_layer): Linear(in_features=9728, out_features=2560, bias=False)

[2026-04-28 14:11:45] (lora_dropout): ModuleDict(

[2026-04-28 14:11:45] (default): Dropout(p=0.05, inplace=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_A): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=9728, out_features=8, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_B): ModuleDict(

[2026-04-28 14:11:45] (default): Linear(in_features=8, out_features=2560, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lora_embedding_A): ParameterDict()

[2026-04-28 14:11:45] (lora_embedding_B): ParameterDict()

[2026-04-28 14:11:45] (lora_magnitude_vector): ModuleDict()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (act_fn): SiLU()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (input_layernorm): Qwen3RMSNorm((0,), eps=1e-06)

[2026-04-28 14:11:45] (post_attention_layernorm): Qwen3RMSNorm((0,), eps=1e-06)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (norm): Qwen3RMSNorm((0,), eps=1e-06)

[2026-04-28 14:11:45] (rotary_emb): Qwen3RotaryEmbedding()

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] (lm_head): Linear(in_features=2560, out_features=151936, bias=False)

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] )

[2026-04-28 14:11:45] [INFO:swift] model_parameter_info: PeftModelForCausalLM: 4038.9832M Params (16.5151M Trainable [0.4089%]), 0.0001M Buffers.

[2026-04-28 14:11:46] Detected kernel version 5.4.250, which is below the recommended minimum of 5.5.0; this can cause the process to hang. It is recommended to upgrade the kernel to the minimum version or higher.

[2026-04-28 14:11:46] [INFO:swift] use_reentrant: True

[2026-04-28 14:11:46] [INFO:swift] The logging file will be saved in: /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled/v5-20260428-141125/logging.jsonl

[2026-04-28 14:11:47] Parameter Offload: Total persistent parameters: 5946880 in 469 params

[2026-04-28 14:11:50] swanlab: Tracking run with swanlab version 0.7.16

[2026-04-28 14:11:50] swanlab: Run data will be saved locally in 

[2026-04-28 14:11:50] /swanlog/run-20260428_141150-k3ykg0n1xcrxwpa0r5uql

[2026-04-28 14:11:50] swanlab: 👋 Hi dxindi14700,welcome to swanlab!

[2026-04-28 14:11:50] swanlab: Syncing run grpo_qwen3-4b-struct_plan_v6 to the cloud

[2026-04-28 14:11:50] swanlab: 🏠 View project at 

[2026-04-28 14:11:50] https://swanlab.cn/@dxindi14700/grpo_qwen3-4b-struct_plan_v6-temp07

[2026-04-28 14:11:50] swanlab: 🚀 View run at 

[2026-04-28 14:11:50] https://swanlab.cn/@dxindi14700/grpo_qwen3-4b-struct_plan_v6-temp07/runs/k3ykg0n

[2026-04-28 14:11:50] 1xcrxwpa0r5uql

Train: 0%| | 0/30 [00:00<?, ?it/s][plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=54.11s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:29] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=58.87s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:33] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=63.10s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:34] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=64.05s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:35] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=64.65s rewards={1.000:5, -0.800:3} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:40] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=69.50s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:51] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=80.74s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:13:51] [plan_xml_reward_decoupled] call_counter=1 version=v2 cost_time=81.20s rewards={1.000:4, -0.800:3, 0.830:1} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:20] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:14:20] warnings.warn(

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2338:3699 [0] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2339:3670 [1] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2344:3701 [6] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2343:3678 [5] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2341:3697 [3] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2342:3696 [4] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2345:3705 [7] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2340:3688 [2] NCCL INFO Comm config Blocking set to 1

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Using non-device net plugin version 0

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Using network IB

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO ncclCommInitRank comm 0x7fd97aa3efa0 rank 4 nranks 16 cudaDev 4 nvmlDev 4 busId 69020 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO ncclCommInitRank comm 0x7efd4a7402a0 rank 5 nranks 16 cudaDev 5 nvmlDev 5 busId 69030 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO ncclCommInitRank comm 0x7f2be2a4fb30 rank 3 nranks 16 cudaDev 3 nvmlDev 3 busId 67030 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO ncclCommInitRank comm 0x7f2362a19490 rank 7 nranks 16 cudaDev 7 nvmlDev 7 busId 6b030 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO ncclCommInitRank comm 0x7fa30aa9e7e0 rank 2 nranks 16 cudaDev 2 nvmlDev 2 busId 67020 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO ncclCommInitRank comm 0x7f1c16a5b820 rank 6 nranks 16 cudaDev 6 nvmlDev 6 busId 6b020 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO ncclCommInitRank comm 0x7f941aa939e0 rank 1 nranks 16 cudaDev 1 nvmlDev 1 busId 65030 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO ncclCommInitRank comm 0x7f3f8ea49340 rank 0 nranks 16 cudaDev 0 nvmlDev 0 busId 65020 commId 0x43cd2ca05a3d557 - Init START

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:42] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NCCL_TOPO_FILE set by environment to /var/run/nvidia-topologyd/virtualTopology.xml

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Setting affinity for GPU 7 to ffff,ffffffff,ffffffff,ffffffff,ff000000,00000000,00000000,00000000

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS multicast support is available on dev 7

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Setting affinity for GPU 2 to ffffff,ffffffff,ffffffff,ffffffff

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS multicast support is available on dev 2

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Setting affinity for GPU 6 to ffff,ffffffff,ffffffff,ffffffff,ff000000,00000000,00000000,00000000

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS multicast support is available on dev 6

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Setting affinity for GPU 4 to ffff,ffffffff,ffffffff,ffffffff,ff000000,00000000,00000000,00000000

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS multicast support is available on dev 4

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Setting affinity for GPU 0 to ffffff,ffffffff,ffffffff,ffffffff

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS multicast support is available on dev 0

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Setting affinity for GPU 1 to ffffff,ffffffff,ffffffff,ffffffff

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS multicast support is available on dev 1

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Setting affinity for GPU 3 to ffffff,ffffffff,ffffffff,ffffffff

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS multicast support is available on dev 3

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Setting affinity for GPU 5 to ffff,ffffffff,ffffffff,ffffffff,ff000000,00000000,00000000,00000000

[2026-04-28 14:14:47] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS multicast support is available on dev 5

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO comm 0x7fa30aa9e7e0 rank 2 nRanks 16 nNodes 2 localRanks 8 localRank 2 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO comm 0x7f2be2a4fb30 rank 3 nRanks 16 nNodes 2 localRanks 8 localRank 3 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO comm 0x7f2362a19490 rank 7 nRanks 16 nNodes 2 localRanks 8 localRank 7 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO comm 0x7efd4a7402a0 rank 5 nRanks 16 nNodes 2 localRanks 8 localRank 5 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO comm 0x7fd97aa3efa0 rank 4 nRanks 16 nNodes 2 localRanks 8 localRank 4 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO comm 0x7f1c16a5b820 rank 6 nRanks 16 nNodes 2 localRanks 8 localRank 6 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO comm 0x7f941aa939e0 rank 1 nRanks 16 nNodes 2 localRanks 8 localRank 1 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Trees [0] 3/-1/-1->2->1 [1] 3/10/-1->2->-1 [2] 3/-1/-1->2->1 [3] 3/-1/-1->2->1 [4] -1/-1/-1->2->3 [5] 1/-1/-1->2->3 [6] 1/-1/-1->2->3 [7] 1/-1/-1->2->3 [8] 3/-1/-1->2->1 [9] 3/-1/-1->2->10 [10] 3/-1/-1->2->1 [11] 3/-1/-1->2->1 [12] -1/-1/-1->2->3 [13] 1/-1/-1->2->3 [14] 1/-1/-1->2->3 [15] 1/-1/-1->2->3

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO comm 0x7f3f8ea49340 rank 0 nRanks 16 nNodes 2 localRanks 8 localRank 0 MNNVL 0

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Trees [0] 4/-1/-1->3->2 [1] 4/-1/-1->3->2 [2] -1/-1/-1->3->2 [3] 4/-1/-1->3->2 [4] 2/-1/-1->3->4 [5] 2/11/-1->3->-1 [6] 2/-1/-1->3->4 [7] 2/-1/-1->3->4 [8] 4/-1/-1->3->2 [9] 4/-1/-1->3->2 [10] -1/-1/-1->3->2 [11] 4/-1/-1->3->2 [12] 2/-1/-1->3->4 [13] 2/-1/-1->3->11 [14] 2/-1/-1->3->4 [15] 2/-1/-1->3->4

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Trees [0] -1/-1/-1->7->6 [1] 0/-1/-1->7->6 [2] 0/-1/-1->7->6 [3] 0/-1/-1->7->6 [4] 6/-1/-1->7->0 [5] 6/-1/-1->7->0 [6] 6/-1/-1->7->0 [7] 6/15/-1->7->-1 [8] -1/-1/-1->7->6 [9] 0/-1/-1->7->6 [10] 0/-1/-1->7->6 [11] 0/-1/-1->7->6 [12] 6/-1/-1->7->0 [13] 6/-1/-1->7->0 [14] 6/-1/-1->7->0 [15] 6/-1/-1->7->15

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS Head 0: 0 8

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS Head 1: 2 10

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Trees [0] 6/-1/-1->5->4 [1] 6/-1/-1->5->4 [2] 6/-1/-1->5->4 [3] -1/-1/-1->5->4 [4] 4/-1/-1->5->6 [5] 4/-1/-1->5->6 [6] 4/13/-1->5->-1 [7] 4/-1/-1->5->6 [8] 6/-1/-1->5->4 [9] 6/-1/-1->5->4 [10] 6/-1/-1->5->4 [11] -1/-1/-1->5->4 [12] 4/-1/-1->5->6 [13] 4/-1/-1->5->6 [14] 4/-1/-1->5->13 [15] 4/-1/-1->5->6

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Trees [0] 5/-1/-1->4->3 [1] 5/-1/-1->4->3 [2] 5/12/-1->4->-1 [3] 5/-1/-1->4->3 [4] 3/-1/-1->4->5 [5] -1/-1/-1->4->5 [6] 3/-1/-1->4->5 [7] 3/-1/-1->4->5 [8] 5/-1/-1->4->3 [9] 5/-1/-1->4->3 [10] 5/-1/-1->4->12 [11] 5/-1/-1->4->3 [12] 3/-1/-1->4->5 [13] -1/-1/-1->4->5 [14] 3/-1/-1->4->5 [15] 3/-1/-1->4->5

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS Head 2: 4 12

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Trees [0] 2/-1/-1->1->0 [1] -1/-1/-1->1->0 [2] 2/-1/-1->1->0 [3] 2/-1/-1->1->0 [4] 0/9/-1->1->-1 [5] 0/-1/-1->1->2 [6] 0/-1/-1->1->2 [7] 0/-1/-1->1->2 [8] 2/-1/-1->1->0 [9] -1/-1/-1->1->0 [10] 2/-1/-1->1->0 [11] 2/-1/-1->1->0 [12] 0/-1/-1->1->9 [13] 0/-1/-1->1->2 [14] 0/-1/-1->1->2 [15] 0/-1/-1->1->2

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Trees [0] 7/-1/-1->6->5 [1] 7/-1/-1->6->5 [2] 7/-1/-1->6->5 [3] 7/14/-1->6->-1 [4] 5/-1/-1->6->7 [5] 5/-1/-1->6->7 [6] -1/-1/-1->6->7 [7] 5/-1/-1->6->7 [8] 7/-1/-1->6->5 [9] 7/-1/-1->6->5 [10] 7/-1/-1->6->5 [11] 7/-1/-1->6->14 [12] 5/-1/-1->6->7 [13] 5/-1/-1->6->7 [14] -1/-1/-1->6->7 [15] 5/-1/-1->6->7

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS Head 3: 6 14

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/16 : 0 7 6 5 4 3 2 1 8 15 14 13 12 11 10 9

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 01/16 : 0 7 6 5 4 3 10 9 8 15 14 13 12 11 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 02/16 : 0 7 6 5 12 11 10 9 8 15 14 13 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 03/16 : 0 7 14 13 12 11 10 9 8 15 6 5 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/16 : 0 7 6 5 4 3 2 1 8 15 14 13 12 11 10 9

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 05/16 : 0 7 6 5 4 3 10 9 8 15 14 13 12 11 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 06/16 : 0 7 6 5 12 11 10 9 8 15 14 13 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 07/16 : 0 7 14 13 12 11 10 9 8 15 6 5 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/16 : 0 7 6 5 4 3 2 1 8 15 14 13 12 11 10 9

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 09/16 : 0 7 6 5 4 3 10 9 8 15 14 13 12 11 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 10/16 : 0 7 6 5 12 11 10 9 8 15 14 13 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 11/16 : 0 7 14 13 12 11 10 9 8 15 6 5 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/16 : 0 7 6 5 4 3 2 1 8 15 14 13 12 11 10 9

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 13/16 : 0 7 6 5 4 3 10 9 8 15 14 13 12 11 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 14/16 : 0 7 6 5 12 11 10 9 8 15 14 13 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 15/16 : 0 7 14 13 12 11 10 9 8 15 6 5 4 3 2 1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Trees [0] 1/8/-1->0->-1 [1] 1/-1/-1->0->7 [2] 1/-1/-1->0->7 [3] 1/-1/-1->0->7 [4] 7/-1/-1->0->1 [5] 7/-1/-1->0->1 [6] 7/-1/-1->0->1 [7] -1/-1/-1->0->1 [8] 1/-1/-1->0->8 [9] 1/-1/-1->0->7 [10] 1/-1/-1->0->7 [11] 1/-1/-1->0->7 [12] 7/-1/-1->0->1 [13] 7/-1/-1->0->1 [14] 7/-1/-1->0->1 [15] -1/-1/-1->0->1

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO P2P Chunksize set to 131072

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 01/0 : 3[3] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 00/0 : 1[1] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 02/0 : 5[5] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 02/0 : 13[5] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 03/0 : 15[7] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 05/0 : 3[3] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 04/0 : 1[1] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 01/0 : 11[3] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 06/0 : 5[5] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 09/0 : 3[3] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 07/0 : 15[7] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 06/0 : 13[5] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 08/0 : 1[1] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 10/0 : 5[5] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/0 : 9[1] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 05/0 : 11[3] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 11/0 : 15[7] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 10/0 : 13[5] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 12/0 : 1[1] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 14/0 : 5[5] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 13/0 : 3[3] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 09/0 : 11[3] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/0 : 9[1] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 15/0 : 15[7] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 14/0 : 13[5] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 13/0 : 11[3] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/0 : 9[1] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/0 : 9[1] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 01/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 02/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 03/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 03/0 : 7[7] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 07/0 : 7[7] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 11/0 : 7[7] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 15/0 : 7[7] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 05/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 06/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 07/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 09/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 10/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 11/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 00/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 00/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 01/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 13/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 01/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 02/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 14/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 02/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 03/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 15/0 : 0[0] -> 7[7] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 03/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 04/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 00/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 00/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 00/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 00/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 04/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 05/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 01/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 02/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 01/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 01/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 05/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 06/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 03/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 03/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 02/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 02/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 06/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 07/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 04/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 04/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 04/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 03/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 07/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 08/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 05/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 06/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 04/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 05/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 08/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 07/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 09/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 07/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 05/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 06/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 09/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 10/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 08/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 06/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 11/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 08/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 10/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 07/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 12/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 08/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 08/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 10/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 09/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 11/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 13/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 10/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 12/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 09/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 14/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 11/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 09/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 13/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 11/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 15/0 : 4[4] -> 3[3] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 12/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 10/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 14/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 12/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 11/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 13/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 12/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 15/0 : 2[2] -> 1[1] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 13/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 12/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 14/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 13/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 15/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 14/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 15/0 : 6[6] -> 5[5] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 14/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 15/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 01/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 02/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 03/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 05/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 06/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 07/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 09/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 10/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 11/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 13/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 14/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:48] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 15/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 00/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 01/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 02/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 04/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 05/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 07/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 08/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 09/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 10/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 12/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 13/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 15/0 : 5[5] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 00/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 01/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 02/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 03/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 04/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 05/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 06/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 07/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 00/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 08/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 01/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 09/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Connected all rings

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 01/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 02/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 10/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 02/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 03/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 11/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 03/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 04/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 12/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 05/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 13/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 05/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 06/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 14/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 06/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 07/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 15/0 : 4[4] -> 5[5] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 07/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 08/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 09/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 09/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 10/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 10/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 11/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 11/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 12/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 13/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 00/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 13/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 14/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 01/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 14/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 15/0 : 6[6] -> 7[7] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 02/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 15/0 : 0[0] -> 1[1] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 06/0 : 13[5] -> 5[5] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 03/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 03/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 07/0 : 15[7] -> 7[7] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 14/0 : 13[5] -> 5[5] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 11/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 15/0 : 15[7] -> 7[7] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 06/0 : 5[5] -> 13[5] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 03/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 07/0 : 7[7] -> 15[7] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 14/0 : 5[5] -> 13[5] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 11/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 15/0 : 7[7] -> 15[7] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 00/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 04/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 00/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 02/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 05/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 01/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 03/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 06/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 03/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 05/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 07/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 04/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 06/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 08/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 06/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 07/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 09/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 07/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 08/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 10/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 08/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 10/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 11/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 09/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 11/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 12/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 11/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 13/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 13/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 12/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 14/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 14/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 14/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 15/0 : 1[1] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 15/0 : 2[2] -> 3[3] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 15/0 : 3[3] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 00/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 04/0 : 9[1] -> 1[1] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 08/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 01/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 12/0 : 9[1] -> 1[1] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 05/0 : 11[3] -> 3[3] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 02/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 09/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 04/0 : 1[1] -> 9[1] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 01/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 10/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 13/0 : 11[3] -> 3[3] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 12/0 : 1[1] -> 9[1] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 02/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 09/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 05/0 : 3[3] -> 11[3] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 10/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 13/0 : 3[3] -> 11[3] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 01/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 02/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 03/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 04/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 05/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 06/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 09/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 10/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 11/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 12/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 13/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 14/0 : 7[7] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 00/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 04/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 08/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Channel 12/0 : 1[1] -> 0[0] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 01/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 02/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 05/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 06/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 09/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 10/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Channel 13/0 : 3[3] -> 2[2] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Channel 14/0 : 5[5] -> 4[4] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 03/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 07/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 11/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Channel 15/0 : 7[7] -> 6[6] via P2P/IPC

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Connected all trees

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO NVLS comm 0x7efd4a7402a0 headRank -1 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO NVLS comm 0x7fd97aa3efa0 headRank 2 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO NVLS comm 0x7f2be2a4fb30 headRank -1 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO NVLS comm 0x7fa30aa9e7e0 headRank 1 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO NVLS comm 0x7f941aa939e0 headRank -1 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO NVLS comm 0x7f3f8ea49340 headRank 0 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO NVLS comm 0x7f1c16a5b820 headRank 3 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO NVLS comm 0x7f2362a19490 headRank -1 nHeads 4 buffSize 1048576 memSize 2097152 nvlsPerRankSize 100663296 nvlsTotalSize 402653184

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 00/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 02/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 00/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 03/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 01/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 04/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 03/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 00/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 01/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 04/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 05/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 01/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 02/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 05/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 06/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 02/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 06/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 07/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 04/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 08/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 07/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 05/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 10/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 06/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 08/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 09/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 07/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 11/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 08/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 12/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 11/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 09/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 13/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 12/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 10/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 14/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 13/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 12/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 15/0 : 10[2] -> 2[2] [receive] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 14/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 03/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 13/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 00/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 15/0 : 12[4] -> 4[4] [receive] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 14/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 00/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 01/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 15/0 : 14[6] -> 6[6] [receive] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 00/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 03/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 01/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 04/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 02/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 05/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 04/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 06/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 05/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 07/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 02/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 06/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 03/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 08/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 07/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 08/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 09/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 10/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 12/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 13/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 05/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 14/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Channel 15/0 : 6[6] -> 14[6] [send] via NET/IB/3/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 04/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 09/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 06/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 07/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 05/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 09/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 06/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 10/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 11/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 11/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 07/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 13/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 08/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 14/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 10/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 15/0 : 8[0] -> 0[0] [receive] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 11/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 01/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 12/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 12/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 02/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 13/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 03/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 14/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 04/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Channel 15/0 : 2[2] -> 10[2] [send] via NET/IB/1/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 05/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 06/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 13/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 07/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 14/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 09/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Channel 15/0 : 4[4] -> 12[4] [send] via NET/IB/2/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 10/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 11/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 12/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 13/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 14/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Channel 15/0 : 0[0] -> 8[0] [send] via NET/IB/0/GDRDMA

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO Connected NVLS tree

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 512 | 512

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO 16 coll channels, 16 collnet channels, 16 nvls channels, 16 p2p channels, 2 p2p channels per peer

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2344:3739 [6] NCCL INFO ncclCommInitRank comm 0x7f1c16a5b820 rank 6 nranks 16 cudaDev 6 nvmlDev 6 busId 6b020 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2338:3737 [0] NCCL INFO ncclCommInitRank comm 0x7f3f8ea49340 rank 0 nranks 16 cudaDev 0 nvmlDev 0 busId 65020 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2340:3744 [2] NCCL INFO ncclCommInitRank comm 0x7fa30aa9e7e0 rank 2 nranks 16 cudaDev 2 nvmlDev 2 busId 67020 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2342:3742 [4] NCCL INFO ncclCommInitRank comm 0x7fd97aa3efa0 rank 4 nranks 16 cudaDev 4 nvmlDev 4 busId 69020 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2341:3740 [3] NCCL INFO ncclCommInitRank comm 0x7f2be2a4fb30 rank 3 nranks 16 cudaDev 3 nvmlDev 3 busId 67030 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2345:3743 [7] NCCL INFO ncclCommInitRank comm 0x7f2362a19490 rank 7 nranks 16 cudaDev 7 nvmlDev 7 busId 6b030 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2343:3741 [5] NCCL INFO ncclCommInitRank comm 0x7efd4a7402a0 rank 5 nranks 16 cudaDev 5 nvmlDev 5 busId 69030 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:49] t-20260428141043-cmz7n-worker-0:2339:3738 [1] NCCL INFO ncclCommInitRank comm 0x7f941aa939e0 rank 1 nranks 16 cudaDev 1 nvmlDev 1 busId 65030 commId 0x43cd2ca05a3d557 - Init COMPLETE

[2026-04-28 14:14:59] Invalidate trace cache @ step 1555: expected module 3068, but got module 3069

{'loss': 0.0, 'grad_norm': 0.00732043, 'learning_rate': 5e-07, 'completions/mean_length': 93.234375, 'completions/min_length': 41.0, 'completions/max_length': 196.0, 'completions/clipped_ratio': 0.0, 'reward': -0.29843751, 'reward_std': 0.10928577, 'frac_reward_zero_std': 0.8125, 'rewards/PlanReward/mean': -0.29843751, 'rewards/PlanReward/std': 0.80521297, 'kl': 0.0, 'clip_ratio/low_mean': 0.0, 'clip_ratio/low_min': 0.0, 'clip_ratio/high_mean': 0.0, 'clip_ratio/high_max': 0.0, 'clip_ratio/region_mean': 0.0, 'epoch': 0.03, 'global_step/max_steps': '1/30', 'percentage': '3.33%', 'elapsed_time': '3m 14s', 'remaining_time': '1h 33m 50s', 'memory(GiB)': 22.01, 'train_speed(iter/s)': 0.005151}

Train: 7%|▋ | 2/30 [03:37<43:43, 93.71s/it] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=46.70s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:16:59] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=62.52s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:00] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=62.80s rewards={-0.800:7, 1.000:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:06] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=69.25s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:07] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=70.30s rewards={-0.800:7, 0.930:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:08] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=71.38s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:14] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=76.55s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:24] [plan_xml_reward_decoupled] call_counter=2 version=v2 cost_time=86.68s rewards={-0.800:5, 0.880:1, 0.895:1, 0.930:1} stages={fatal_by_judge:5, scored:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:17:26] Invalidate trace cache @ step 1555: expected module 6137, but got module 6136

Train: 13%|█▎ | 4/30 [06:22<33:57, 78.35s/it] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=53.89s rewards={-0.800:7, 0.840:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:19:47] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=62.47s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:19:50] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=65.30s rewards={-0.800:4, 0.950:1, 1.000:1, 0.675:1, 0.890:1} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:19:55] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=70.15s rewards={-0.800:7, 0.930:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:19:59] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=73.71s rewards={-0.800:5, 0.810:2, 1.000:1} stages={fatal_by_judge:5, scored:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:19:59] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=74.10s rewards={-0.800:6, 1.000:1, 0.915:1} stages={fatal_by_judge:6, scored:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:20:06] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=80.74s rewards={-0.800:7, 0.950:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:20:22] [plan_xml_reward_decoupled] call_counter=3 version=v2 cost_time=97.07s rewards={0.910:3, -0.800:3, 0.925:1, 0.890:1} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:20:24] Invalidate trace cache @ step 1555: expected module 9205, but got module 9204

{'loss': 3.44e-06, 'grad_norm': 0.00774626, 'learning_rate': 9.7e-07, 'kl': -2.67e-06, 'clip_ratio/low_mean': 0.0, 'clip_ratio/low_min': 0.0, 'clip_ratio/high_mean': 0.0, 'clip_ratio/high_max': 0.0, 'clip_ratio/region_mean': 0.0, 'completions/mean_length': 93.55859375, 'completions/min_length': 45.0, 'completions/max_length': 140.0, 'completions/clipped_ratio': 0.0, 'reward': -0.35031252, 'reward_std': 0.20021517, 'frac_reward_zero_std': 0.734375, 'rewards/PlanReward/mean': -0.35031251, 'rewards/PlanReward/std': 0.75553727, 'epoch': 0.17, 'global_step/max_steps': '5/30', 'percentage': '16.67%', 'elapsed_time': '9m 4s', 'remaining_time': '45m 24s', 'memory(GiB)': 25.6, 'train_speed(iter/s)': 0.009175}

Train: 20%|██ | 6/30 [09:23<31:09, 77.88s/it] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=36.11s rewards={1.000:4, 0.705:2, 0.725:1, 0.695:1} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:22:57] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=53.47s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:06] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=62.41s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:08] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=64.71s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:10] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=66.35s rewards={-0.800:5, 0.670:1, 0.690:1, 0.580:1} stages={fatal_by_judge:5, scored:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:12] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=68.09s rewards={-0.800:6, 0.860:1, 1.000:1} stages={fatal_by_judge:6, scored:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:25] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=81.56s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:25] [plan_xml_reward_decoupled] call_counter=4 version=v2 cost_time=81.77s rewards={-0.800:7, 0.965:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:23:28] Invalidate trace cache @ step 1555: expected module 12273, but got module 12272

Train: 27%|██▋ | 8/30 [12:28<28:50, 78.64s/it] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=52.73s rewards={-0.800:4, 1.000:3, 0.735:1} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:11] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=59.17s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:11] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=59.59s rewards={-0.800:4, 1.000:2, 0.970:1, 0.985:1} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:17] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=65.71s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:20] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=68.68s rewards={-0.800:6, 0.840:1, 0.810:1} stages={fatal_by_judge:6, scored:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:22] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=70.33s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:23] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=71.96s rewards={-0.800:7, 0.850:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:44] [plan_xml_reward_decoupled] call_counter=5 version=v2 cost_time=91.98s rewards={-0.800:2, 0.970:2, 0.910:1, 0.860:1, 0.715:1, 1.000:1} stages={scored:6, fatal_by_judge:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:26:47] Invalidate trace cache @ step 1555: expected module 15341, but got module 15340

{'loss': -3.4e-07, 'grad_norm': 0.00633409, 'learning_rate': 8.1e-07, 'kl': -5.21e-06, 'clip_ratio/low_mean': 0.0, 'clip_ratio/low_min': 0.0, 'clip_ratio/high_mean': 0.0, 'clip_ratio/high_max': 0.0, 'clip_ratio/region_mean': 0.0, 'completions/mean_length': 101.515625, 'completions/min_length': 49.5, 'completions/max_length': 269.0, 'completions/clipped_ratio': 0.0, 'reward': -0.2217383, 'reward_std': 0.19795913, 'frac_reward_zero_std': 0.671875, 'rewards/PlanReward/mean': -0.22173829, 'rewards/PlanReward/std': 0.82877958, 'epoch': 0.33, 'global_step/max_steps': '10/30', 'percentage': '33.33%', 'elapsed_time': '15m 48s', 'remaining_time': '31m 37s', 'memory(GiB)': 25.6, 'train_speed(iter/s)': 0.010539}

Train: 33%|███▎ | 10/30 [15:48<27:31, 82.59s/it][INFO:swift] Saving model checkpoint to /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled/v5-20260428-141125/checkpoint-10

[2026-04-28 14:28:40] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=26.05s rewards={1.000:8} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:10] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=56.13s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:11] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=57.15s rewards={1.000:5, -0.800:2, 0.500:1} stages={scored:6, fatal_by_judge:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:16] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=62.48s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:26] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=72.45s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:30] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=75.77s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:29:36] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=81.84s rewards={-0.800:7, 0.955:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:30:09] [plan_xml_reward_decoupled] call_counter=6 version=v2 cost_time=114.93s rewards={-0.800:2, 1.000:2, 0.960:1, 0.980:1, 0.650:1, 0.500:1} stages={scored:6, fatal_by_judge:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:30:09] /usr/local/lib/python3.11/site-packages/torch/utils/checkpoint.py:87: UserWarning: None of the inputs have requires_grad=True. Gradients will be None

[2026-04-28 14:30:09] warnings.warn(

[2026-04-28 14:30:10] Invalidate trace cache @ step 1555: expected module 18409, but got module 18408

Train: 40%|████ | 12/30 [19:04<24:54, 83.05s/it] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=32.05s rewards={1.000:8} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:20] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=53.01s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:24] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=56.89s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:27] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=59.44s rewards={-0.800:7, 0.800:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:30] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=62.65s rewards={-0.800:4, 0.620:1, 0.515:1, 0.475:1, 0.630:1} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:41] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=73.71s rewards={-0.800:6, 0.930:1, 0.910:1} stages={fatal_by_judge:6, scored:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:44] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=76.47s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:49] [plan_xml_reward_decoupled] call_counter=7 version=v2 cost_time=82.03s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:32:51] Invalidate trace cache @ step 1555: expected module 21477, but got module 21476

Train: 47%|████▋ | 14/30 [21:44<20:06, 75.43s/it] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=19.00s rewards={1.000:8} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:35:29] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=57.93s rewards={1.000:6, 0.985:2} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:35:30] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=58.58s rewards={-0.800:7, 0.940:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:35:38] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=67.24s rewards={-0.800:4, 1.000:3, 0.970:1} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:35:39] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=67.77s rewards={-0.800:6, 1.000:2} stages={fatal_by_judge:6, scored:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:35:42] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=70.85s rewards={1.000:4, -0.800:2, 0.590:1, 0.525:1} stages={scored:6, fatal_by_judge:2} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:36:00] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=88.86s rewards={-0.800:6, 0.800:1, -1.000:1} stages={fatal_by_judge:6, scored:1, hard_invalid:1} gate_reasons={local_parser_fail:XML-Where结构错误:1} exceptions={} invalid_like=1/8

[2026-04-28 14:36:00] [plan_xml_reward_decoupled][sample] idx=0 stage=fatal_by_judge gate_ok=True gate_reason=<empty> fatal_reason=1. select部分使用的分类名称不在给定的Select-Schema允许的字段范围内，违反Schema硬约束，无法正常执行查询；2. 仅覆盖了Query中三款候选车型的筛选要求，遗漏了用户明确提及的充电条件、行驶场景、行驶里程、使用地域等核心选车需求；3. 未发现幻觉类冗余约束。 judge_reason=1. select部分使用的分类名称不在给定的Select-Schema允许的字段范围内，违反Schema硬约束，无法正常执行查询；2. 仅覆盖了Query中三款候选车型的筛选要求，遗漏了用户明确提及的充电条件、行驶场景、行驶里程、使用地域等核心选车需求；3. 未发现幻觉类冗余约束。 exception=<empty> select_schema_present=True where_schema_present=True query_preview=至境L7，领克10emp240U，和别克至境L7奢享逍遥智行款，每年8000公里，2000高速，市区通勤偏多，浙江省内使用，没有固定车位装不了充电桩 completion_preview=<params1>\n <select>\n 汽车基础参数, 汽车能耗类属性\n </select>\n <where>\n 候选车系##至境L7##必要\n </where>\n <order>\n </order>\n <group_by>\n ...

[2026-04-28 14:36:00] [plan_xml_reward_decoupled][sample] idx=1 stage=fatal_by_judge gate_ok=True gate_reason=<empty> fatal_reason=存在Schema硬约束违反，<select>所选字段不在允许范围内；候选车信息提取不完整，未提取用户明确给出的使用场景、使用区域、充电条件等约束，整体可执行性差。 judge_reason=存在Schema硬约束违反，<select>所选字段不在允许范围内；候选车信息提取不完整，未提取用户明确给出的使用场景、使用区域、充电条件等约束，整体可执行性差。 exception=<empty> select_schema_present=True where_schema_present=True query_preview=至境L7，领克10emp240U，和别克至境L7奢享逍遥智行款，每年8000公里，2000高速，市区通勤偏多，浙江省内使用，没有固定车位装不了充电桩 completion_preview=<params1>\n <select>\n 汽车基础参数, 汽车能耗类属性, 汽车续航类属性\n </select>\n <where>\n 候选车系##至境L7##必要\n </where>\n <order>\n </order>\n <gro...

[2026-04-28 14:36:00] [plan_xml_reward_decoupled][sample] idx=3 stage=hard_invalid gate_ok=False gate_reason=local_parser_fail:XML-Where结构错误 fatal_reason=<empty> judge_reason=<empty> exception=<empty> select_schema_present=True where_schema_present=True query_preview=至境L7，领克10emp240U，和别克至境L7奢享逍遥智行款，每年8000公里，2000高速，市区通勤偏多，浙江省内使用，没有固定车位装不了充电桩 completion_preview=<params1>\n <select>\n 汽车基础参数, 汽车能耗类属性\n </select>\n <where>\n 候选车系##至境L7##必要\n </where>\n <order>\n </order>\n <group_by>\n ...

[2026-04-28 14:36:07] [plan_xml_reward_decoupled] call_counter=8 version=v2 cost_time=96.23s rewards={1.000:4, -0.800:3, 0.980:1} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:36:09] Invalidate trace cache @ step 1555: expected module 24545, but got module 24544

{'loss': -4.96e-06, 'grad_norm': 0.00678222, 'learning_rate': 5.6e-07, 'completions/mean_length': 94.54427083, 'completions/min_length': 41.0, 'completions/max_length': 213.0, 'completions/clipped_ratio': 0.0, 'reward': -0.09744792, 'reward_std': 0.20627892, 'frac_reward_zero_std': 0.67708333, 'rewards/PlanReward/mean': -0.09744793, 'rewards/PlanReward/std': 0.86356654, 'kl': -2.08e-06, 'clip_ratio/low_mean': 0.0, 'clip_ratio/low_min': 0.0, 'clip_ratio/high_mean': 0.0, 'clip_ratio/high_max': 0.0, 'clip_ratio/region_mean': 0.0, 'epoch': 0.5, 'global_step/max_steps': '15/30', 'percentage': '50.00%', 'elapsed_time': '24m 51s', 'remaining_time': '24m 51s', 'memory(GiB)': 25.6, 'train_speed(iter/s)': 0.010058}

Train: 53%|█████▎ | 16/30 [25:07<18:54, 81.03s/it] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=58.97s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:38:39] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=64.61s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:38:43] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=67.82s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:38:44] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=69.21s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:38:49] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=74.42s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:38:55] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=80.49s rewards={-0.800:7, 0.910:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:39:00] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=84.73s rewards={-0.800:5, 0.870:2, 1.000:1} stages={fatal_by_judge:5, scored:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:39:03] [plan_xml_reward_decoupled] call_counter=9 version=v2 cost_time=87.96s rewards={1.000:4, -0.800:3, 0.980:1} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:39:28] Invalidate trace cache @ step 1555: expected module 27613, but got module 27612

Train: 60%|██████ | 18/30 [28:22<16:27, 82.29s/it] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=28.70s rewards={1.000:8} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:41:45] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=52.07s rewards={1.000:5, 0.950:1, 0.970:1, 0.994:1} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:41:48] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=54.58s rewards={1.000:5, -0.800:3} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:41:50] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=56.28s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:41:52] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=58.35s rewards={1.000:4, 0.585:1, 0.530:1, 0.795:1, 0.765:1} stages={scored:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:41:55] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=61.62s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:42:10] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=76.59s rewards={-0.800:3, 0.975:2, 1.000:2, 0.950:1} stages={scored:5, fatal_by_judge:3} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:42:24] [plan_xml_reward_decoupled] call_counter=10 version=v2 cost_time=90.33s rewards={-0.800:4, 0.990:1, 0.890:1, 0.930:1, 0.910:1} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:42:26] Invalidate trace cache @ step 1555: expected module 30681, but got module 30680

{'loss': -9.49e-06, 'grad_norm': 0.00698948, 'learning_rate': 2.8e-07, 'kl': -3.06e-06, 'clip_ratio/low_mean': 0.0, 'clip_ratio/low_min': 0.0, 'clip_ratio/high_mean': 0.0, 'clip_ratio/high_max': 0.0, 'clip_ratio/region_mean': 0.0, 'completions/mean_length': 101.8046875, 'completions/min_length': 45.0, 'completions/max_length': 199.0, 'completions/clipped_ratio': 0.0, 'reward': -0.23178126, 'reward_std': 0.23207857, 'frac_reward_zero_std': 0.6875, 'rewards/PlanReward/mean': -0.23178127, 'rewards/PlanReward/std': 0.81763405, 'epoch': 0.67, 'global_step/max_steps': '20/30', 'percentage': '66.67%', 'elapsed_time': '31m 21s', 'remaining_time': '15m 40s', 'memory(GiB)': 25.6, 'train_speed(iter/s)': 0.01063}

Train: 67%|██████▋ | 20/30 [31:21<13:11, 79.11s/it][INFO:swift] Saving model checkpoint to /dcar_ai_vepfs/daixindi/models/grpo_qwen3-4b-struct_plan_v6_plan_reward_decoupled/v5-20260428-141125/checkpoint-20

[2026-04-28 14:44:46] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=59.32s rewards={1.000:4, -0.800:4} stages={scored:4, fatal_by_judge:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:44:50] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=63.24s rewards={-0.800:7, 0.970:1} stages={fatal_by_judge:7, scored:1} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:44:59] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=71.69s rewards={-0.800:4, 1.000:4} stages={fatal_by_judge:4, scored:4} gate_reasons={} exceptions={} invalid_like=0/8

[2026-04-28 14:45:00] [plan_xml_reward_decoupled] call_counter=11 version=v2 cost_time=72.67s rewards={-0.800:8} stages={fatal_by_judge:8} gate_reasons={} exceptions={} invalid_like=0/8

    ```

### 1.3.2 实验观测 1
- 尝试调整数据格式，改成带 system prompt 的 messages 格式
    - 能正常输出了。log 见上
    ![[GRPO 踩坑记-5.png]]

现在设置的温度是 1.2，max_output_len 512
2worker，8 卡/机，每个卡显存占用大概 3-4GB
![[GRPO 踩坑记-6.png]]
# 2 