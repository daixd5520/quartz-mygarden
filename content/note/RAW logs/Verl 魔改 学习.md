cy权威网页部分
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

# from reply_auto import compute_score as auto_compute_score

from reply_cards_eval import compute_score as cards_compute_score

from reply_instruct import compute_score as instruct_compute_score

from reply_useful import compute_score as useful_compute_score

  

# def compute_score(data_source, solution_str, ground_truth, extra_info={}):

# try:

# if "<think>" in solution_str:

# print("answer with <think> is 0:", solution_str)

# return 0.

# reward_card = cards_compute_score(data_source, solution_str, ground_truth, extra_info)

# reward_auto_rag = auto_compute_score("RAG遵循性", solution_str, ground_truth, extra_info)

# reward_auto_useful = auto_compute_score("信息有用性", solution_str, ground_truth, extra_info)

# reward_auto_logic = auto_compute_score("逻辑一致性", solution_str, ground_truth, extra_info)

# reward_auto_instruct = auto_compute_score("指令遵循性", solution_str, ground_truth, extra_info)

# try:

# score_card = reward_card["score"] if "score" in reward_card else 1.0

# score_auto_rag = reward_auto_rag["score"] if "score" in reward_auto_rag else 1.0

# score_auto_useful = reward_auto_useful["score"] if "score" in reward_auto_useful else 1.0

# score_auto_logic = reward_auto_logic["score"] if "score" in reward_auto_logic else 1.0

# score_auto_instruct = reward_auto_instruct["score"] if "score" in reward_auto_instruct else 1.0

  

# final_score = 0.2 * score_card + 0.3 * score_auto_rag + 0.1 * score_auto_useful + 0.1 * score_auto_logic + 0.3 * score_auto_instruct

# if score_card==0.:

# final_score = min(final_score, 0.5)

  

# return final_score

# # return {

# # "score_card": score_card,

# # "score_auto_rag": score_auto_rag,

# # "score_auto_useful": score_auto_useful,

# # "score_auto_logic": score_auto_logic,

# # "score_auto_instruct": score_auto_instruct,

# # "score": final_score

# # }

# except TimeoutError as e:

# print("超时了 error:", e)

# return 1.0

# except Exception as e:

# print("执行出错 error:", e)

# return 1.0

# except Exception as e:

# print("error:",e)

# return 1.0

  

def compute_score_call(func, *args, timeout: float = 200.0, max_retries: int = 3, default={"score":0.5},):

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

print(f"Warning: {data_source} [compute_score_call] re-attempt {attempt}/{max_retries} ")

result_box = {}

continue

return result_box['val']

print(f"error: {data_source} [compute_score_call] failed after {max_retries} attempts, return default={default!r}")

return default

  

def compute_score_parallel(data_source, solution_str, ground_truth, extra_info={}):

try:

if "<think>" in solution_str:

# print("answer with <think> is 0:", solution_str)

# return {"score": 0.5}

solution_str = solution_str.split("</think>")[1]

return_d = {}

with ThreadPoolExecutor(max_workers=10) as executor:

r_card = executor.submit(compute_score_call, cards_compute_score, data_source, solution_str, ground_truth, extra_info)

r_auto_rag = executor.submit(compute_score_call, acc_compute_score, "RAG遵循性", solution_str, ground_truth, extra_info)

r_auto_useful = executor.submit(compute_score_call, useful_compute_score, "信息有用性", solution_str, ground_truth, extra_info)

r_auto_instruct = executor.submit(compute_score_call, instruct_compute_score, "指令遵循性", solution_str, ground_truth, extra_info)

try:

reward_card = r_card.result(timeout=600)

if 'score' in reward_card:

return_d["score_card"] = reward_card["score"] if reward_card["score"] is not None else 0.5

except Exception as e:

print("reward_card error:", e)

try:

reward_auto_rag = r_auto_rag.result(timeout=600)

if 'score' in reward_auto_rag:

return_d["score_auto_rag"] = reward_auto_rag["score"] if reward_auto_rag["score"] is not None else 0.5

except Exception as e:

print("reward_auto_rag error:", e)

try:

reward_auto_useful = r_auto_useful.result(timeout=600)

if 'score' in reward_auto_useful:

return_d["score_auto_useful"] = reward_auto_useful["score"] if reward_auto_useful["score"] is not None else 0.5

except Exception as e:

print("reward_auto_useful error:", e)

try:

reward_auto_instruct = r_auto_instruct.result(timeout=600)

if 'score' in reward_auto_instruct:

return_d["score_auto_instruct"] = reward_auto_instruct["score"] if reward_auto_instruct["score"] is not None else 0.5

except Exception as e:

print("reward_auto_instruct error:", e)

# if "out_len" in extra_info:

# out_len = extra_info["out_len"]

# # print("eval out_len:", out_len)

# else:

# out_len = len(solution_str)

# print("error: no out_len token input")

# reward_out_len = 1.0

# if out_len>512:

# reward_out_len = 1.0 - (out_len - 512) / (512*2.)

# reward_out_len = reward_out_len ** 2

# elif out_len>512*3:

# reward_out_len = 0.

# return_d["score_out_len"] = reward_out_len

# 禁止字符

banned_str = ["权威信息", "权威资料", "^未知", "_未知", 'toolcall', 'serieslink',

'carreputation', '提及', '检索信息', '信息说明', '网页', '卡片', '未明确', '标注']

score_rule = 1.

for bs in banned_str:

if bs in solution_str:

score_rule=0.

break

return_d["score_rule"] = score_rule

weights = {

"score_card": 0.2,

"score_auto_rag": 0.2,

"score_auto_useful": 0.2,

"score_auto_instruct": 0.2,

"score_rule":0.2

}

final_score = 0.

weights_all = 0.

zero_sub_score = False

for k, v in return_d.items():

if k in weights:

weights_all += weights[k]

final_score += v * weights[k]

if v == 0.:

zero_sub_score = True

return_d['score'] = final_score/weights_all

# 遇到严重问题直接置为0分

if zero_sub_score: return_d['score'] = 0.

return return_d

except Exception as e:

print("多进程错误？ error:",e)

return {"score": 0.5}

  

if __name__ == "__main__":

import pandas as pd

df = pd.read_csv("/dcar_ai_vepfs/chenyu.jojoychen/verl/data_preprocess/test/GRPO强化对比.csv")

df = df[df['answer1'].notna()]

df = df[["log_id", "query", "system_prompt", "final_prompt1", "answer1"]]

row = df[df["query"]=="车载充电头超级快充"].head(1)

res = compute_score_parallel("", df['answer1'], "", {

"system_prompt":df['system_prompt'],

"user_prompt":df['final_prompt1'],

})

print(res)

```

