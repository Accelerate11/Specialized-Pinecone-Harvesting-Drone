# LocateAnything 松果微调

供以后换入高质量 YOLO 数据集后复用。训练在 Win11 的 WSL2 `ld666` 中进行，不能直接拿 GGUF 训练。

## 固定环境

- Eagle：`/home/ld666/projects/Eagle`，commit `783f656d127ee498137b5ff52603ce36c292d317`
- Python 环境：`/home/ld666/venvs/locateanything`
- 完整基础模型：`/home/ld666/model-cache/LocateAnything-3B`
- 训练脚本：`/mnt/e/locate4b/finetune_pipeline/scripts/train_pinecone_lora.sh`
- checkpoint 建议放在：`/home/ld666/work_dirs/locateanything_pinecone/`

当前 Eagle 已应用 WSL/SDPA 兼容修改。若以后重新克隆 Eagle，在仓库根目录执行：

```bash
git apply --check /mnt/e/locate4b/finetune_pipeline/patches/eagle_wsl_sdpa.patch
git apply /mnt/e/locate4b/finetune_pipeline/patches/eagle_wsl_sdpa.patch
```

## 1. 准备数据

推荐新数据单独放置，不覆盖旧数据：

```text
E:\locate4b\dataset_hq\
  data.yaml
  images\train\
  images\val\
  labels\train\
  labels\val\
```

标注必须是 YOLO 检测格式：每行 `class x_center y_center width height`，坐标归一化到 0–1。训练前重点确认：所有可见松果尽量完整标注、框不越界、训练集和验证集不重复、空标注图片确实没有松果。

## 2. 转换为 ShareGPT JSONL

在 Windows PowerShell 中执行：

```powershell
python E:\locate4b\finetune_pipeline\scripts\convert_yolo_to_locateanything.py `
  --source E:\locate4b\dataset_hq `
  --output E:\locate4b\dataset_hq_true `
  --wsl-output-root /mnt/e/locate4b/dataset_hq_true
```

检查 `dataset_hq_true\conversion_report.json`，确认样本数、框数、类别和错误记录。训练使用自动生成的 `dataset_hq_true\recipe_train.json`。

## 3. 一步拉烟测试

每次换数据后必须先跑 1 步，确认数据、显存和保存流程正常：

```powershell
wsl.exe -d ld666 -- bash -lc 'META_PATH=/mnt/e/locate4b/dataset_hq_true/recipe_train.json OUTPUT_DIR=/home/ld666/work_dirs/locateanything_pinecone/smoke_hq MAX_STEPS=1 SAVE_STEPS=1 GRADIENT_ACC=1 bash /mnt/e/locate4b/finetune_pipeline/scripts/train_pinecone_lora.sh'
```

`OUTPUT_DIR` 必须使用新的空目录。

## 4. 正式微调

先跑 100 步小规模实验，不要在新数据上直接长时间训练：

```powershell
wsl.exe -d ld666 -- bash -lc 'META_PATH=/mnt/e/locate4b/dataset_hq_true/recipe_train.json OUTPUT_DIR=/home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1 MAX_STEPS=100 SAVE_STEPS=50 bash /mnt/e/locate4b/finetune_pipeline/scripts/train_pinecone_lora.sh'
```

默认配置是 BF16、SDPA、LoRA rank 32、冻结视觉骨干和 MLP、上下文 4096。正式步数应根据新数据规模和验证结果决定，不要只看训练 loss。

## 5. 评估

基础模型和新 checkpoint 必须使用相同样本、相同 seed、相同 `slow` 解码分别评估：

```powershell
wsl.exe -d ld666 -- bash -lc 'source /home/ld666/venvs/locateanything/bin/activate && python /mnt/e/locate4b/finetune_pipeline/scripts/evaluate_pinecone.py --model /home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1 --annotation /mnt/e/locate4b/dataset_hq_true/annotations/pinecone_val.jsonl --image-root /mnt/e/locate4b/dataset_hq_true/images --output /mnt/e/locate4b/results/eval_lora_hq_v1.json --limit 0 --seed 42 --generation-mode slow'
```

关注 Precision、Recall、F1、非法框数量和实际叠加图。只有验证集优于基础模型，才进入合并和部署转换。

## 6. 合并 LoRA

```powershell
wsl.exe -d ld666 -- bash -lc 'source /home/ld666/venvs/locateanything/bin/activate && python /mnt/e/locate4b/finetune_pipeline/scripts/merge_locateanything_lora.py --input /home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1 --output /home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1_merged'
```

输出目录必须为空。

## 7. 转为 GGUF 并量化

```bash
source /home/ld666/venvs/locateanything/bin/activate
mkdir -p /home/ld666/work_dirs/locateanything_pinecone/gguf_hq_v1

python /home/ld666/projects/llama.cpp-locateanything/convert_hf_to_gguf.py \
  /home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1_merged \
  --outfile /home/ld666/work_dirs/locateanything_pinecone/gguf_hq_v1/LocateAnything-Pinecone-BF16.gguf \
  --outtype bf16

python /home/ld666/projects/llama.cpp-locateanything/convert_hf_to_gguf.py \
  /home/ld666/work_dirs/locateanything_pinecone/lora_hq_v1_merged \
  --outfile /home/ld666/work_dirs/locateanything_pinecone/gguf_hq_v1/LocateAnything-Pinecone-BF16.gguf \
  --outtype bf16 --mmproj

/home/ld666/projects/llama.cpp-locateanything/build/bin/llama-quantize \
  /home/ld666/work_dirs/locateanything_pinecone/gguf_hq_v1/LocateAnything-Pinecone-BF16.gguf \
  /home/ld666/work_dirs/locateanything_pinecone/gguf_hq_v1/LocateAnything-Pinecone-Q4_K_M.gguf \
  Q4_K_M
```

最后再用 `evaluate_gguf_pinecone.py` 对原始 Q4 和新 Q4 做同口径评估，通过后复制到 Jetson。

## 当前原型结果

固定 12 张低质量验证样本、llama.cpp Q4_K_M、`slow` 解码：原始模型 IoU@0.5 F1 为 0.7671，当前微调模型为 0.8010。该结果只证明流程可行，不能代替高质量数据上的重新评估。
