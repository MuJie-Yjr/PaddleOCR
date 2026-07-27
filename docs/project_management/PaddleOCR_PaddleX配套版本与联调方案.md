# PaddleOCR 与 PaddleX 配套版本及联调方案

> 统一根目录：`D:\model\V-model\PPOCR`  
> PaddleOCR 唯一源码目录：`D:\model\V-model\PPOCR\PaddleOCR`  
> PaddleX 唯一源码目录：`D:\model\V-model\PPOCR\PaddleX`

## 1. 配套原则

PaddleOCR 3.x 依赖 PaddleX。不能简单把两个项目各自的“最新版本”随意组合。

正确顺序：

```text
先选择 PaddleOCR 目标 tag
读取该 tag 的 pyproject.toml
确认 paddlex 依赖范围
再选择范围内的 PaddleX tag 或 custom 分支
```

检查命令：

```powershell
git -C "D:\model\V-model\PPOCR\PaddleOCR" `
  show v3.7.0:pyproject.toml |
  Select-String -Pattern "paddlex"
```

## 2. 常见配套示例

| PaddleOCR 基线 | PaddleX 要求 | PaddleX 示例 |
|---|---|---|
| `v3.5.0` | `>=3.5.0,<3.6.0` | `v3.5.0`、`v3.5.1`、`v3.5.2` |
| `v3.6.0` | `>=3.6.0,<3.7.0` | `v3.6.0`、`v3.6.1` |
| `v3.7.0` | `>=3.7.0,<3.8.0` | `v3.7.0`、`v3.7.1`、`v3.7.2` |

合理示例：

```text
PaddleOCR custom/v3.5.0 + PaddleX custom/v3.5.2
PaddleOCR custom/v3.7.0 + PaddleX custom/v3.7.2
```

错误示例：

```text
PaddleOCR v3.5.0 + PaddleX v3.7.2
```

## 3. 分支配对

```text
OCR 3.5 组合：
PaddleOCR custom/v3.5.0
PaddleX   custom/v3.5.2

OCR 3.6 组合：
PaddleOCR custom/v3.6.0
PaddleX   custom/v3.6.1

OCR 3.7 组合：
PaddleOCR custom/v3.7.0
PaddleX   custom/v3.7.2
```

## 4. 切换一组版本

先停止训练、推理和 VS Code 调试进程。

```powershell
git -C "D:\model\V-model\PPOCR\PaddleX" `
  switch custom/v3.7.2

git -C "D:\model\V-model\PPOCR\PaddleOCR" `
  switch custom/v3.7.0
```

检查：

```powershell
git -C "D:\model\V-model\PPOCR\PaddleX" status
git -C "D:\model\V-model\PPOCR\PaddleOCR" status
```

## 5. 环境隔离

建议每一代组合一个 Conda 环境：

```text
ppocr-35
ppocr-36
ppocr-37
```

例如：

```powershell
conda create -n ppocr-37 python=3.10 -y
conda activate ppocr-37
```

源码仍然只有 PaddleOCR 和 PaddleX 各一份。Conda 环境隔离是为了解决 PaddlePaddle、CUDA、推理引擎和可选依赖差异。

## 6. 正确安装顺序

先安装适合硬件的 PaddlePaddle。

安装 PaddleX editable：

```powershell
python -m pip install -e `
  "D:\model\V-model\PPOCR\PaddleX[ocr]"
```

安装 PaddleOCR 训练依赖：

```powershell
python -m pip install -r `
  "D:\model\V-model\PPOCR\PaddleOCR\requirements.txt"
```

安装 PaddleOCR editable：

```powershell
python -m pip install -e `
  "D:\model\V-model\PPOCR\PaddleOCR"
```

检查：

```powershell
python -m pip check

python -c "import importlib.metadata as m, paddlex, paddleocr; print('PaddleX:', m.version('paddlex'), paddlex.__file__); print('PaddleOCR:', m.version('paddleocr'), paddleocr.__file__)"
```

两个路径必须分别指向本地 `PaddleX` 和 `PaddleOCR` 源码目录。

## 7. 版本 Profile

创建：

```text
D:\model\V-model\PPOCR\profiles\ocr-stack-3.7.yaml
```

示例：

```yaml
profile: ocr-stack-3.7

paddleocr:
  upstream_tag: v3.7.0
  branch: custom/v3.7.0
  commit: 填写完整commit哈希

paddlex:
  upstream_tag: v3.7.2
  branch: custom/v3.7.2
  commit: 填写完整commit哈希

environment:
  conda: ppocr-37
  python: "3.10"
  paddlepaddle: 记录实际版本
  cuda: 记录实际版本

features:
  - four-angle-classifier
  - custom-recognition-head
  - custom-pipeline
```

获取 commit：

```powershell
git -C "D:\model\V-model\PPOCR\PaddleOCR" rev-parse HEAD
git -C "D:\model\V-model\PPOCR\PaddleX" rev-parse HEAD
```

保存环境：

```powershell
python -m pip freeze `
  > "D:\model\V-model\PPOCR\environments\ppocr-37-requirements.txt"
```

## 8. 两个仓库同时修改

跨仓库功能要在两边分别创建清晰提交。

示例：

```text
PaddleOCR:
feat: add custom recognition architecture

PaddleX:
feat: register custom recognition architecture
```

在 Profile 中记录对应关系：

```yaml
feature: custom-recognition-architecture
paddleocr_commit: abcdef...
paddlex_commit: 123456...
```

Git 无法跨两个仓库创建原子 commit，必须用 Profile 管理对应关系。

## 9. 升级顺序

从 3.6 组合升级到 3.7 组合：

```text
旧组合：
PaddleOCR custom/v3.6.0
PaddleX   custom/v3.6.1

新组合：
PaddleOCR custom/v3.7.0
PaddleX   custom/v3.7.2
```

正确顺序：

```text
1. 获取两个仓库的新 tags。
2. 检查目标 PaddleOCR 的 paddlex 依赖范围。
3. 先迁移 PaddleX 修改。
4. 验证 PaddleX。
5. 再迁移 PaddleOCR 修改。
6. 先安装 PaddleX editable。
7. 再安装 PaddleOCR editable。
8. 执行 python -m pip check。
9. 完整联调回归。
10. 创建新的 Profile。
```

## 10. 升级 PaddleX

```powershell
git -C "D:\model\V-model\PPOCR\PaddleX" fetch upstream --prune --tags

git -C "D:\model\V-model\PPOCR\PaddleX" switch custom/v3.6.1

git -C "D:\model\V-model\PPOCR\PaddleX" `
  switch -c upgrade/v3.6.1-to-v3.7.2

git -C "D:\model\V-model\PPOCR\PaddleX" `
  rebase --onto v3.7.2 v3.6.1
```

## 11. 升级 PaddleOCR

```powershell
git -C "D:\model\V-model\PPOCR\PaddleOCR" fetch upstream --prune --tags

git -C "D:\model\V-model\PPOCR\PaddleOCR" switch custom/v3.6.0

git -C "D:\model\V-model\PPOCR\PaddleOCR" `
  switch -c upgrade/v3.6.0-to-v3.7.0

git -C "D:\model\V-model\PPOCR\PaddleOCR" `
  rebase --onto v3.7.0 v3.6.0
```

## 12. 联调回归

```powershell
conda activate ppocr-37

python -m pip install -e `
  "D:\model\V-model\PPOCR\PaddleX[ocr]"

python -m pip install -r `
  "D:\model\V-model\PPOCR\PaddleOCR\requirements.txt"

python -m pip install -e `
  "D:\model\V-model\PPOCR\PaddleOCR"

python -m pip check
```

至少测试：

```text
PaddleX import
PaddleOCR import
版本与源码路径
官方模型缓存
文本检测
文本识别
方向分类
完整 OCR pipeline
自定义训练
模型导出
本地模型推理
服务化部署
全部自定义功能回归
```

## 13. 快速检查命令

```powershell
Write-Host "=== Branches ==="

git -C "D:\model\V-model\PPOCR\PaddleOCR" branch --show-current
git -C "D:\model\V-model\PPOCR\PaddleX" branch --show-current

Write-Host "=== PaddleX requirement ==="

Select-String `
  -Path "D:\model\V-model\PPOCR\PaddleOCR\pyproject.toml" `
  -Pattern "paddlex"

Write-Host "=== Installed versions and paths ==="

python -c "import importlib.metadata as m, paddlex, paddleocr; print('PaddleX:', m.version('paddlex'), paddlex.__file__); print('PaddleOCR:', m.version('paddleocr'), paddleocr.__file__)"

python -m pip check
```

## 14. 必须避免

```text
不要任意组合两个项目各自的最新版本。
不要只看 PaddleX tag，不检查 PaddleOCR pyproject.toml。
不要只切换一个仓库后直接运行。
不要使用 git worktree add。
不要复制 PaddleOCR-3.7、PaddleX-3.7 等多份源码目录。
不要用 --no-deps 掩盖依赖冲突。
不要忽略 python -m pip check。
不要在跨仓库功能中只记录一边 commit。
```

## 15. 最终规则

```text
源码目录只有两份：
PaddleOCR 一份
PaddleX 一份

版本由 Git 管理：
每个仓库通过 git switch 切换

运行由环境管理：
每一代组合建议一个 Conda 环境

组合由 Profile 管理：
记录两个仓库的 tag、branch、commit 和环境

升级顺序：
先 PaddleX，后 PaddleOCR，最后联调
```
