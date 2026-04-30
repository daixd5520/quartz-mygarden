---
tags:
  - landing
  - prompt
  - GRPO
title: 500 human eval 的 plan prompt 长度统计
draft: "true"
---

# 500 human eval 的 plan prompt 长度统计

`FinalPlan500` 这份 500 条 human eval 集对应的 plan prompt，跑了一次 token 长度统计（`count_plan_prompt_tokens.py`）。一句话结论：**这份数据集下 plan prompt 平均接近 8k token，最长 10k**——这是做 GRPO `--max_length` 预算、judge 链路切分时必须记住的数字。

## 分布

```
File: /cloudide/workspace/motor_dynamic_summary/old_csvs/bigprompts/FinalPlan500_with_plan_prompt-1.csv
Column: plan_prompt
Rows: 500
Total tokens: 3,987,791
Average tokens: 7,975.58
Median tokens:  7,899.50
Min tokens:     6,983
Max tokens:    10,080
```

Top 10（按 token 数）：

| Row | Tokens | Query |
| --- | --- | --- |
| 349 | 10080 | 你好，请根据以下条件，帮我选出3-5款推荐的车型... 主要个人使用，不用考虑家庭。场景是城市代步通勤，日常 |
| 362 | 9857  | 中型或中大型 SUV 汽油车，不要双离合，15 万以内，皮实耐操 |
| 384 | 9744  | 国产操作好，高速适用，跑市区，SUV 旅行可放平，绿色新能源 |
| 293 | 9610  | 预算 4 万以下，要电车，空间大的，三箱或者小 SUV |
| 332 | 9474  | 十五万左右，省油，耐造，SUV，纯油，内饰好的车 |
| 267 | 9278  | SUV，13 万以下大众，本田，丰田 |
| 377 | 9237  | 硬派越野风，溜背造型，掀背式车身，低趴外形，小钢炮 |
| 167 | 9107  | 凌际星云对比雅升 VITO |
| 230 | 9080  | 价格 10 万以下，轿车，汽油，马力大，推荐一些热门车型 |
| 132 | 9073  | 瑞虎 9 与 RAV4 如何选 |

## 几个工程上用得到的观察

分布很紧：median 7.9k，min 6.9k。prompt 模板大、schema 全量灌进去，query 长短对总长度影响几乎可以忽略。所以 plan 阶段上下文窗口 8k 做不了，至少 12k 才够留出输出空间。

`--max_completion_length` 留 512–1024 比较保险，对应的 GRPO `--max_length` 建议 12288 以上。

Top 10 里出现了 `凌际星云对比雅升 VITO`、`瑞虎 9 与 RAV4 如何选` 这类短 query 对比题，它们的 prompt 并不比长 query 短——**长度瓶颈在 schema，不在用户输入**。这意味着优化上下文的首选动作是裁 schema（分场景发放、惰性加载），而不是卷 query 压缩。
