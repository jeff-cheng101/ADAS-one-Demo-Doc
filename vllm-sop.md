# vLLM 效能測試 SOP

本文件提供「從空白環境到完成效能驗證」的標準流程。

## 1. 測試目標與門檻

- 情境：Input 2,048 tokens / Output 128 tokens
- Concurrency：512
- 模型：`mistralai/Mistral-7B-Instruct-v0.3`（FP16，無量化）
- 資料集：ShareGPT

驗收門檻：

1. 解碼吞吐量（Output token throughput）>= 3,000 tok/s
2. 單請求平均速度 >= 10 tok/s
3. RPM >= 1,400 req/min

---

## 2. 前置檢查

```bash
nvidia-smi
lscpu
free -h
python3 --version
```

確認重點：

- GPU 可被偵測
- Python 可用
- 記憶體與 CPU 規格符合需求

---

## 3. 安裝環境

```bash
apt-get update && apt-get install -y python3-pip python3-venv

python3 -m venv /root/vllm-bench-venv
/root/vllm-bench-venv/bin/python -m pip install -U pip setuptools wheel
/root/vllm-bench-venv/bin/pip install vllm
```

（可選）下載 InferenceX（InferenceMAX 後繼）：

```bash
git clone https://github.com/SemiAnalysisAI/InferenceX.git /root/InferenceX
```

---

## 4. 準備資料集

```bash
wget -O /root/ShareGPT_V3_unfiltered_cleaned_split.json \
  https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
```

確認檔案存在：

```bash
python3 -c "import os; p='/root/ShareGPT_V3_unfiltered_cleaned_split.json'; print(os.path.exists(p), os.path.getsize(p))"
```

---

## 5. 啟動 vLLM 服務

```bash
nohup /root/vllm-bench-venv/bin/vllm serve mistralai/Mistral-7B-Instruct-v0.3 \
  --host 127.0.0.1 --port 8000 --dtype half --max-model-len 2176 \
  --gpu-memory-utilization 0.95 --max-num-batched-tokens 4096 --max-num-seqs 512 \
  > /root/vllm-serve.log 2>&1 &
```

等待服務就緒：

```bash
for i in $(seq 1 120); do
  if curl -s http://127.0.0.1:8000/v1/models >/tmp/vllm_models.json 2>/dev/null; then
    echo READY
    cat /tmp/vllm_models.json
    break
  fi
  sleep 5
done
```

---

## 6. 執行 ShareGPT 壓測

```bash
/root/vllm-bench-venv/bin/vllm bench serve \
  --backend vllm \
  --model mistralai/Mistral-7B-Instruct-v0.3 \
  --base-url http://127.0.0.1:8000 \
  --dataset-name sharegpt \
  --dataset-path /root/ShareGPT_V3_unfiltered_cleaned_split.json \
  --num-prompts 2048 \
  --max-concurrency 512 \
  --input-len 2048 \
  --output-len 128 \
  --sharegpt-output-len 128 \
  --save-result \
  --result-dir /root/bench-results \
  --result-filename mistral7b_fp16_sharegpt_isl2048_osl128_conc512
```

---

## 7. 指標計算與門檻比對

### 7.1 直接看 CLI 指標

主要欄位：

- `Output token throughput (tok/s)` → 解碼吞吐量
- `Request throughput (req/s)` → 乘以 60 得 RPM
- `Mean TPOT (ms)` → 單請求平均速度約為 `1000 / Mean TPOT`

### 7.2 以結果檔自動計算

```bash
python3 - <<'PY'
import json

f = '/root/bench-results/mistral7b_fp16_sharegpt_isl2048_osl128_conc512'
d = json.load(open(f))

decode_tps = d['output_token_throughput']
req_s = d['request_throughput']
rpm = req_s * 60
mean_tpot_ms = d['mean_tpot_ms']
single_req_tps = 1000 / mean_tpot_ms

print('decode_tps=', decode_tps)
print('single_req_tps=', single_req_tps)
print('rpm=', rpm)

print('PASS_decode=', decode_tps >= 3000)
print('PASS_single=', single_req_tps >= 10)
print('PASS_rpm=', rpm >= 1400)
PY
```

---

## 8. 結果歸檔與報告

建議至少保留：

- 結果檔：`/root/bench-results/*`
- 服務日誌：`/root/vllm-serve.log`
- 報告文件：`/root/0226.md`、`/root/sop.md`

---

## 9. 常見問題

1. 服務起不來（OOM）
   - 降低 `--max-num-seqs`
   - 降低 `--gpu-memory-utilization`
   - 降低 `--max-num-batched-tokens`

2. InferenceX `benchmark_serving.py` 不支援 `sharegpt`
   - 該版本僅接受 `--dataset-name random`
   - ShareGPT 場景建議用 `vllm bench serve`

3. 效能未達標
   - 優先檢查是否為單卡瓶頸（VRAM、batching）
   - 嘗試硬體升級、多卡、或允許量化
