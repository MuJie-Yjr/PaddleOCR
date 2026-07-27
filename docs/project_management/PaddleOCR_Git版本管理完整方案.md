# PaddleOCR Git 版本管理完整方案

> 适用环境：Windows + PowerShell + VS Code  
> GitHub 用户：`MuJie-Yjr`  
> 统一根目录：`D:\model\V-model\PPOCR`  
> 唯一源码目录：`D:\model\V-model\PPOCR\PaddleOCR`  
> 管理原则：所有版本通过 `git switch` 切换，不使用 `git worktree add`

## 1. 核心规则

```text
main 不修改
官方 tag 只测试和作为分支基线
custom 分支才长期开发
feature 分支只承载单个较大功能
upgrade 分支只用于版本迁移
一个独立功能对应一个或少量清晰 commit
```

PaddleOCR 官方日常开发位于 `main`，正式版本使用 `vX.Y.Z` tag。你的稳定修改应从官方 tag 建立，而不是直接在 `main` 上修改。

## 2. 目录结构

```text
D:\model\V-model\PPOCR\
├── PaddleOCR\                         # 唯一 PaddleOCR 源码仓库
├── PaddleX\                           # 唯一 PaddleX 源码仓库
├── official-models\
│   ├── PaddleOCR\
│   └── PaddleX\
├── models\
│   ├── PaddleOCR\
│   │   ├── trained\
│   │   ├── exported\
│   │   └── deployment\
│   └── PaddleX\
├── datasets\
│   ├── PaddleOCR\
│   └── PaddleX\
├── runs\
│   ├── PaddleOCR\
│   └── PaddleX\
├── environments\
└── profiles\
```

不要把数据集、模型权重、训练输出、缓存、ONNX、RKNN、TensorRT engine 或部署包提交进源码仓库。

## 3. GitHub 仓库关系

```text
官方仓库：https://github.com/PaddlePaddle/PaddleOCR
自己的 Fork：https://github.com/MuJie-Yjr/PaddleOCR

origin    -> 自己的 Fork
upstream  -> 官方 PaddleOCR
```

## 4. 第一次创建目录

```powershell
$ROOT = "D:\model\V-model\PPOCR"

$directories = @(
    "$ROOT",
    "$ROOT\official-models\PaddleOCR",
    "$ROOT\official-models\PaddleX",
    "$ROOT\models\PaddleOCR\trained",
    "$ROOT\models\PaddleOCR\exported",
    "$ROOT\models\PaddleOCR\deployment",
    "$ROOT\models\PaddleX\trained",
    "$ROOT\models\PaddleX\exported",
    "$ROOT\models\PaddleX\deployment",
    "$ROOT\datasets\PaddleOCR",
    "$ROOT\datasets\PaddleX",
    "$ROOT\runs\PaddleOCR",
    "$ROOT\runs\PaddleX",
    "$ROOT\environments",
    "$ROOT\profiles"
)

foreach ($directory in $directories) {
    New-Item -ItemType Directory -Force -Path $directory | Out-Null
}
```

## 5. Fork 和克隆

在 GitHub 打开官方 PaddleOCR 仓库，点击 `Fork`：

```text
Owner：MuJie-Yjr
Repository name：PaddleOCR
Copy the main branch only：勾选
```

克隆自己的 Fork：

```powershell
$ROOT = "D:\model\V-model\PPOCR"

git clone `
  https://github.com/MuJie-Yjr/PaddleOCR.git `
  "$ROOT\PaddleOCR"

Set-Location "$ROOT\PaddleOCR"
```

检查：

```powershell
git status
git remote -v
```

## 6. 添加官方 upstream

```powershell
git remote add upstream https://github.com/PaddlePaddle/PaddleOCR.git
git fetch upstream --prune --tags
git remote -v
```

只跟踪官方 `main`，避免 VS Code 出现大量官方分支：

```powershell
git remote set-branches upstream main
```

清理已经获取的多余 `upstream/*` 引用：

```powershell
$refs = git for-each-ref --format="%(refname)" refs/remotes/upstream

$refs |
    Where-Object {
        $_ -ne "refs/remotes/upstream/main" -and
        $_ -ne "refs/remotes/upstream/HEAD"
    } |
    ForEach-Object {
        git update-ref -d $_
    }

git fetch upstream --prune --tags
git branch -r
```

正常主要显示：

```text
origin/HEAD -> origin/main
origin/main
upstream/main
```

## 7. Git 配置

```powershell
git config rerere.enabled true
git config merge.conflictStyle zdiff3
git config fetch.prune true
```

提交身份：

```powershell
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
```

## 8. 分支设计

```text
main
└── 只同步 upstream/main

vX.Y.Z
└── 官方稳定 tag，只测试和作为基线

custom/vX.Y.Z
└── 基于指定官方 tag 的长期开发分支

feature/vX.Y.Z/功能名
└── 单个较大功能

upgrade/v旧版本-to-v新版本
└── 版本迁移和冲突处理

custom-vX.Y.Z-rN
└── 自己验证通过的稳定 tag
```

## 9. 查看官方版本

```powershell
git fetch upstream --prune --tags
git tag --sort=-version:refname | Select-Object -First 20
```

验证 tag：

```powershell
git rev-parse --verify refs/tags/v3.7.0
```

检查指定 PaddleOCR 版本要求的 PaddleX 范围：

```powershell
git show v3.7.0:pyproject.toml |
    Select-String -Pattern "paddlex"
```

这一步不能省略。PaddleOCR 和 PaddleX 不能任意组合。

## 10. 创建自己的开发分支

以 `v3.7.0` 为例：

```powershell
Set-Location "D:\model\V-model\PPOCR\PaddleOCR"

git status
git fetch upstream --prune --tags
git switch -c custom/v3.7.0 v3.7.0
git push -u origin custom/v3.7.0
```

检查：

```powershell
git branch --show-current
git status
```

创建其他版本同理：

```powershell
git switch -c custom/v3.5.0 v3.5.0
git push -u origin custom/v3.5.0
```

已有分支直接切换：

```powershell
git switch custom/v3.7.0
git switch custom/v3.5.0
git switch main
```

禁止：

```powershell
git worktree add ...
```

本机只保留一份 PaddleOCR 源码目录。

## 11. VS Code 切换版本

VS Code 永远打开：

```text
D:\model\V-model\PPOCR\PaddleOCR
```

```powershell
code "D:\model\V-model\PPOCR\PaddleOCR"
```

点击左下角分支名称切换：

```text
main
custom/v3.7.0
custom/v3.5.0
feature/...
upgrade/...
```

测试官方 tag：

```powershell
git switch --detach v3.7.0
git status
git describe --tags --exact-match
```

测试完成后：

```powershell
git switch custom/v3.7.0
```

```text
分支图标 custom/v3.7.0    可以修改和提交
标签图标 v3.7.0           官方原始版本，只测试
```

## 12. 本地源码安装

建议按一组 PaddleOCR/PaddleX 版本创建独立环境：

```powershell
conda create -n ppocr-37 python=3.10 -y
conda activate ppocr-37
```

先安装适合当前硬件与 CUDA 的 PaddlePaddle。

先安装配套 PaddleX 源码：

```powershell
python -m pip install -e `
  "D:\model\V-model\PPOCR\PaddleX[ocr]"
```

进入 PaddleOCR：

```powershell
Set-Location "D:\model\V-model\PPOCR\PaddleOCR"
```

安装训练和导出依赖：

```powershell
python -m pip install -r requirements.txt
```

可编辑安装：

```powershell
python -m pip install -e .
```

需要全部可选能力时：

```powershell
python -m pip install -e ".[all]"
```

检查：

```powershell
python -m pip check

python -c "import importlib.metadata as m, paddleocr; print(m.version('paddleocr')); print(paddleocr.__file__)"
```

路径必须指向：

```text
D:\model\V-model\PPOCR\PaddleOCR\paddleocr\...
```

## 13. 日常开发流程

```powershell
Set-Location "D:\model\V-model\PPOCR\PaddleOCR"

git switch custom/v3.7.0
git pull --ff-only
git branch --show-current
git status

# 修改源码

git diff
git add 具体文件
git commit -m "feat: add custom recognition head"
git push
```

推荐提交：

```text
feat: add four-direction angle classifier
feat: add custom recognition head
feat: support custom OCR annotation format
fix: correct recognition postprocess
fix: resolve model export error
refactor: separate custom augmentation pipeline
```

## 14. 大功能使用 feature 分支

```powershell
git switch custom/v3.7.0
git pull --ff-only
git switch -c feature/v3.7.0/four-angle-cls
```

开发完成：

```powershell
git add 具体文件
git commit -m "feat: add four-direction angle classifier"
git push -u origin feature/v3.7.0/four-angle-cls
```

整理并合并：

```powershell
git switch feature/v3.7.0/four-angle-cls
git rebase custom/v3.7.0

git switch custom/v3.7.0
git merge --ff-only feature/v3.7.0/four-angle-cls
git push
```

清理：

```powershell
git branch -d feature/v3.7.0/four-angle-cls
git push origin --delete feature/v3.7.0/four-angle-cls
```

## 15. 未完成代码需要切换版本

```powershell
git stash push -m "unfinished PaddleOCR modification"

git switch main

git switch custom/v3.7.0
git stash pop
```

`stash` 只用于短期临时保存，重要代码必须正常提交。

## 16. 同步官方 main

```powershell
git fetch upstream --prune --tags

git switch main
git merge --ff-only upstream/main
git push origin main

git switch custom/v3.7.0
```

禁止把持续变化的 `upstream/main` 直接合并进固定 `custom/vX.Y.Z` 分支。

## 17. 全部修改迁移到新版本

示例：

```text
旧官方版本：v3.6.0
旧开发分支：custom/v3.6.0
新官方版本：v3.7.0
```

先确认目标 PaddleOCR 对 PaddleX 的要求，再迁移 PaddleOCR：

```powershell
git fetch upstream --prune --tags

git switch custom/v3.6.0
git pull --ff-only
git status

git switch -c upgrade/v3.6.0-to-v3.7.0
git rebase --onto v3.7.0 v3.6.0
```

冲突：

```powershell
git status
git add 冲突文件
git rebase --continue
```

放弃：

```powershell
git rebase --abort
```

测试通过：

```powershell
git branch -m custom/v3.7.0
git push -u origin custom/v3.7.0
```

如果目标分支已存在，不要覆盖，应使用新的升级分支测试后再决定合并方式。

## 18. 只迁移部分修改

```powershell
git log --reverse --oneline v3.6.0..custom/v3.6.0

git switch -c custom/v3.7.0 v3.7.0

git cherry-pick 提交哈希1 提交哈希2
```

冲突：

```powershell
git add 冲突文件
git cherry-pick --continue
```

放弃：

```powershell
git cherry-pick --abort
```

## 19. 升级后的验证

```powershell
python -m compileall paddleocr ppocr tools
python -m pip install -r requirements.txt
python -m pip install -e .
python -m pip check
```

至少验证：

```text
配置读取
数据集加载
模型构建
单次前向
loss 反向传播
单批次训练
1 个 epoch
验证指标
模型导出
推理接口
自定义后处理
与 PaddleX 产线联调
```

## 20. 数据、模型和输出路径

数据集：

```text
D:\model\V-model\PPOCR\datasets\PaddleOCR
```

训练输出：

```text
D:\model\V-model\PPOCR\runs\PaddleOCR
```

训练命令示例：

```powershell
python tools/train.py `
  -c configs/rec/PP-OCRv5/PP-OCRv5_server_rec.yml `
  -o Global.save_model_dir="D:/model/V-model/PPOCR/runs/PaddleOCR/rec/experiment-01"
```

确认保留的模型：

```text
D:\model\V-model\PPOCR\models\PaddleOCR\trained
```

导出模型：

```text
D:\model\V-model\PPOCR\models\PaddleOCR\exported
```

## 21. 官方模型缓存

PaddleOCR 3.x 的很多官方产线模型由 PaddleX 下载和管理。

统一缓存根目录：

```powershell
setx PADDLE_PDX_CACHE_HOME `
  "D:\model\V-model\PPOCR\official-models\PaddleX"
```

重新打开终端后：

```powershell
echo $env:PADDLE_PDX_CACHE_HOME
```

实际模型通常进入该目录下的 `official_models`。

## 22. 稳定 tag

```powershell
git switch custom/v3.7.0
git pull --ff-only
git status

git tag -a custom-v3.7.0-r1 `
  -m "PaddleOCR v3.7.0 custom stable release r1"

git push origin custom-v3.7.0-r1
```

## 23. 恢复错误修改

放弃文件修改：

```powershell
git restore 文件路径
```

取消暂存：

```powershell
git restore --staged 文件路径
```

撤销已经推送的提交：

```powershell
git log --oneline
git revert 提交哈希
git push
```

不要随意使用：

```powershell
git reset --hard
git push --force
```

## 24. 常用状态命令

```powershell
git branch --show-current
git status
git branch
git branch -a
git branch -r
git tag --sort=-version:refname
git rev-parse HEAD
git describe --tags --exact-match
git log --oneline --graph --decorate --all -30
```

查看当前分支要求的 PaddleX：

```powershell
Select-String `
  -Path "D:\model\V-model\PPOCR\PaddleOCR\pyproject.toml" `
  -Pattern "paddlex"
```

## 25. 必须避免

```text
不要保留 PaddleOCR-old、PaddleOCR-new、PaddleOCR-final。
不要使用 git worktree add。
不要在 main 修改源码。
不要在官方 tag 的 detached HEAD 中长期开发。
不要把 upstream/main 直接合并到固定 custom 分支。
不要整目录覆盖新版源码。
不要把多个无关功能塞进一个 commit。
不要忽略 PaddleX 依赖范围。
不要把模型、数据集、runs、缓存和部署包提交到源码仓库。
```

## 26. 最终规则

```text
唯一源码目录：
D:\model\V-model\PPOCR\PaddleOCR

Git 中保存所有版本：
main
custom/v3.5.0
custom/v3.6.0
custom/v3.7.0
feature/...
upgrade/...
官方 tags

VS Code 只打开唯一源码目录，通过左下角切换分支。
```

## 27. PaddleOCR 与 PaddleX 配套示例

```text
PaddleOCR v3.5.0 -> PaddleX >=3.5.0,<3.6.0
PaddleOCR v3.6.0 -> PaddleX >=3.6.0,<3.7.0
PaddleOCR v3.7.0 -> PaddleX >=3.7.0,<3.8.0
```

例如：

```text
PaddleOCR custom/v3.5.0 + PaddleX custom/v3.5.2
PaddleOCR custom/v3.7.0 + PaddleX custom/v3.7.2
```

每次都应以目标 PaddleOCR tag 的 `pyproject.toml` 为最终依据。

## 28. 许可证

PaddleOCR 使用 Apache-2.0。Fork、修改和商业使用时仍需保留许可证、版权与通知要求，并检查依赖项目的许可证。
