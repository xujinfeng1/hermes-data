# AnimateDiff SD1.5 工作流踩坑记录

## 已验证可用的工作流拓扑

```
CheckpointLoaderSimple → AnimateDiffLoaderV1 → KSampler → VAEDecode → VHS_VideoCombine
                              ↑                    ↑
                        EmptyLatentImage      CLIPTextEncode (pos/neg)
```

## 关键参数名

| 节点 | 参数 | 正确写法 |
|------|------|---------|
| AnimateDiffLoaderV1 | 模型文件名 | `model_name` (不是 `ckpt_name`) |
| AnimateDiffLoaderV1 | 输出 [0] | MODEL (给 KSampler model) |
| AnimateDiffLoaderV1 | 输出 [1] | LATENT (给 KSampler latent_image) |
| VHS_VideoCombine | 必填 | `pingpong: false` (否则 400) |
| AnimateDiffLoaderV1 | 额外参数 | `unlimited_area_hack: false`, `beta_schedule: "sqrt_linear (AnimateDiff)"` |

## ADE 节点（废弃方案）

`ADE_LoadAnimateDiffModel` + `ADE_ApplyAnimateDiffModelSimple` 配合失败：
- KSampler 期望 MODEL 类型，ADE 输出 M_MODELS 类型不匹配
- 正确做法是用 `AnimateDiffLoaderV1`，它内部完成了模型包裹

## 模型文件

SD1.5 checkpoint: `sd_v1-5.safetensors` (~4GB)
AnimateDiff 运动模块: `mm_sd_v15_v2.ckpt` (~1.7GB)
支持 `.safetensors` 和 `.ckpt` 两种格式

## 生成时间

M1 Max 32GB, 512×512, 16帧, 15步: 约 4-6 分钟
