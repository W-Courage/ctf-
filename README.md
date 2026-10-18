# CTF 赛历：可迁移完整源码包

这是从正在运行的赛历取得的源码与数据快照，不是页面截图。无需前端构建、Node.js 或外部 CDN。解压后即可在另一台服务器安装。

## 包内有什么

| 文件                                      | 用途                                                      |
| ----------------------------------------- | --------------------------------------------------------- |
| `app.py`                                  | Flask + SQLite 后端与 API                                 |
| `static/`                                 | HTML、CSS、JavaScript、图标、摆件、默认壁纸等全部前端文件 |
| `wallpapers/`                             | 截至打包时服务器上的全部用户上传壁纸                      |
| `data.db`                                 | 使用 SQLite 在线备份得到的数据快照：比赛、便签、外观设置  |
| `requirements.txt`                        | Python 依赖与版本                                         |
| `deploy.sh`、`saili.service`              | Ubuntu/Debian 一键部署、开机自启                          |
| `run-local.sh`                            | Linux/macOS 临时运行                                      |
| `Dockerfile`                              | 可选容器构建                                              |
| `SOURCE_MANIFEST.json`、`MANIFEST.sha256` | 原服务器快照和最终包校验清单                              |

打包快照时间：`2026-10-18 15:43:40`（服务器本地时间）。当时数据库有 **0 条比赛、1 条便签**；外观为摆件 610px、右侧，毛玻璃模糊 4px / 遮罩 31%，当前选中的上传壁纸也在 `wallpapers/` 内。**之后在旧服务器新增的数据不在本次包内**，需要重新导出或迁移。包内不含服务器登录密码、虚拟环境和操作系统软件。

## 新 Ubuntu/Debian 服务器：三步部署

建议 Ubuntu 24.04 / Python 3.12，需能连接系统软件源和 Python 包索引。安装依赖时会从这些源下载；运行页面时不依赖外部 CDN。

1. 把 `saili-portable-20261018.zip` 上传到新服务器，例如 `/home/ubuntu/`，并 SSH 登录。

2. 执行：

   ```bash
   cd /home/ubuntu
   sudo apt-get update && sudo apt-get install -y unzip
   unzip saili-portable-20261018.zip
   cd saili-portable-20261018
   sudo bash deploy.sh
   ```

3. 在云服务器安全组放行 **TCP 8000**，打开 `http://新服务器IP:8000/`。如果启用了 UFW，再执行 `sudo ufw allow 8000/tcp`。

安装脚本会安装 Python/venv/pip，建立 `saili` 系统账号、`/opt/saili/venv` 与 systemd 服务，复制源码及壁纸，**仅在目标 `data.db` 不存在时**恢复快照，然后启用开机自启并进行本机 HTTP 健康检查。它不会删除旧服务器的数据。重新运行部署脚本会更新代码和缺失壁纸，不覆盖目标服务器现有数据库。

## 检查和维护

```bash
sudo systemctl status saili --no-pager
curl -fsS http://127.0.0.1:8000/api/appearance
sudo journalctl -u saili -n 100 --no-pager
sudo systemctl restart saili
```

数据库及上传图片分别在 `/opt/saili/data.db`、`/opt/saili/wallpapers/`；修改代码后重启服务即可。默认监听 `0.0.0.0:8000`。要改端口，编辑 `/etc/systemd/system/saili.service` 的 `Environment=PORT=8000`，再执行 `sudo systemctl daemon-reload && sudo systemctl restart saili`，同时调整云防火墙。

## 只在本机运行

```bash
cd saili-portable-20261018
bash run-local.sh
```

Windows 可用 PowerShell：

```powershell
cd saili-portable-20261018
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe app.py
```

浏览器访问 `http://localhost:8000/`。本地运行会直接读写包内 `data.db`；想保留原始快照，可先复制整个文件夹。

## 已有新服务器数据时怎样迁移快照

部署脚本不会覆盖已有 `data.db`。确实要用包内快照替换目标数据时，先停服务并用 SQLite 在线备份保存目标数据库，再替换：

```bash
sudo systemctl stop saili
sudo /opt/saili/venv/bin/python -c "import sqlite3; a=sqlite3.connect('/opt/saili/data.db'); b=sqlite3.connect('/opt/saili/data-before-migration.db'); a.backup(b); b.close(); a.close()"
sudo install -o saili -g saili -m 644 data.db /opt/saili/data.db
sudo cp -n wallpapers/* /opt/saili/wallpapers/
sudo chown -R saili:saili /opt/saili/wallpapers
sudo systemctl start saili
```

其中 `data-before-migration.db` 是目标数据库的恢复点；若需撤销替换，停服务后用它恢复 `data.db` 并重启。复制壁纸时保留了目标服务器同名文件。

## 可选 Docker

```bash
docker build -t saili-calendar .
docker run -d --name saili-calendar -p 8000:8000 \
  -v "$PWD/data.db:/app/data.db" -v "$PWD/wallpapers:/app/wallpapers" \
  saili-calendar
```

数据通过两个 bind mount 持久化；不要仅把容器内的 `data.db` 当成备份。需要修改端口时同时修改 `-p` 的前半部分；应用内部仍监听 8000。

## 完整性和数据说明

在包目录执行 `sha256sum -c MANIFEST.sha256`，应逐项显示 `OK`。打包脚本还校验了 SQLite `PRAGMA quick_check=ok`、当前选中壁纸存在、页面和静态文件可加载。`data.db` 包含实际便签及外观，复制或分享压缩包前请按需决定是否保留这些记录。前端和后端使用同一份 `data.db`，不同访客看到的是服务器上的共享数据。
