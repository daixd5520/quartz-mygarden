# 请求线上模型

`select_car_feature_server` 里的 `call_plan_tag_extractor`，跑完以后执行 `xml_usability_all`。

![[实验运行方式记录-2.png]]

# 请求未上线模型

流程：火山机器学习平台 → 在线服务 → 最后一页 `ai-search-reply` → 调用指南最后一页，复制"公网访问地址"，改到 `motor_dynamic_summary` 的 `dcd_llm.py` 里。

![[实验运行方式记录-1.png]]

跑数用 `batch_plan_tag_from_csv.py`：

```shell
python /cloudide/workspace/motor_dynamic_summary/batch_plan_tag_from_csv.py \
  --csv "/cloudide/workspace/motor_dynamic_summary/old_csvs/224human/224plan_eval_human_224_first224_with_label_where_schema_with_llmoutput_validator_20260416_164211.csv" \
  --query "query" \
  --label "label"
```

# 获取选车部分大 prompt

`motor_dynamic_summary` 的——（待补）
