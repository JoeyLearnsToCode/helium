# Helium AI Agent Guidelines

## 运作模型（先读这段）

这是 Helium 的个人自用 fork：**只做 Windows 版**，产物只通过 GitHub Releases 分发，不向上游提 PR，也不发到任何其他渠道。改动只服务于自己的构建。沟通、文档、提交说明一律用中文。

编译、打包、签名、发布一律在 GitHub Actions 完成，本仓库没有构建目标。

本地校验以"依赖最轻"为门槛，按改动类型分：

| 改动 | 本地做 | 停手条件 |
| --- | --- | --- |
| `utils/`、`devutils/` 的 Python | `.venv/Scripts/python -m py_compile <file>`，必要时跑该目录测试 | 需要重依赖才能本地验证时直接跳过，并说明"未本地验证" |
| `patches/` | 只做结构检查（且在 Windows 上不可信，见下） | 能否编译通过只能由 Actions 判定 |
| `i18n/`、`resources/` | JSON 可解析、被引用的文件存在 | 同上 |

编译失败只从 Actions 日志定位，不在本地为验证去准备 Chromium 依赖。

## Windows 环境事实

- 工具链在 `.venv`（uv + CPython 3.10.9，依赖见 `.requirements.txt`）。PATH 上的 `python3` 是未配置版本的 mise shim，直接报错；统一用 `.venv/Scripts/python`。
- 未安装 `quilt`、`zstd`（`patch`、`tar`、`xz`、`git` 可用）。
- 在 Windows 上可信的只有两项：`python -m yapf --style .style.yapf -e '*/third_party/*' -rpd utils`（或 `devutils`）和 `devutils` 的测试（`.venv/Scripts/python -m pytest -c devutils/pytest.ini`，需在 `devutils/` 目录内执行）。
- 以下会给出假结果，不要当作结论汇报：`devutils/validate_config.py`（`devutils/check_patch_files.py:89` 拿反斜杠路径比正斜杠的 `series`，346 条 patch 被判"未使用"）、`devutils/lint.py`（同因）、`run_*_pylint.py`（`devutils/run_other_pylint.py:35` 调 `os.getuid()`）、`utils` 测试（`utils/tests/test_patches.py:24` 依赖 `/bin/false`，`test_domain_substitution.py` 依赖 LF）。这些都是 POSIX 假设，CI 在 Ubuntu 上跑，不是仓库 bug。

## 仓库是什么

共享层：quilt patch 序列、依赖清单、品牌与 i18n 资源、GN flags，以及把准备好的 Chromium 源码变成 Helium 树的 Python 工具。编译、签名、发布在平台仓（`helium-macos`、`helium-linux`、`helium-windows`），它们把本仓库作为 `helium-chromium` 子模块引用——本仓库没有 `.gitmodules`，也没有需要初始化或 `submodule update` 的外部仓库。

| 路径 | 内容 |
| --- | --- |
| `patches/series` | 全部 patch 的顺序表，自上而下应用 |
| `patches/<vendor>/` | 继承来的 patch，按来源分组（`ungoogled-chromium`、`inox-patchset`、`iridium-browser`、`bromite`、`brave`、`debian`、`upstream-fixes`） |
| `patches/helium/{core,settings,ui}` | 自己的 patch，占大头（`core/network`、`core/noise` 另分） |
| `flags.gn` | 追加到平台构建的 GN 参数（关 PGO、关上报与 safebrowsing、Google API key 置空） |
| `downloads.ini` | Chromium 源码 tarball，用同名 `.hashes` 校验 |
| `deps.ini` | Chromium 就位后再拉的东西：非自由搜索引擎数据、`helium-onboarding`、uBlock Origin（解到 `third_party/ublock`） |
| `utils/` | 流水线各步：`downloads.py`、`clone.py`、`patches.py`、`prune_binaries.py`、`domain_substitution.py`、`name_substitution.py`、`i18n_apply.py`、`generate_resources.py`、`helium_version.py` |
| `devutils/` | 开发侧校验与 i18n 工具 |
| `i18n/` | `source.gen.json`（从 patch 生成）、各语言 `translations/*.json`、`owners.yml` |
| `domain_regex.list` → `domain_substitution.list` | 域名替换的正则与生成后的逐文件清单 |
| `pruning.list` | 生成后的待删二进制清单 |

`patches/series` 约 370 条，其中 256 条属于 `helium/`。顺序是有语义的（quilt 逐条应用，后面的 patch 带着针对前面结果的刷新过的 hunk），`series` 与实际文件必须双向一致。

## 版本与 Release

`utils/helium_version.py` 读四个文件拼出浏览器版本 `<major>.<minor>.<patch>`：

- `version.txt` → major（现为 `0`）
- `chromium_version.txt` → minor，取 Chromium 大版本减 `CHROME_VERSION_BASE = 136`（`156.0.8078.79` → `20`）
- `revision.txt` → patch（本仓库的 bump 计数）
- 平台仓的 `revision.txt` → 第四段，仅在传 `--platform-tree` 时出现

`release-and-tag.yml` 在 push 到 `main` 且该版本号变化时建 tag，tag 长相是 `devutils/print_tag_version.sh` 的输出 `<chromium_version>-<revision>`（如 `156.0.8078.79-3`），与浏览器版本号不是一回事。Windows 二进制由平台仓的 Actions 产出，不在本仓库。

## 改 patch 的规矩

- 新 patch 放 `patches/helium/{core,settings,ui}`，并加进 `patches/series`；vendor 分组保持原样。
- 新增到 Chromium 树里的文件要带 Helium 版权头，新头文件要带对应的 include guard（`devutils/_lint_tests.py` 检查）。
- 有 Chromium 树时用 quilt 维护 hunk（`source devutils/set_quilt_vars.sh`，然后在 `build/src` 里 `quilt push`/`refresh`/`pop`），手改 `patches/` 里的 hunk 会让 offset 失准。**本地没有 quilt**，所以 patch 改动的唯一验收是 Actions 构建。

## 开发后验证 Actions

按"先看本仓库 CI，再看平台仓构建/Release"的顺序，只关心 Windows 结果：

```bash
gh run list --repo <owner>/helium --limit 10                 # 本仓库 ci.yml / lint.yml
gh run view <run-id> --repo <owner>/helium --log-failed      # 失败日志
gh run watch <run-id> --repo <owner>/helium-windows          # Windows 构建进度
gh release view <tag> --repo <owner>/helium-windows          # 产物是否挂上
```

平台仓的 Linux/macOS 任务失败与本 fork 无关，忽略。
