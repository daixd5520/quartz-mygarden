# 1 Reward全为-1

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

![[GRPO 踩坑记-2.png]]

![[GRPO 踩坑记-3.png]]
![[GRPO 踩坑记-4.png]]


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
        
    - 尝试调整数据格式，改成带 system prompt 的 messages 格式
        
        暂时无法在飞书文档外展示此内容