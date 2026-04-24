## VSCode + Git 上传文件记录 — 我的第二次Git折腾

> 2026-04-24，在第一次折腾的基础上，用 VSCode 和 Git 完成了第一个正式研究项目的代码提交。

---

### 起因

在和 Claude 合作（说起来倒是合作其实谁干的活更多都清楚）完成《毛主席语录》出处时期分布分析（见我的marxism-works库）之后，需要把文章、代码、图表和原始数据一起提交到 GitHub 公开仓库。这次不用 Obsidian Git，直接用 VSCode + Git 命令行。

---

### 第一关：安装依赖库

代码用到了 `matplotlib` 和 `pdfplumber`，第一次运行报错：

```
ModuleNotFoundError: No module named 'matplotlib'
```

在 VSCode 终端输入：

bash

```bash
pip install matplotlib pdfplumber
```

等待安装完成，两个库一起装好。

---

### 第二关：PDF 路径问题

代码里写的路径和实际文件名不一致，报错：

```
FileNotFoundError: [Errno 2] No such file or directory: '毛主席语录_1966.pdf'
```

用 `dir` 命令查看文件夹，发现 PDF 的实际文件名是：

```
《毛主席语录》1966年版 (毛泽东) (z-library.sk, 1lib.sk, z-lib.sk).pdf
```

把代码里的 `PDF_PATH` 改成实际文件名，问题解决。

---

### 第三关：成功运行分析脚本

bash

```bash
python analysis.py
```

输出：

```
共提取年份引用：350 条
年份范围：1911 — 1966
条形图已保存：figures/bar_chart.png
饼状图已保存：figures/pie_chart.png
--- 数据汇总 ---
井冈山／苏区 (1928–1934): 9 条 (2.6%)
延安 (1935–1945): 158 条 (45.1%)
解放战争 (1946–1949): 62 条 (17.7%)
建国后 (1950–): 115 条 (32.9%)
其他／更早: 6 条 (1.7%)
合计：350 条
```

`figures/` 文件夹里生成了条形图和饼状图两张图片。

---

### 第四关：提交到 GitHub

bash

```bash
git add .
git commit -m "添加分析代码与图表"
git push
```

提交成功，GitHub 网页刷新后所有文件都上传了。

---

### 第五关：试图修改 commit message — 翻车记录

看到满屏"添加分析代码与图表"不舒服，想改 commit message，用了：

bash

```bash
git rebase -i --root
```

在 Notepad++ 里把 `pick` 改成 `reword`，保存关闭后 git 弹出编辑界面，改完第一条后继续，报错：

```
error: The following untracked working tree files would be overwritten by merge:
        .obsidian/workspace.json
```

原因是 Obsidian 在后台一直修改 `workspace.json`，和 rebase 产生冲突。尝试：

bash

```bash
git rebase --abort
```

又报错，因为同样的文件被占用。手动删掉文件后 abort 成功：

bash

```bash
del .obsidian\workspace.json
git rebase --abort
```

---

### 第六关：用更简单的方法解决

放弃逐条改 message，直接把所有 commit 压缩成一条：

bash

```bash
git reset --soft 3cd8ad7
git add .
git commit -m "init: add analysis article and code"
git push --force
```

GitHub 上的历史变成了：

```
init: add analysis article and code
Initial commit
```

看着舒服多了。

---

### 最终成果

`marxism-works` 仓库现在包含：

|文件|内容|
|---|---|
|`《毛主席语录》出处时期分布分析.md`|中英双语研究文章|
|`analysis.py`|数据提取与可视化代码|
|`figures/bar_chart.png`|各时期引用次数条形图|
|`figures/pie_chart.png`|各时期占比饼状图|
|`《毛主席语录》1966年版.pdf`|原始数据来源|

---

### 这次学到了什么

- VSCode 终端可以直接运行 Python 脚本，不需要额外工具
- `pip install` 安装 Python 库
- `git add . && git commit && git push` 三步提交流程
- `git reset --soft` 可以压缩历史 commit
- `git push --force` 有风险，个人仓库可以用，协作仓库慎用
- Obsidian 在后台会一直修改文件，做 rebase 之前最好把 Obsidian 关掉

---

✊ 全世界无产者，联合起来！