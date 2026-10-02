# 1Cat-vLLM 本地 fork 工作流（2026-10-02 建立）

## 仓库布局
- remote `origin` = https://github.com/1CatAI/1Cat-vLLM（blobless clone，211M）
- 分支 `main`：只跟踪上游，禁止直接提交。
- 分支 `archive/fp8mtp1`：已放弃的 E4M3 KV+MTP 实验（用户 2026-10-02 确认放弃，仅作资产存档，基于 `d304698`）
- 分支 `local-docs`：本工作流文档（不混入 main）
- 分支 `local-dev`：**当前活跃**的本地改动（现在为空=与上游一致），未来的改动在此分支开发（2026-09-29 上游 main，= 生产 wheel `1.5.0+flash.d304698` 的基线，已证明与 labs/flash-next-d304698/build-src 逐字节一致）。
- 放弃线 `archive/fp8mtp1` = commit `5448ed5`（E4M3 KV + MTP 实验，5 文件 +137/-35；2026-10-02 用户确认放弃，不进 local-dev）。
  补丁副本：`~/projects/V100_LLM/labs/flash-fp8-mtp-dev-20261002/local-dev.patch`

## 升级上游的标准流程
1. `git fetch origin main`，看 `git log --oneline <旧基线>..origin/main`。
2. **冲突门禁**：`git diff --name-only <旧基线>..origin/main` 与我们的改动文件表（`git diff --name-only $(git merge-base origin/main local-dev)..local-dev`）求交集；有交集 → 先试 `git rebase origin/main` 于 local-dev 分支解决冲突（在 worktree 里做：`git worktree add ../1cat-vllm-wt local-dev`）。
3. **重编门禁**：交集为空 **且** 上游 diff 不含 `csrc/ | CMakeLists.txt | setup.py | pyproject.toml | requirements/` → 才允许快速通道（zip 替换 py、保留原生 .so，即 build_patch_wheel.py 模式，其基线字节校验必须保留）；否则一律全量重建 wheel（CUDA 12.8 / py3.12 / 含 submodules：`git submodule update --init`）。
   - 例：d304698→e0f3fed（+46 commits）命中两条门禁（envs.py 被上游改、setup.py 被改）→ 必须 rebase + 全量重建。
4. 新 wheel 装**隔离 env**（labs/ 下新目录），跑既有验收：quality-regression、thinking-off 工具调用探针（name=add 整数参数）、speed matrix 对照（基线 d4/b2048/BF16 ≈107 tps）。
5. 验收通过 → 经 `~/projects/V100_LLM/ops/flash_production/deploy.py` 晋升；vLLM 停/启只按 pid 文件精确 PID，且**必须用户确认**（AGENTS.md §4）。

## 版本命名
`1.5.0+flash.<上游短SHA>[.<本地特征>]`，例 `1.5.0+flash.d304698.fp8mtp1`。wheel sha256 + 上游 SHA + 本地改动清单三件套记入 nmem。

## 禁止
- 在 main 上提交；直接改 labs/flash-next-d304698/build-src（只读归档基线，勿动）。
- 用 pkill 停 vLLM；只按 pid 文件。
