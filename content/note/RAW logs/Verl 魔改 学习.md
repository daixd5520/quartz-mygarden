---
tags:
  - landing
  - RL
  - verl
draft: "true"
---

这里存的是 verl reward plugin 里 reply 部分的打分代码，学习/魔改用。原文里混了很多注释掉的旧版 `compute_score`，留着看思路没问题，但太干扰阅读，这里只保留当前线上跑的 `compute_score_parallel` 和它的尾巴 `__main__`。

## 入口与设计

- 文件路径：`verl/utils/reward_score/plugins/reply/reply_eval.py`
- 四个子打分器：`reply_cards_eval` / `reply_acc`（当 RAG 遵循性）/ `reply_useful` / `reply_instruct`。另有一个 `reply_auto` 的老版本，和上面三合一重叠，已经注释掉。
- 并发策略：`ThreadPoolExecutor(max_workers=10)`，四路打分并发触发，每路都通过 `compute_score_call` 包一层超时+重试+兜底默认值，单个子分器挂了不会阻塞整体。
- 最终 `score` 按等权融合四路分 + 一路 rule 分（banned_str 命中置 0）；任一子分归零则总分直接置 0，避免"其他维度高分拉平硬伤"。

## 代码

```python fold file:verl/verl/utils/reward_score/plugins/reply/reply_eval.py
import os
import re
import sys
import threading

current_file = __file__
current_dir = os.path.dirname(os.path.abspath(current_file))
sys.path.append(current_dir)

from concurrent.futures import ThreadPoolExecutor, as_completed
from multiprocessing import Pool

from prompts import PROMPTS
from reply_acc import compute_score as acc_compute_score
from reply_cards_eval import compute_score as cards_compute_score
from reply_instruct import compute_score as instruct_compute_score
from reply_useful import compute_score as useful_compute_score


def compute_score_call(func, *args, timeout: float = 200.0, max_retries: int = 3, default={"score": 0.5}):
    """单个子打分器的安全壳：超时 / 异常 / 返回 None 都会重试，全失败返回 default。"""
    data_source = args[0]
    for attempt in range(1, max_retries + 1):
        result_box = {}

        def target():
            try:
                result_box['val'] = func(*args)
            except Exception as e:
                result_box['err'] = e

        t = threading.Thread(target=target, daemon=True)
        t.start()
        t.join(timeout)

        if t.is_alive() or 'val' not in result_box or 'err' in result_box or result_box['val'] is None:
            print(f"Warning: {data_source} [compute_score_call] re-attempt {attempt}/{max_retries}")
            result_box = {}
            continue
        return result_box['val']

    print(f"error: {data_source} [compute_score_call] failed after {max_retries} attempts, return default={default!r}")
    return default


def compute_score_parallel(data_source, solution_str, ground_truth, extra_info={}):
    try:
        # <think>...</think> 只留回答段，避免推理泄漏影响打分
        if "<think>" in solution_str:
            solution_str = solution_str.split("</think>")[1]

        return_d = {}

        with ThreadPoolExecutor(max_workers=10) as executor:
            r_card = executor.submit(compute_score_call, cards_compute_score, data_source, solution_str, ground_truth, extra_info)
            r_auto_rag = executor.submit(compute_score_call, acc_compute_score, "RAG遵循性", solution_str, ground_truth, extra_info)
            r_auto_useful = executor.submit(compute_score_call, useful_compute_score, "信息有用性", solution_str, ground_truth, extra_info)
            r_auto_instruct = executor.submit(compute_score_call, instruct_compute_score, "指令遵循性", solution_str, ground_truth, extra_info)

            for key, future in [
                ("score_card", r_card),
                ("score_auto_rag", r_auto_rag),
                ("score_auto_useful", r_auto_useful),
                ("score_auto_instruct", r_auto_instruct),
            ]:
                try:
                    reward = future.result(timeout=600)
                    if "score" in reward:
                        return_d[key] = reward["score"] if reward["score"] is not None else 0.5
                except Exception as e:
                    print(f"{key} error:", e)

        # 禁止字符：一旦命中规则分归零，下面会把总分也拉到 0
        banned_str = [
            "权威信息", "权威资料", "^未知", "_未知", "toolcall", "serieslink",
            "carreputation", "提及", "检索信息", "信息说明", "网页", "卡片", "未明确", "标注",
        ]
        score_rule = 1.0
        for bs in banned_str:
            if bs in solution_str:
                score_rule = 0.0
                break
        return_d["score_rule"] = score_rule

        weights = {
            "score_card":          0.2,
            "score_auto_rag":      0.2,
            "score_auto_useful":   0.2,
            "score_auto_instruct": 0.2,
            "score_rule":          0.2,
        }

        final_score = 0.0
        weights_all = 0.0
        zero_sub_score = False
        for k, v in return_d.items():
            if k in weights:
                weights_all += weights[k]
                final_score += v * weights[k]
                if v == 0.0:
                    zero_sub_score = True
        return_d["score"] = final_score / weights_all

        # 任一子分归零 → 总分置 0，避免通过其他维度高分"蒙混过关"
        if zero_sub_score:
            return_d["score"] = 0.0

        return return_d

    except Exception as e:
        print("多进程错误？ error:", e)
        return {"score": 0.5}


if __name__ == "__main__":
    import pandas as pd
    df = pd.read_csv("/dcar_ai_vepfs/chenyu.jojoychen/verl/data_preprocess/test/GRPO强化对比.csv")
    df = df[df["answer1"].notna()]
    df = df[["log_id", "query", "system_prompt", "final_prompt1", "answer1"]]
    row = df[df["query"] == "车载充电头超级快充"].head(1)
    res = compute_score_parallel("", df["answer1"], "", {
        "system_prompt": df["system_prompt"],
        "user_prompt": df["final_prompt1"],
    })
    print(res)
```

## 几个会踩坑的点

1. `compute_score_call` 用 `threading.Thread` + `t.join(timeout)` 实现超时：Python 线程是没法真正 kill 的，超时后 daemon 线程还在后台跑，高并发下可能积累僵尸线程；短期可以接受，长期建议换成 `multiprocessing.Process` + `terminate()` 或者 subprocess 隔离。
2. 默认兜底值是 `0.5`，对 GRPO 这种"相对优势"训练来说是中性的；别改成 `1.0`，否则 judge 挂了会被当成高质量样本奖励。
3. `banned_str` 里的 `"^未知"` / `"_未知"` 是字面匹配，不是正则，注意别当成 regex 维护。
4. `if zero_sub_score: return_d['score'] = 0.0` 这条短路规则非常关键——它让四路判分里任何一个硬性违规能一票否决，和常见的"加权平均"思路明确区分。
