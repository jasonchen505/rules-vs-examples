# rules-vs-examples 深入学习文档

> 上游：`tinlaboratory/rules-vs-examples`（Boston University tin lab，Najoung Kim 组）
> 本 fork：`jasonchen505/rules-vs-examples`（2026-10-07 创建）
> 论文："LLMs Learn Better In-Context from Rules than from Examples"，Xiang Fu, Seungmin Cho, Yukyung Lee, Najoung Kim；arXiv:2609.03213（2026-09-02 提交），EMNLP 2026 accepted
> License：MIT
> 学习日期：2026-10-07｜状态：只读学习，main 分支未动，本文件位于 `notes/learning` 分支

---

## 1. 一句话定位

这是一个**方法论型**的论文复现仓库：它不提出新模型，而是在 5 个精心设计的"新任务"上，做 rules（自然语言规则描述）vs examples（few-shot 输入输出示例）两种 in-context 学习方式的**受控对照实验**，回答"LLM 到底更擅长从规则学，还是从例子学"。

## 2. 论文速览

**研究问题**：in-context learning 有两种主流形态——instruction following（给规则）和 few-shot prompting（给例子）。当两者描述的是**同一个底层任务**时，哪种更有效？哪些模型/任务属性会调节这个差距？

**实验设计**：5 个横跨游戏、算术、语言推理的新任务 × 3 种学习模式（rules / examples / combined）× 多个难度 × 开源权重模型（三个模型家族，含 base 与 instruct 版本）。

**核心发现**（转述，非原文引用）：

1. 模型总体上从 rules 学得比从 examples 更可靠。
2. 在 rules 之上再加 examples，或单纯增加示例数量，都没有带来一致且显著的增益。
3. Instruction tuning 放大 rules 的收益，同时不损害 examples 的能力——也就是说 examples 的能力在 instruct 模型里是保留的，只是 rules 涨得更多。
4. 出乎意料：即使在 base 模型里也没有发现 examples 的"特权效应"；在代数类任务上 rules 依然带来增益。
5. 任务属性的调节作用：任务越依赖**代数抽象与计算**，rules 优势越大；任务越需要**分布敏感性**或调用**参数知识**，rules 优势越小。

## 3. 五个任务详解

五个任务的共同设计原则：**全新、程序可生成、程序可验证**——避免预训练语料泄漏，保证 rules 和 examples 描述的是同一个任务。

### 3.1 Set Game（纸牌游戏变体）

- 玩法：9 张牌中选 3 张构成 GAME-SET。
- 每张牌有多个属性；一个 GAME-SET 要求**每个属性**上三张牌要么全相同、要么全不同（经典 Set 规则）。
- 难度梯度：
  - d1：2 属性（动物 × 生态位，各 10 种）
  - d2：3 属性（+ 食物，10 种）
  - d3：数字 + 3 属性，且**数字属性的规则是反转的**——数字要求"两张相同、一张不同"才算合法。这个反转规则是点睛之笔：它与通用规则直接冲突，纯靠 examples 很难归纳出来，是 rules-vs-examples 对照的绝佳试金石。
- 输出格式：`\boxed{}` 包裹三张牌。

### 3.2 Tapatan（新棋盘游戏）

- 玩法：A/B 轮流在 n×n 点阵上先放子（每人 p 颗），放完后轮流把子移到相邻空点；走出 k 连线即胜。
- **核心难点是邻接规则**：正交邻居永远相邻；对角邻居**仅当** (x+y) 为偶数的点才相邻。这是一个需要精确执行的代数/组合规则。
- 难度梯度：easy=3×3/3子/3连，medium=5×5/4子/4连，hard=7×7/5子/5连（`tapatan_core.py` 的 `TapatanConfig`）。
- 两种输入格式：`move_sequence`（一串走子记录，判定 A win / B win / continue）和 `final_board_state`（终局棋盘字符画，直接找连线）。前者更难——需要先在脑内重建棋盘。
- 默认 demo 数：easy=12，medium=18，hard=24（`get_default_num_demos`）。

### 3.3 Operator Function（新定义算子）

- 玩法：给一个新定义的函数，代入整数求值。
- 难度梯度：
  - d1：f(x,y,z) = x*y + z（2 个算子）
  - d2：g(x,y,z,a) = (x−y)*z + a（3 个算子）
  - d3：h(x,y,z,a,b) = ((x+y)*z − a) mod b（4 个算子 + 取余）
- 输出：`\boxed{结果}`。纯代数计算任务——按论文结论，这是 rules 优势最大的任务类型。
- 注意：`main.py` 里 operator_func 的默认 `num_demos=1`（set_game 是 20），各任务默认示例数差异很大，读结果时要留意这个不对称。

### 3.4 Noun Class Agreement（人工语言语法判断）

- 玩法：给定人工语言句子，判断是否合语法（Yes/No）。
- 难度梯度：
  - d1：2 个名词类（A/B），句型 DET NOUN VERB DET NOUN，限定词 ka/ti 必须与名词类别匹配
  - d2：4 个名词类（A–D），名词短语变为 DET ADJ-SUFF NOUN，形容词后缀（-en/-os/-im/-at）也要匹配类别
  - d3：6 个名词类（A–F），动词变为 PREFIX-ROOT-SUFFIX，前缀匹配主语类、后缀匹配宾语类（如 ge-dax-eb）
- 这是典型的"分布敏感"任务：类别成员是任意约定的（如 tac 是 A 类），必须靠记忆/分布对应，rules 优势按论文结论在这里最小。

### 3.5 Lexical Category Inference（词汇类别推理）

- 玩法：给 4 个类别定义 + 一个 4 词列表，判断列表是否"正确"（第 i 个词属于第 i 类），Yes/No。
- 逻辑条件 × 难度构成设计矩阵：
  - shared/d1：类别定义直接共享
  - either/d2、either/d3：满足任一描述即可（OR）
  - both/d2、both/d3：满足全部描述才行（AND）
- 类别成员用 3 字母无意义词（如 JEN、LOR），其"真实"语义来自 McRae 特征库 / THINGSplus 等外部资源（`bundle_ids` 如 `source_mcrae_feature_a_mammal`），candidate 还有 `candidate_confusability_score` 混淆度分数——负例是精心构造的高混淆项，不是随机词。
- 这是代码量最大的任务：`code/tasks/lexical_category_inference.py` 3320 行，`data_generation/generate_lexical_category_inference.py` 9935 行；默认测 600 个样本（其他任务默认 100）。

## 4. 三种学习模式的 prompt 构造

这是仓库方法论的核心，`code/prompts/*_prompts.py` 逐任务实现：

- **rules**：从 `rules-mode-prompts/*.md` 按标题路径取出模板（见下），填入任务参数（棋盘尺寸、难度等）+ 测试输入。测试输入前有完整的自然语言规则。
- **examples**：极简——N 行 `输入 → 输出`（如 tapatan 的 `A place (0,0) ; B place (1,1) → continue`），末尾一句输出格式指令 + 测试输入。不给任何规则解释，纯粹靠例子归纳。
- **combined**：rules 全文 + `EXAMPLES:` 段（同 examples 的示例行）+ 测试输入。注意是**拼接**，不是交错。

**demo 采样机制**（以 set_game 的 `sample_balanced_demos` 为代表）：从 train split 按**类别均衡**采样（`valid_set_category` 分桶，每类取 k 个再打乱），`random.Random(seed)` 控制可复现。lexical 任务还有专门的 `get_demo_schedule_for_num_demos` 和 prompt 覆盖度校验（`_validate_prompt_demo_coverage`）——保证示例覆盖到所有逻辑条件。

**模板渲染**：`code/prompts/markdown_loader.py` 实现了一个轻量 markdown 模板引擎——按标题路径（如 `("Move Sequence Variant", "Rules", "Easy")`，大小写不敏感、允许顶层标题前缀）在 md 文件里定位 section，取其内容做 Python `str.format` 渲染。`lru_cache` 缓存文件读取。rules-mode-prompts 与可执行代码分离，改 prompt 不用碰代码。

一个值得注意的细节：Tapatan 的 md 里 Easy/Medium/Hard 三个 section 的**规则文本几乎完全相同**，难度差异完全由模板参数（board_size / pieces_per_player / line_length）和数据配置承载——规则描述本身是难度无关的，变量控制干净。

## 5. 代码架构走读

```
.
├── code/
│   ├── main.py                  # 实验入口：argparse → configure_runtime → 异步分发各任务
│   ├── prompts/                 # 逐任务 prompt 构造（rules/examples/combined）+ markdown_loader
│   ├── tasks/                   # 逐任务 runner：数据加载 → prompt 构造 → 批量推理 → 打分 → 落盘
│   └── utils/
│       ├── dataset_io.py        # 数据根目录三级 fallback：RULES_VS_EXAMPLES_DATA_DIR → 仓库内 data/ → 旁边的 rules-vs-examples-dataset/data
│       └── cluster/             # 集群/推理后端：vLLM 单例、MODEL_PATHS（来自 ROR_MODEL_ROOT）、checkpoint/resume、API 并发调用
├── data_generation/             # 5 个数据生成脚本（lexical 的 9935 行是全场最大）
├── rules-mode-prompts/          # 5 个 markdown 规则模板（prompt 的"源代码"）
└── requirements.txt             # vllm / numpy / tqdm / openai / scipy / matplotlib
```

**main.py 的关键设计**：

- 所有运行时配置通过环境变量 `ROR_*` 传递（`ROR_INFERENCE_BACKEND`、`ROR_VLLM_*`、`ROR_API_*`、`ROR_RESUME`、`ROR_CHECKPOINT_EVERY`），argparse 只做一层薄封装——这是为集群批量提交设计的。
- `--mode all` 会跑 rules→examples→combined 三遍；`--task all_final_tasks` 跑全部 5 个任务。
- 默认测试样本数：lexical 600，其余 100；默认 `max_tokens` 很小（tapatan 4、noun_class 8、lexical 2）——输出都是短标签/数字，限制生成长度既省钱又减少格式漂移。
- 支持断点续跑：`--resume` + `checkpoint_every`（默认 50），输出写 `experiment_outputs/`，每条请求记录 prompt、demos、模型输出、打分，可审计。

**推理后端**：vLLM（本地模型，`ROR_MODEL_ROOT` 下按名注册，默认 `qwen3-14b`）或 OpenAI-compatible API（含 `--api_token_param` 适配 GPT-5.x 的 `max_completion_tokens` 参数名变化——这个细节说明作者实际用 GPT-5.4 跑过）。

## 6. 数据集

- 单独存放在 Hugging Face：`tin-lab/rules_vs_examples`，代码仓库里**没有**数据（`data_generation/` 只有生成脚本）。
- 使用方式：`export RULES_VS_EXAMPLES_DATA_DIR=/path/to/rules-vs-examples-dataset/data`，或把数据集 checkout 到代码仓库旁边的 `rules-vs-examples-dataset/` 目录（自动发现）。
- 数据格式：JSON/JSONL，`unwrap_item_record` 把 paper-facing 的 `{id, task, difficulty, split, label, condition, source, item: {...}}` 行记录展开成任务可用的 item。
- **实地观察**：HF 页面上的 dataset viewer 目前是坏的——parquet 生成时 schema cast 失败（某列的 struct 类型与声明的 features 不匹配，`DatasetGenerationError`）。数据文件本身可下载不受影响，但在线预览看不了。这是上游仓库的一个小瑕疵，复现时直接 `huggingface-cli download` 拿文件即可。

## 7. 复现指南（基于代码整理，未实际跑过）

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 数据（二选一）
export RULES_VS_EXAMPLES_DATA_DIR=/path/to/rules-vs-examples-dataset/data
# 或：把数据集 checkout 到本仓库旁边的 rules-vs-examples-dataset/ 目录

# vLLM 本地模型
export ROR_MODEL_ROOT=/path/to/downloaded/models
python code/main.py --inference_backend vllm --model qwen3-14b \
  --task tapatan --mode combined --tapatan_input_format final_board_state

# OpenAI API（以 GPT-5.4 为例）
export OPENAI_API_KEY=...
python code/main.py --inference_backend api --model gpt-5.4 \
  --api_token_param max_completion_tokens --max_tokens 128 \
  --task tapatan --mode combined --tapatan_input_format final_board_state
```

注意点：

- 复现的**主要成本**是推理量：5 任务 × 3 mode × 多难度 × N 模型，论文量级需要集群或可观的 API 预算；单任务 smoke test（如 tapatan easy + rules）成本很低。
- `data_generation/` 脚本可重新生成数据，但 lexical 的生成脚本近万行，依赖外部特征资源，重新生成不 trivial。
- 先跑 `--list_models` 确认 `ROR_MODEL_ROOT` 下的模型注册名与 `--model` 参数一致。

## 8. 方法论观察与批判性思考

**做得好的地方**：

1. **新任务防泄漏**：5 个任务全是程序生成的全新任务，rules 和 examples 描述同一任务——对照干净，不像很多 ICL 论文用的是模型可能见过的公开 benchmark。
2. **变量控制细**：难度只改任务参数不改规则文本；demo 按类别均衡采样且 seed 可复现；输出长度限制、打分规则逐任务明确。
3. **工程完备**：checkpoint/resume、请求级日志（prompt+demos+输出全记录）、双后端（vLLM/API），别人复现时不容易踩坑。
4. **任务选择有理论意图**：代数计算（operator）vs 分布记忆（noun class）vs 组合推理（tapatan/set game）的对比，直接对应论文"任务属性调节 rules 优势"的结论——任务不是随便选的。

**可商榷 / 值得深想的地方**：

1. **examples 的默认数量不对称**：set_game 默认 20 个 demo，operator_func 默认 1 个，tapatan 按难度 12/18/24。论文结论之一是"增加示例数无一致增益"，但各任务的 demo 数默认值差异大，跨任务比较 rules-vs-examples gap 时，这个超参数的影响值得单独审视。
2. **combined 只是拼接**：rules 全文 + examples 列表的简单拼接，没有探索更深度的融合方式（如用规则去标注示例、或示例驱动规则细化）。"rules+examples 无增益"的结论严格来说是"这种拼接方式无增益"。
3. **rules 的质量是作者写的**：rules 模板是研究者精心撰写的清晰规则——现实中的 instruction 往往没这么干净。结论的外推边界是"高质量规则 vs 裸示例"。
4. **评估是 exact match**：短标签/数字任务上合理，但会低估"推理对、格式错"的情况；作者用很小的 max_tokens 和严格的输出指令来缓解。
5. **base vs instruct 的比较**："base 模型也没有 examples 特权"这个发现很反直觉（通常认为 base 模型更依赖 few-shot 格式对齐），值得精读论文的实验细节——可能是因为任务太新，base 模型的 few-shot 归纳能力本身就弱。

## 9. 待探索

- [ ] 精读论文全文（重点看实验细节：模型列表、demo 数量消融、统计显著性处理）
- [ ] 单任务 smoke 复现（如 tapatan easy + rules + 小模型/API），验证链路跑通
- [ ] 检查上游 issue 区是否有已知问题（如 HF viewer 的 schema 错误是否有人提）
- [ ] 与 humanize/kda 的关系：无直接关系；这是纯研究复现仓库，不考虑 PR（用户已确认）

---

*本文档为个人学习笔记，基于 2026-10-07 对 `e276b4a`（fork 时上游 HEAD）的代码走读 + arXiv 摘要/结论页。引用论文观点处均为转述。*
