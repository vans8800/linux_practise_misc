## wget 递归下载


要使用 wget递归下载一个 URL（包含子目录和重定向），需结合以下参数实现完整功能：

​​核心命令​​
wget -r -l inf -k -L -p -np -nd -P /保存路径/ "目标URL"
​​参数详解​​：
​​-r/ --recursive​​启用递归下载，抓取目标页面及其所有子目录内容。
​​-l inf/ --level=inf​​设置递归深度为无限（默认递归深度为 5 层），确保下载所有子目录。
​​-k/ --convert-links​​将下载的 HTML 页面中的链接转换为本地相对路径，便于离线浏览。
​​-L/ --relative​​仅跟踪相对链接，避免跳转到外部域名（如 www.example.com的链接不会触发下载 www.other.com）。
​​-p/ --page-requisites​​下载页面所需的所有资源（如图片、CSS、JS 等）。
​​-np/ --no-parent​​不追溯父目录，仅下载指定 URL 及其子目录内容。
​​-nd/ --no-directories​​不创建目录结构，所有文件直接保存到指定路径（若需保留原始目录结构，可省略此参数）。
​​-P /保存路径/​​指定本地保存路径（默认当前目录）。
​​扩展功能​​
​​1. 处理重定向​​
​​自动处理重定向​​：wget默认会自动跟随 HTTP 重定向（如 301/302），无需额外参数。
​​强制限制重定向范围​​：若需仅跟随特定域名的重定向，可结合 -D参数：
wget -r -l inf -k -L -D example.com,sub.example.com "http://example.com"
​​2. 保留原始目录结构​​
移除 -nd参数，并添加 --cut-dirs=N简化路径层级：

wget -r -l inf -k -L -p -np --cut-dirs=1 -P /保存路径/ "http://example.com/dir/"
--cut-dirs=1：忽略 URL 中的前 1 级目录（如 http://example.com/dir/下载到 /保存路径/dir/）。
​​3. 过滤文件类型​​
​​仅下载特定文件​​：通过 -A指定扩展名（如下载 PDF 和图片）：
wget -r -l inf -k -L -p -np -A "*.pdf,*.jpg" -P /保存路径/ "http://example.com"
​​排除特定文件​​：通过 -R排除文件（如跳过 index.html）：
wget -r -l inf -k -L -p -np -R "index.html" -P /保存路径/ "http://example.com"
​​4. 断点续传与限速​​
​​断点续传​​：使用 -c参数从中断处继续下载：
wget -c -r -l inf -k -L -p -np -P /保存路径/ "http://example.com"
​​限速下载​​：通过 --limit-rate限制带宽（如 100KB/s）：
wget --limit-rate=100k -r -l inf -k -L -p -np -P /保存路径/ "http://example.com"
​​完整示例​​
wget -r -l inf -k -L -p -np -P /home/user/website_backup/ \
     --cut-dirs=1 -A "*.html,*.css,*.js,*.jpg" \
     "http://old-website.com"
​​效果​​：递归下载 http://old-website.com下所有子目录和文件，保留原始目录结构（忽略 URL 中的 old-website.com一级目录），仅保存 HTML/CSS/JS/图片文件，并保存到 /home/user/website_backup/。
​​注意事项​​
​​合法性与权限​​：确保目标网站允许爬取，避免违反 robots.txt或版权规定。
​​服务器负载​​：添加 --wait=2或 --random-wait参数降低请求频率，减轻服务器压力。
​​调试信息​​：使用 -o 日志文件记录下载过程：
wget -o download.log -r -l inf ... "http://example.com"
通过上述参数组合，可实现高效、可控的递归下载，适用于网站备份、离线浏览等场景。

