# Python 环境安装问题复盘（pymysql/venv/pip）- 2026-03-09

## 背景
在执行“MySQL → 统计/筛选 →（可选）LLM 打标 → 输出文件/写 ES”任务时，脚本依赖 `pymysql` 连接 MySQL。

早先在某次会话中能够 `import pymysql` 并执行成功，但后续在当前执行环境中出现：
- `ModuleNotFoundError: No module named 'pymysql'`
- 且无法创建 `venv`
- 且系统没有 `pip`

导致无法在本机直接跑 MySQL 相关 Python 脚本。

---

## 问题现象
### 1) 系统 Python 缺 pip
- `/usr/bin/python3 -m pip` 报错：`No module named pip`

### 2) 无法创建 venv
- `python3 -m venv .venv` 报错：
  - `ensurepip is not available`
  - 提示需要安装 `python3.11-venv`

原因：系统缺 `venv/ensurepip` 组件，而安装需要 apt 权限。

### 3) 无 sudo / 无 apt 权限
- `sudo -n true` 返回 `NO_SUDO`
- 因此不能通过 `apt install python3.11-venv python3-pip` 来补齐。

### 4) PEP 668 限制（Externally Managed Environment）
即使使用 `get-pip.py` 安装 pip，也会遇到 PEP 668 保护：
- `error: externally-managed-environment`

---

## 解决方案（最终可行且不需要 sudo）

核心思路：**使用仓库内已有的 `get-pip.py`，在用户目录安装 pip，并用 `--break-system-packages` 绕过 PEP 668 限制**。

### Step 1) 安装 pip（用户目录）
仓库已有：`/home/node/.openclaw/workspace/.tmp/get-pip.py`

命令：
```bash
python3 /home/node/.openclaw/workspace/.tmp/get-pip.py --user --break-system-packages
```
结果：
- pip/setuptools/wheel 安装到：`/home/node/.local/`
- pip 可执行文件在：`/home/node/.local/bin/pip3`

> 注意：会提示 `~/.local/bin` 不在 PATH；可用完整路径调用。

### Step 2) 安装 pymysql（用户目录）
命令：
```bash
/home/node/.local/bin/pip3 install --user pymysql --break-system-packages
```

### Step 3) 验证
```bash
python3 -c "import pymysql; print(pymysql.__version__)"
python3 - <<'PY'
import os, pymysql
conn=pymysql.connect(host=os.environ['NETA__MYSQL_HOST_OFFLINE'],user=os.environ['NETA__MYSQL_USER'],password=os.environ['NETA__MYSQL_PASSWORD'],database=os.environ.get('NETA__MYSQL_DB','talesofai'),charset='utf8mb4')
cur=conn.cursor(); cur.execute('SELECT 1'); print(cur.fetchone())
cur.close(); conn.close()
PY
```

验证通过后即可在本机执行依赖 pymysql 的统计/筛选脚本。

---

## 经验与建议
1) **不要假设“同一台机器/同一 python”永远有同样的 site-packages**：不同 runtime/镜像可能导致依赖忽隐忽现。
2) **venv 依赖系统包**（Debian/Ubuntu 的 `python3.x-venv`），没有 sudo 时无法使用 venv，是常见卡点。
3) **PEP 668** 会阻止 pip 直接改系统环境；
   - 若无 venv 能力，可用 `get-pip.py --break-system-packages` + `pip install --break-system-packages` 在用户目录落包。
4) 长期建议：把跑批任务放到一个固定镜像（自带 venv+pymysql+requests），避免运行时环境漂移。

---

## 本次关联任务
- 在完成上述环境修复后，已能按条件在 MySQL 中筛选：
  - 指定 hashtags 下精选（HIGHLIGHT）
  - review_status=PASS, status=PUBLISHED, recommend_score>0
  - 并统计 same_style_count 分布区间。
