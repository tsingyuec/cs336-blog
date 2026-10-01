# Stanford CS336《从头构建大语言模型》· 博客与文稿

> 视频来源：哔哩哔哩《【2026年最新版极致中配】Stanford CS336: Language Modeling from Scratch》
> BV 号：`BV11LEA6eEuj`（https://www.bilibili.com/video/BV11LEA6eEuj/）
> 本仓库包含 **18 篇中文技术博客** 与 **18 篇「图-字幕」原始文稿**。

## 项目简介

Stanford CS336《Language Modeling from Scratch（从头构建大语言模型）》2026 中配版的图文整理。
本仓库把每个分 P 的视频逐秒抽帧、抓取 B 站 AI 字幕，先做成可核查的「图-字幕」原始文稿，
再据此写成初学者友好的金字塔式技术博客。所有观点与知识点均来自视频字幕本身。

本仓库的整理流程采用开源技能 **[video2blog-skill](https://github.com/tsingyuec/video2blog-skill)** 完成（把 B 站 / YouTube 视频自动整理成中文技术博客），欢迎使用与关注。

## 博客与原始文稿索引

| # | 标题 | 博客 | 原始文稿 | 内容概要 |
| --- | --- | --- | --- | --- |
| 01 | Lecture 1: 课程概览与分词 | [bl01](<blog/Lecture 1 课程概览与分词.md>) | [p01](<transcripts/Lecture 1 课程概览与分词.md>) | 课程理念、"苦涩的教训"、语言模型发展史、五大板块与分词/BPE 详解。 |
| 02 | Lecture 2: PyTorch(einops) | [bl02](<blog/Lecture 2 PyTorch(einops).md>) | [p02](<transcripts/Lecture 2 PyTorch(einops).md>) | 张量存储与精度、einops、FLOPs 核算、MFU、算术强度与屋顶线、`6ND`、显存优化。 |
| 03 | Lecture 3: Architectures | [bl03](<blog/Lecture 3 Architectures.md>) | [p03](<transcripts/Lecture 3 Architectures.md>) | pre/post-norm、RMSNorm、GLU、并行块、RoPE、QK-norm/soft-cap、MQA/GQA/滑动窗口。 |
| 04 | Lecture 4: Attention Alternatives | [bl04](<blog/Lecture 4 Attention Alternatives.md>) | [p04](<transcripts/Lecture 4 Attention Alternatives.md>) | 线性注意力、Mamba-2/Gated DeltaNet、DeepSeek 稀疏注意力、MoE 路由与负载均衡。 |
| 05 | Lecture 5: GPUs, TPUs | [bl05](<blog/Lecture 5 GPUs, TPUs.md>) | [p05](<transcripts/Lecture 5 GPUs, TPUs.md>) | GPU vs CPU 设计哲学、SM 与内存层次、TPU、屋顶线、六种加速技巧、FlashAttention。 |
| 06 | Lecture 6: Kernels, Triton, XLA | [bl06](<blog/Lecture 6 Kernels, Triton, XLA.md>) | [p06](<transcripts/Lecture 6 Kernels, Triton, XLA.md>) | warp/占用率/bank 冲突/合并访存/波量化、profiling、Triton 内核实例与 PTX。 |
| 07 | Lecture 7: Parallelism | [bl07](<blog/Lecture 7 Parallelism.md>) | [p07](<transcripts/Lecture 7 Parallelism.md>) | 集合通信原语、NVLink/InfiniBand/RDMA、torch.distributed、数据/张量/流水线并行。 |
| 08 | Lecture 8: Parallelism | [bl08](<blog/Lecture 8 Parallelism.md>) | [p08](<transcripts/Lecture 8 Parallelism.md>) | ZeRO/FSDP、流水线气泡、张量/专家并行、激活内存、4D 并行与真实训练案例。 |
| 09 | Lecture 9: Scaling Laws | [bl09](<blog/Lecture 9 Scaling Laws.md>) | [p09](<transcripts/Lecture 9 Scaling Laws.md>) | 缩放律历史、数据缩放、幂律直觉、临界批次大小、学习率缩放、Kaplan vs Chinchilla。 |
| 10 | Lecture 10: Inference | [bl10](<blog/Lecture 10 Inference.md>) | [p10](<transcripts/Lecture 10 Inference.md>) | 预填充 vs 生成、KV 缓存压缩、量化剪枝、推测解码、连续批处理、分页注意力。 |
| 11 | Lecture 11: Scaling Laws | [bl11](<blog/Lecture 11 Scaling Laws.md>) | [p11](<transcripts/Lecture 11 Scaling Laws.md>) | muP 初始化与 WSD、DeepSeek 网格搜索、优化器规模依赖、Muon 与 muP 理论。 |
| 12 | Lecture 12: Evaluation | [bl12](<blog/Lecture 12 Evaluation.md>) | [p12](<transcripts/Lecture 12 Evaluation.md>) | 从困惑度到考试/聊天/智能体/安全基准、生态效度与数据污染。 |
| 13 | Lecture 13: Data (Sources, Datasets) | [bl13](<blog/Lecture 13 Data (Sources, Datasets).md>) | [p13](<transcripts/Lecture 13 Data (Sources, Datasets).md>) | 爬取约束、robots.txt/服务条款、版权与合理使用、BERT→Common Pile 演进。 |
| 14 | Lecture 14: Data | [bl14](<blog/Lecture 14 Data.md>) | [p14](<transcripts/Lecture 14 Data.md>) | HTML/PDF 转换、过滤、MinHash/LSH 去重、UniMax/RegMix 混合、代码合成数据。 |
| 15 | Lecture 15: Mid/Post-Training | [bl15](<blog/Lecture 15 Mid-Post-Training.md>) | [p15](<transcripts/Lecture 15 Mid-Post-Training.md>) | SFT 数据演进与坑、指令微调融入预训练、RLHF 数据、PPO/DPO 推导与过度优化。 |
| 16 | Lecture 16: Post-Training - RLVR | [bl16](<blog/Lecture 16 Post-Training - RLVR.md>) | [p16](<transcripts/Lecture 16 Post-Training - RLVR.md>) | PPO 痛点、GRPO、DeepSeek R1、Kimi K1.5、Qwen3、数据课程与奖励钻空子。 |
| 17 | Lecture 17: Alignment - Multimodality | [bl17](<blog/Lecture 17 Alignment - Multimodality.md>) | [p17](<transcripts/Lecture 17 Alignment - Multimodality.md>) | CLIP→SigLIP→LLaVA/Qwen-VL→Chameleon、视觉编码与对齐、M-RoPE。 |
| 18 | Lecture 18: Guest Lecture Dan Fu | [bl18](<blog/Lecture 18 Guest Lecture Dan Fu.md>) | [p18](<transcripts/Lecture 18 Guest Lecture Dan Fu.md>) | token 生命周期、连续批处理、KV 缓存 bug、CDP、MegaKernel 与 Parcae。 |
| 19 | Guest Lecture: Dan Fu | — | [p19](<transcripts/Guest Lecture Dan Fu.md>) | 与第 18 讲为同一场 Dan Fu 客座讲座的重复分 P，不单独成篇博客。 |

> 合并稿见 [`blog/CS336_全课博客合集.md`](blog/CS336_全课博客合集.md)。

## 目录结构

```
cs336_blog/
├─ README.md                     # 本文件：博客与文稿索引
├─ blog/
│  ├─ CS336_全课博客合集.md               # 18 篇博客合并稿
│  ├─ Lecture 1 课程概览与分词.md …      # 各讲博客（按原始标题命名，共 18 篇）
│  └─ assets/pNN/                        # 各讲博客配图
├─ transcripts/
│  ├─ Lecture 1 课程概览与分词.md …      # 「图-字幕」原始文稿（按原始标题命名，共 19 篇）
│  └─ img/pNN/                           # 文稿保留的代表帧
└─ subs/                         # 抓取到的字幕（.srt / .txt）
```

## 说明

- 文中所有观点与知识点均来自视频字幕本身；本仓库仅用于个人学习与笔记整理。
- 博客配图来源于视频抽帧，版权归原视频/课程作者所有。
