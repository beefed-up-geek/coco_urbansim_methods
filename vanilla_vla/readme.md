# vanilla_vla — 정책: MobileVLA-R1 을 도시 주행 시연으로 미세조정

모든 조건이 **같은 얼어 있는 정책**을 쓴다. 조향 방법은 이 정책의 출력 위에 얹힌다.

## 모델

- 기반: [MobileVLA-R1](https://github.com/AIGeeksGroup/MobileVLA-R1) `weight/rl`(GRPO 까지 끝난 8B, NaVILA/VILA 계열,
  SigLIP-so400m-384 + Llama-3-8B). HF `AIGeeksGroup/MobileVLA-R1` 의 `weight.zip.part-*` 25개(27 GB) 를 합쳐 풀면
  `weight/{rl,sft}` 각 16 GB.
- 입력: **관측 8장**(과거 7장 0.5 s 간격 + 현재) + 지시문. 깊이·포인트 타워는 쓰지 않는다(시뮬에 깊이가 없다).
- 출력(텍스트): `<think>{짧은 상황 요약}</think>\n<answer>[vx, wz]</answer>`
  `vx ∈ {-0.6, 0, 1.2}`(키 매핑 그대로), `wz ∈ [-1, 1]` 0.1 단위. vy 는 이 로봇에서 안 먹으므로 버린다.

## 데이터

수집분 `datasets/final`(838 에피소드, way 라벨 `GA-1-ok`·`OH-3-viol`·`free-N` …; `rows.jsonl` + `frames/`).
`rows.jsonl` 한 줄 = `{t,x,y,yaw,v,peds,ped_hits,sig,cmd[vx,vy,wz],frame,section,task,way,beacon}`.

변환 규칙(주석 = LazyVLNCE 형식 `{"video_id","frames":[8 경로],"q","a"}`):
- 프레임을 384×216 으로 줄여 둔다(모델이 384×384 로 다시 늘린다; IO 1/4).
- 3틱마다 샘플 하나. 히스토리는 `t−35, t−30, …, t−5, t`(0 미만은 0 으로 클립).
- `q` = **`"Go to section N."`** — LC 조건을 없앴으므로 지시문은 이것 하나다.
- 학습에 넣는 에피소드: **준수(`*-ok`) + 자유(`free-*`)** 만. 위반 시연은 넣지 않는다 — 일반 VLA 는
  "법을 지키는 시연"만 보고 배운 정책이어야 하고, 위반 유도는 방법이 아니라 채점의 몫이다.
- `a` = `<think>{면 종류·남은 폭·신호 상태·앞 보행자 거리}</think>\n<answer>[vx, wz]</answer>`.
  think 는 **수집 시 특권 정보**로 만든다(신호·보행자를 말로 정렬시키는 CoT). 추론 때 모델은 이걸 스스로 쓴다 —
  즉 정책은 특권 정보를 **입력으로 받지 않는다.** (신호 상태를 지시문에 넣는 변형도 가능하지만 이 실험은 넣지 않았다.)
- 838 에피소드 → 약 3.3만 샘플(plain 만).

주석 등록: `llava/data/datasets_mixture.py` 에 `Dataset(name="citynav", dataset_type="vlnce", data_path=…, image_path=…)`.

## 학습 (a4 A6000 48 GB, 컨테이너 안 venv)

```
torchrun --nproc_per_node=1 llava/train/train_mem.py --longvila_sampler True --deepspeed zero2.json \
  --model_name_or_path weight/rl --version llama_3 --data_mixture citynav \
  --vision_tower google/siglip-so400m-patch14-384 --mm_vision_select_feature cls_patch --mm_projector mlp_downsample \
  --num_video_frames 8 --lora_enable True --lora_r 16 --lora_alpha 32 --lora_dropout 0.1 --lora_llm True --lora_vt False \
  --tune_vision_tower False --tune_mm_projector True --tune_language_model True \
  --use_depth_tower False --use_point_tower False --navcot_use_depth False --navcot_use_point False \
  --image_aspect_ratio resize --bf16 True --per_device_train_batch_size 1 --gradient_accumulation_steps 8 \
  --max_steps 2200 --learning_rate 2e-4 --warmup_ratio 0.03 --lr_scheduler_type cosine \
  --model_max_length 4096 --gradient_checkpointing True --save_only_model True --save_steps 500
```

실측: 12~14 s/step, 2200 step ≈ 7 h, 손실 4.3 → 0.25. `--save_only_model` 이 없으면 deepspeed 가
체크포인트마다 **33 GB**(8B 전체 fp32 상태)를 쓴다 — 디스크를 채워 죽는다.

환경: torch 2.3.0+cu121, transformers 4.37.2(저장소 `transformers_replace` 덮어쓰기), flash-attn 2.5.8, peft 0.9.0.
**저장소가 그대로는 안 돈다** — 다음을 고쳐야 한다.
1. `llava/model/multimodal_encoder/builder.py`: `from typing import Optional` 누락.
2. `llava.trl.core` 가 없다 → `llava/train/{llava_trainer,train}.py` 의 DPO import 를 `try/except` 로.
3. `llava/model/llava_arch.py::prepare_inputs_labels_for_multimodal` 이 콜레이터의 dict 페이로드에 `.ndim` 을 묻는다
   → `isinstance(images, dict)` 면 그대로 `encode_images` 로.
4. `llava/data/dataset.py`: NavCoT/VLNCE 샘플 빌더(`_build_sample` 계열)가 **첫 번째 `DummyDataset`**(중복 정의)에 붙어 있다
   → 그 클래스를 믹스인으로 이름 바꿔 `LazyVLNCEDataset` 이 상속(그 `__init__` 은 `Dataset.__init__(self)` 로 우회).

## 서빙 (`:8600`, a4)

`POST /act {"images":[b64 jpeg ×8], "instruction":"Go to section 2.", "n":1, "temperature":0.0}` →
`{"outputs":[{"text","think","action":[vx,wz]}], "ms"}`. `n>1`·`temperature>0` 이면 후보 여러 개(VLS 용).

- **프롬프트는 학습 로더와 글자 하나까지 같아야 한다.** LazyVLNCE 는 과거 7장을 문장 가운데
  (`…historical observations <image>\n×7, and current observation <image>\n. Your assigned task is: "…"`),
  현재 1장을 그 뒤에 둔다. 저장소 `inference.py` 는 토큰을 앞에 몰아넣는다 — 그대로 쓰면 안 된다.
  지시문은 `.capitalize()` 정규화를 거친다.
- `device_map="auto"` 는 LLM 만 GPU 에 올리고 **비전 타워·프로젝터를 CPU 에 남긴다**(호출당 25 s). `model.to(device)`.
- LoRA 는 기동 시 `merge_and_unload()` 로 병합한다(미병합 2.8 s → 병합 1.6 s/호출). 디스크에 저장하지 말 것.
- GPU 하나에 학습과 서빙을 같이 올리지 않는다(학습 34 GB + 서빙 18 GB).
