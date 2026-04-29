================================================================================
                        GitHacker 工具使用手册
================================================================================

一、工具简介
--------------------------------------------------------------------------------
GitHacker 是一个多线程 .git 文件夹泄露漏洞利用工具。
- 能几乎完整下载目标 .git 文件夹（源代码、提交历史、分支、标签、stash、reflog）
- 即使服务器关闭了 DirectoryListing（目录浏览），也能通过暴力破解下载常用文件
- 支持多线程、支持批量目标
- 推荐在 Docker 容器中运行（目标 .git 文件夹可能是恶意的）


二、环境要求
--------------------------------------------------------------------------------
- Python 3
- git >= 2.11.0


三、安装方式
--------------------------------------------------------------------------------
# pip 安装
python3 -m pip install -i https://pypi.org/simple/ GitHacker


四、常用参数（命令行）
--------------------------------------------------------------------------------

必选参数（二选一）：
  --url URL                  目标网站的 .git 文件夹 URL
                             示例: http://example.com/.git/
  --url-file URL_FILE        包含多个目标 URL 的文件，每行一个

必选参数：
  --output-folder DIR        存放下载结果的本地文件夹
                             每个仓库以 md5(url) 命名存放在该目录下

可选参数：
  --brute                    启用暴力破解分支名/标签名
                             当服务器关闭目录浏览时使用

  --threads THREADS          下载线程数（默认多个线程）

  --delay DELAY              HTTP 请求之间的延迟秒数
                             避免触发 WAF / 频率限制

  --enable-manually-check-dangerous-git-files
                             禁用危险文件自动检测
                             如果使用该参数，GitHacker 将不会下载可能
                             导致 RCE 的危险文件（如 .git/config, .git/hooks/*）
                             默认行为是下载这些文件（不检查）

  --version                  显示版本号

  -h, --help                 显示帮助信息


五、常用用法示例
--------------------------------------------------------------------------------

# 1. 基础用法 — 下载指定目标的 .git 文件夹
githacker --url http://127.0.0.1/.git/ --output-folder result

# 2. 暴力破解 — 服务器关闭目录浏览时（尝试常用分支名和标签名）
githacker --brute --url http://127.0.0.1/.git/ --output-folder result

# 3. 批量目标 — 从文件读取多个 URL
githacker --brute --url-file websites.txt --output-folder result

# 4. 增加延迟 — 避免触发 WAF / 请求频率限制
githacker --url http://target.com/.git/ --output-folder result --delay 0.5

# 5. 自定义线程数
githacker --url http://target.com/.git/ --output-folder result --threads 20

# 6. Docker 方式运行（推荐，安全隔离）
docker run wangyihang/githacker --help
docker run -v $(pwd)/results:/tmp/githacker/results wangyihang/githacker \
    --output-folder /tmp/githacker/results --url http://127.0.0.1/.git/
docker run -v $(pwd)/results:/tmp/githacker/results wangyihang/githacker \
    --brute --output-folder /tmp/githacker/results --url http://127.0.0.1/.git/
docker run -v $(pwd)/results:/tmp/githacker/results wangyihang/githacker \
    --brute --output-folder /tmp/githacker/results --url-file websites.txt

# 7. 批量目标文件格式（websites.txt）
http://target1.com/.git/
http://target2.com/.git/
http://target3.com/path/.git/


六、与其他工具的对比
--------------------------------------------------------------------------------
GitHacker 是目前功能最全的 .git 泄露利用工具，即使在目录浏览关闭时也能工作：

功能项       | GitHack | GitTools | dvcs-ripper | git-dumper | GitHacker
源文件       |   Y     |    Y     |     Y       |     Y      |    Y
Reflog       |   N     |    Y     |     Y       |     Y      |    Y
Stash        |   N     |    N     |     N       |     Y      |    Y
Commits      |   N     |    Y     |     Y       |     Y      |    Y
Branches     |   N     |    N     |     N       |     Y      |    Y(支持爆破)
Tags         |   N     |    N     |     N       |     Y      |    Y(支持爆破)
Remotes      |   N     |    Y     |     Y       |     Y      |    Y


================================================================================
             获取 .git 目录后 —— 渗透测试常用 Git 命令
================================================================================

一、仓库整体信息
--------------------------------------------------------------------------------
git status                          # 查看当前仓库状态（哪些文件被修改/删除）
git remote -v                       # 查看远程仓库地址（可能泄露内部 git 服务器地址）
git config --list                   # 查看所有 Git 配置（可能包含账号/邮箱/Token）
git config --global --list          # 查看全局 Git 配置
git ls-files                        # 列出仓库跟踪的所有文件
git ls-files --others               # 列出未跟踪的文件


二、提交历史分析
--------------------------------------------------------------------------------
git log                             # 查看完整提交历史
git log --oneline                   # 单行简洁提交历史（适合快速浏览）
git log --oneline --all             # 查看所有分支的提交历史
git log --graph --all --oneline     # 图形化显示全部分支的提交树
git log -p                          # 查看每次提交的完整 diff（代码差异）
git log --stat                      # 查看每次提交变更的文件统计
git log --all --full-history -- **/secret*  # 搜索某个文件的历史提交
git log --diff-filter=D --summary   # 查找所有被删除的文件及其删除记录
git log -S "password"               # 搜索添加或删除了 "password" 字符串的提交
git log -G "api_key|secret|token"   # 通过正则搜索有敏感信息的提交（推荐）
git log -G "TODO|FIXME|HACK"        # 搜索包含 TODO/FIXME/HACK 注释的提交
git log --since="2023-01-01"        # 查看某个日期之后的提交
git log --until="2023-01-01"        # 查看某个日期之前的提交
git log --author="dev_name"         # 查看某个作者的所有提交
git log --grep="bug|fix|vuln"       # 通过提交信息关键词搜索
git show <commit-hash>              # 查看某个具体提交的详细信息（完整 diff）
git show <commit-hash>:<file-path>  # 查看某个提交中某个文件的内容
git show --name-only <commit-hash>  # 只查看某次提交改了哪些文件
git reflog                          # 查看引用日志（可能包含被删除的分支信息）


三、分支与标签
--------------------------------------------------------------------------------
git branch -a                       # 列出所有分支（包括远程分支）
git branch -r                       # 只列出远程分支
git tag                            # 列出所有标签（可能对应发布版本）
git tag -l "*release*"             # 搜索特定模式的标签
git checkout <branch-name>          # 切换到指定分支
git checkout -b restore <commit>    # 从某个提交创建新分支（恢复历史代码）
git diff master..feature-branch     # 比较两个分支之间的差异
git merge-base master feature-branch # 查找两个分支的共同祖先


四、Stash 分析（开发者暂存的未提交代码）
--------------------------------------------------------------------------------
git stash list                      # 列出所有 stash（开发者暂存的修改）★很重要
git stash show -p                   # 查看最新 stash 的完整内容（可能是未提交的敏感代码）
git stash show -p stash@{0}         # 查看指定 stash 的完整内容
git stash show -p stash@{1}
git stash show -p stash@{2}


五、文件内容搜索（在历史版本中查找敏感信息）
--------------------------------------------------------------------------------
git grep "password" $(git rev-list --all)                          # 在所有提交中搜索 "password"
git grep -i "connectionString" $(git rev-list --all)               # 搜索数据库连接字符串
git grep -i "jdbc|mongodb|redis|mysql" $(git rev-list --all)       # 搜索数据库连接
git grep -i "api[_-]?key|api[_-]?secret" $(git rev-list --all)     # 搜索 API 密钥
git grep -i "AKIA[0-9A-Z]{16}" $(git rev-list --all)              # 搜索 AWS Access Key
git grep -i "sk-[a-zA-Z0-9]{20,}" $(git rev-list --all)           # 搜索 OpenAI / 类 API Key
git grep -i "BEGIN.*PRIVATE KEY" $(git rev-list --all)             # 搜索私钥
git grep -i "token|auth|bearer" $(git rev-list --all)              # 搜索 Token 信息
git grep -i "secret|private|credential" $(git rev-list --all)      # 搜索凭证关键词
git grep -i "ftp://|ssh://|mysql://" $(git rev-list --all)         # 搜索硬编码 URL


六、恢复已删除的文件
--------------------------------------------------------------------------------
git log --diff-filter=D --summary                           # 找哪些文件被删除过
git checkout <commit-hash>^ -- <deleted-file-path>          # 从删除前的提交恢复文件
git rev-list -n 1 -- <file-path>                            # 查看文件最后一次存在的提交
git show <commit-hash>:<file-path>                          # 查看某个提交中文件的内容


七、底层对象操作（深入检查 Git 对象数据库）
--------------------------------------------------------------------------------
git fsck                                 # 检查仓库完整性（可能发现损坏或孤立的对象）
git fsck --lost-found                    # 查找悬空对象，输出到 .git/lost-found/
git cat-file -p <object-hash>            # 查看任意 Git 对象内容
git cat-file -t <object-hash>            # 查看 Git 对象类型（blob/tree/commit/tag）
git rev-list --all                       # 列出所有提交的 hash
git rev-list --objects --all             # 列出所有对象（包括 blob 和 tree）
git count-objects -v                     # 统计对象数量


八、开发者信息收集（用于社工或字典生成）
--------------------------------------------------------------------------------
git log --format='%an' | sort -u         # 提取所有提交者名字（去重）
git log --format='%ae' | sort -u         # 提取所有提交者邮箱（去重）
git log --format='%an %ae' | sort -u     # 提取所有名字+邮箱组合
git shortlog -sne                        # 统计每个开发者的提交次数和邮箱


九、特定敏感文件核对清单
--------------------------------------------------------------------------------
# 进入下载的仓库目录后，先检查是否存在以下文件:
.env                    # 环境变量（DB密码、API Key）
.env.local              # 本地环境变量
.env.production         # 生产环境变量
config.php              # PHP 配置文件
web.config              # ASP.NET 配置
application.properties  # Java Spring 配置
settings.py             # Django 配置
database.yml            # Rails 数据库配置
credentials.json        # GCP/AWS 凭证文件
id_rsa / id_ed25519     # SSH 私钥
*.pem / *.key           # 证书和密钥文件
.git-credentials        # Git 凭据存储
.npmrc                  # NPM 私有源 Token
.pypirc                 # PyPI 上传凭证
.docker/config.json     # Docker 仓库凭证


十、推荐的渗透测试流程
--------------------------------------------------------------------------------
1. git log --oneline                          → 快速了解项目规模和开发活动
2. git log -G "password|secret|token|key"     → 在历史中搜索敏感信息
3. git stash list && git stash show -p        → 查看开发者暂存的未提交代码 ★
4. git branch -a && git tag                   → 查看所有分支和发布版本
5. git reflog                                 → 查看引用日志（可能发现隐藏分支）
6. git config --list                          → 查看配置中的敏感信息
7. 检查 .env / config 等敏感配置文件          → 可能包含生产环境凭据
8. git log --diff-filter=D --summary          → 查找曾经被删除的敏感文件
9. git show <commit> -- <sensitive-file>      → 尝试恢复已删除的敏感文件
10. git fsck --lost-found                     → 查找孤立/悬空的 git 对象

================================================================================
