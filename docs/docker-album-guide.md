# Docker 部署与专辑封面获取指南

本文档详细说明如何将 qq-music-api 部署为 Docker 容器，以及如何使用 API 获取专辑封面和元数据信息。

## 目录

- [Docker 部署](#docker-部署)
  - [Dockerfile 分析](#dockerfile-分析)
  - [构建镜像](#构建镜像)
  - [运行容器](#运行容器)
  - [Linux 环境部署](#linux-环境部署)
- [获取专辑封面与元数据](#获取专辑封面与元数据)
  - [实现流程](#实现流程)
  - [通过搜索获取 ID](#通过搜索获取-id)
  - [获取专辑详细信息](#获取专辑详细信息)
  - [获取专辑封面](#获取专辑封面)
  - [获取歌手专辑列表](#获取歌手专辑列表)
  - [完整实现方案](#完整实现方案)
- [注意事项](#注意事项)

---

## Docker 部署

### Dockerfile 分析

项目使用的 Dockerfile 配置如下：

```dockerfile
FROM node:22

LABEL maintainer="Rain120 <1085131904@qq.com>"

# Create app directory
WORKDIR /app

COPY package.json .

RUN yarn install --registry=https://registry.npmmirror.com

COPY . .

EXPOSE 3200

ENTRYPOINT ["npm", "run"]

CMD ["start"]
```

**说明：**

- 基础镜像：node:22
- 工作目录：/app
- 使用国内镜像源加速依赖安装
- 暴露端口：3200
- 默认启动命令：`npm run start`

### 构建镜像

**本地构建：**

```bash
docker build -t qq-music-api:1.0.5 .
```

**使用脚本构建：**

```bash
# 构建本地镜像
npm run build:local-images

# 构建并标记为远程镜像
npm run build:remote-images

# 同时构建本地和远程镜像
npm run build:images
```

**脚本执行流程：** (scripts/build-images.js:9-12)

- 本地构建：`docker build -t qq-music-api:1.0.5 .`
- 远程构建：`docker image tag qq-music-api:1.0.5 rain120/qq-music-api:1.0.5`

### 运行容器

**基础运行：**

```bash
docker run -d --name qq-music-api -p 3200:3200 qq-music-api
```

**可配置参数：**

| 参数        | 说明                   | 示例                             |
| ----------- | ---------------------- | -------------------------------- |
| `--name`    | 容器名称               | `--name qq-music-api`            |
| `-p`        | 端口映射 (宿主机:容器) | `-p 3200:3200` 或 `-p 8080:3200` |
| `-d`        | 后台运行               | `-d`                             |
| `-v`        | 卷挂载 (宿主机:容器)   | `-v /path/to/config:/app/config` |
| `-e`        | 环境变量               | `-e NODE_ENV=production`         |
| `--restart` | 重启策略               | `--restart always`               |
| `--network` | 网络模式               | `--network bridge`               |
| `--memory`  | 内存限制               | `--memory 512m`                  |
| `--cpus`    | CPU 限制               | `--cpus 1.0`                     |

**完整配置示例：**

```bash
docker run -d \
  --name qq-music-api \
  --restart always \
  -p 3200:3200 \
  -e NODE_ENV=production \
  -v /path/to/config:/app/config \
  qq-music-api
```

**使用脚本运行：**

```bash
npm run run:images
# 等同于: docker run -d --name qq-music-api -p 3200:3200 qq-music-api
```

### Linux 环境部署

**步骤 1：准备代码**

```bash
# 克隆或上传代码到服务器
cd /path/to/qq-music-api
```

**步骤 2：配置用户信息（可选）**

```bash
# 配置 QQ Music Cookie，用于某些高级功能
vim config/user-info.js
```

配置格式：

```javascript
const userInfo = {
	loginUin: 'qq号码',
	cookie: '从浏览器复制的Cookie',
};
```

**步骤 3：构建镜像**

```bash
# 方式一：使用 Dockerfile 直接构建
docker build -t qq-music-api:1.0.5 .

# 方式二：使用 npm 脚本
npm run build:local-images
```

**步骤 4：运行容器**

```bash
# 启动容器
docker run -d --name qq-music-api -p 3200:3200 qq-music-api

# 查看容器状态
docker ps

# 查看日志
docker logs qq-music-api

# 进入容器
docker exec -it qq-music-api /bin/bash
```

**步骤 5：验证服务**

```bash
# 测试接口
curl http://localhost:3200/getHotkey

# 应返回 JSON 格式的热词数据
```

**常用 Docker 命令：**

```bash
# 停止容器
docker stop qq-music-api

# 启动容器
docker start qq-music-api

# 重启容器
docker restart qq-music-api

# 删除容器
docker rm qq-music-api

# 删除镜像
docker rmi qq-music-api:1.0.5

# 查看容器资源使用
docker stats qq-music-api
```

---

## 获取专辑封面与元数据

### 实现流程

```
FLAC 文件 → 提取歌手名/专辑名 → 搜索 API → 获取 ID
  ↓
获取专辑信息 → 下载封面 → 嵌入 FLAC 文件
  ↓
写入发行年份
```

### 通过搜索获取 ID

**API 端点：** `GET /getSearchByKey`

**路由定义：** (routers/router.js:14)

**参数说明：** (routers/context/getSearchByKey.js:7-19)

| 参数          | 类型   | 必填 | 说明                         | 默认值 |
| ------------- | ------ | ---- | ---------------------------- | ------ |
| `key`         | string | 是   | 搜索关键字（歌手名或专辑名） | -      |
| `page`        | number | 否   | 页码                         | 1      |
| `limit`       | number | 否   | 每页数量                     | 10     |
| `remoteplace` | string | 否   | 搜索类型                     | song   |
| `catZhida`    | number | 否   | 保留参数                     | 1      |

**remoteplace 选项：**

| 值         | 说明                           |
| ---------- | ------------------------------ |
| `song`     | 单曲（默认）                   |
| `album`    | 专辑 ⭐ **获取专辑封面时使用** |
| `mv`       | MV                             |
| `playlist` | 歌单                           |
| `user`     | 用户                           |
| `lyric`    | 歌词                           |

**请求示例：**

```bash
# 搜索专辑
curl "http://localhost:3200/getSearchByKey?key=叶惠美&remoteplace=album&page=1&limit=10"

# 搜索歌手
curl "http://localhost:3200/getSearchByKey?key=周杰伦&remoteplace=song&page=1&limit=10"

# 搜索歌手名和专辑名组合
curl "http://localhost:3200/getSearchByKey?key=周杰伦+叶惠美&remoteplace=album&page=1&limit=10"
```

**响应数据结构：**

```json
{
	"response": {
		"code": 0,
		"data": {
			"song": {
				"list": [
					{
						"songid": 12345678,
						"songmid": "003rJSwm3TechU",
						"songname": "晴天",
						"singername": "周杰伦",
						"singermid": "0025NhlN2yWrP4",
						"albumname": "叶惠美",
						"albummid": "000MkMni19ClKG",
						"interval": 288,
						"time_public": "2003"
					}
				],
				"totalnum": 1,
				"curpage": 1
			}
		}
	}
}
```

**关键字段说明：**

- `albummid`: 专辑 ID ⭐ **用于获取封面**
- `singermid`: 歌手 ID ⭐ **用于获取歌手专辑列表**
- `albumname`: 专辑名称
- `singername`: 歌手名称
- `time_public`: 发行时间

### 获取专辑详细信息

**API 端点：** `GET /getAlbumInfo`

**路由定义：** (routers/router.js:83)

**参数说明：** (routers/context/getAlbumInfo.js:5-11)

| 参数       | 类型   | 必填 | 说明    |
| ---------- | ------ | ---- | ------- |
| `albummid` | string | 是   | 专辑 ID |

**请求示例：**

```bash
curl "http://localhost:3200/getAlbumInfo?albummid=000MkMni19ClKG"
```

**响应数据结构：**

```json
{
	"response": {
		"code": 0,
		"data": {
			"list": [
				{
					"albumname": "叶惠美",
					"albummid": "000MkMni19ClKG",
					"singername": "周杰伦",
					"singermid": "0025NhlN2yWrP4",
					"pub_time": 20030731,
					"company": "新力博德曼音乐娱乐(中国)有限公司",
					"genre": "流行",
					"language": "国语",
					"desc": "《叶惠美》是周杰伦的第四张专辑...",
					"songlist": [
						{
							"songname": "以父之名",
							"songmid": "003rJSwm3TechU",
							"interval": 340
						}
					]
				}
			]
		}
	}
}
```

**关键字段说明：**

- `pub_time`: 发行时间戳 ⭐ **转换为 YYYY-MM-DD 格式**
- `albumname`: 专辑名称
- `singername`: 歌手名称
- `genre`: 音乐类型
- `language`: 语言
- `songlist`: 专辑歌曲列表

### 获取专辑封面

**方案 A：使用项目提供的 getImageUrl 接口** ⭐ **推荐**

**API 端点：** `GET /getImageUrl`

**路由定义：** (routers/router.js:106)

**参数说明：** (routers/context/getImageUrl.js:15-21)

| 参数     | 类型   | 必填 | 说明              | 默认值  |
| -------- | ------ | ---- | ----------------- | ------- |
| `id`     | string | 是   | 专辑 ID 或歌曲 ID | -       |
| `size`   | string | 否   | 图片尺寸          | 300x300 |
| `maxAge` | number | 否   | 缓存时间（毫秒）  | 2592000 |

**size 支持的尺寸：**

- `300x300` - 默认尺寸
- `500x500` - 推荐尺寸
- `800x800` - 高清尺寸
- `1000x1000` - 超高清尺寸

**请求示例：**

```bash
# 获取默认尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG"

# 获取 500x500 尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG&size=500x500"

# 获取 800x800 尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG&size=800x800"
```

**响应数据结构：**

```json
{
	"response": {
		"code": 0,
		"data": {
			"imageUrl": "https://y.gtimg.cn/music/photo_new/T002R500x500M000000MkMni19ClKG.jpg?max_age=2592000"
		}
	}
}
```

**方案 B：使用 QQ Music 图片 URL 格式**

```
http://i.gtimg.cn/music/photo/mid_album_500/7/a/专辑ID.jpg
```

**示例：**

```
http://i.gtimg.cn/music/photo/mid_album_500/7/a/000MkMni19ClKG.jpg
```

**两种方案对比：**

| 方案             | 优点                       | 缺点                 |
| ---------------- | -------------------------- | -------------------- |
| getImageUrl 接口 | 支持自定义尺寸，带缓存参数 | 需要调用 API         |
| 直接 URL 格式    | 简单直接                   | 尺寸固定，无缓存控制 |

### 获取歌手专辑列表

如果你有歌手 ID，可以直接获取该歌手的所有专辑。

**API 端点：** `GET /getSingerAlbum`

**路由定义：** (routers/router.js:53)

**参数说明：** (routers/context/getSingerAlbum.js:5-22)

| 参数        | 类型   | 必填 | 说明     | 默认值 |
| ----------- | ------ | ---- | -------- | ------ |
| `singermid` | string | 是   | 歌手 ID  | -      |
| `limit`     | number | 否   | 每页数量 | 5      |
| `page`      | number | 否   | 页码     | 0      |

**请求示例：**

```bash
# 获取歌手的前 20 张专辑
curl "http://localhost:3200/getSingerAlbum?singermid=0025NhlN2yWrP4&limit=20&page=0"

# 获取第二页
curl "http://localhost:3200/getSingerAlbum?singermid=0025NhlN2yWrP4&limit=20&page=1"
```

**响应数据结构：**

```json
{
	"response": {
		"code": 0,
		"data": {
			"list": [
				{
					"albumname": "叶惠美",
					"albummid": "000MkMni19ClKG",
					"pub_time": 20030731,
					"public_time": "2003-07-31"
				}
			]
		}
	}
}
```

### 完整实现方案

以下是一个完整的 Python 实现，演示如何从 FLAC 文件获取专辑信息、下载封面并嵌入文件。

```python
import requests
import time
from mutagen.flac import FLAC, Picture
from mutagen.id3 import ID3, APIC
from mutagen.mp3 import MP3
import io

# 配置
BASE_URL = "http://localhost:3200"

def search_album(artist, album_name):
    """
    搜索专辑获取 ID

    Args:
        artist: 歌手名
        album_name: 专辑名

    Returns:
        dict: 包含 albummid, singermid, albumname, singername
    """
    # 构建搜索关键词
    search_key = f"{artist} {album_name}".strip()

    # 调用搜索 API
    response = requests.get(
        f"{BASE_URL}/getSearchByKey",
        params={
            "key": search_key,
            "remoteplace": "album",
            "page": 1,
            "limit": 10
        },
        timeout=10
    )

    data = response.json()

    # 检查是否有结果
    if "response" not in data or "data" not in data["response"]:
        raise Exception("搜索失败：无返回数据")

    songs = data["response"]["data"]["song"]["list"]
    if not songs:
        raise Exception(f"未找到专辑：{search_key}")

    # 返回第一个匹配项
    first_match = songs[0]
    return {
        "albummid": first_match.get("albummid"),
        "singermid": first_match.get("singermid"),
        "albumname": first_match.get("albumname"),
        "singername": first_match.get("singername")
    }


def get_album_info(albummid):
    """
    获取专辑详细信息

    Args:
        albummid: 专辑 ID

    Returns:
        dict: 包含专辑详细信息
    """
    response = requests.get(
        f"{BASE_URL}/getAlbumInfo",
        params={"albummid": albummid},
        timeout=10
    )

    data = response.json()

    if "response" not in data or "data" not in data["response"]:
        raise Exception("获取专辑信息失败")

    album_info = data["response"]["data"]["list"][0]

    # 处理发行时间
    pub_time = album_info.get("pub_time")
    if pub_time:
        pub_time_str = str(pub_time)
        pub_year = pub_time_str[:4] if len(pub_time_str) >= 4 else ""
    else:
        pub_year = ""

    return {
        "albumname": album_info.get("albumname"),
        "singername": album_info.get("singername"),
        "pub_time": pub_time,
        "pub_year": pub_year,
        "genre": album_info.get("genre"),
        "language": album_info.get("language"),
        "desc": album_info.get("desc")
    }


def get_album_cover(albummid, size="500x500"):
    """
    获取专辑封面

    Args:
        albummid: 专辑 ID
        size: 图片尺寸，默认 500x500

    Returns:
        bytes: 图片数据
    """
    response = requests.get(
        f"{BASE_URL}/getImageUrl",
        params={
            "id": albummid,
            "size": size
        },
        timeout=10
    )

    data = response.json()

    if "response" not in data or "data" not in data["response"]:
        raise Exception("获取封面 URL 失败")

    image_url = data["response"]["data"]["imageUrl"]

    # 下载图片
    image_response = requests.get(image_url, timeout=10)
    image_response.raise_for_status()

    return image_response.content


def embed_cover_to_flac(flac_path, cover_data, metadata=None):
    """
    将封面嵌入到 FLAC 文件

    Args:
        flac_path: FLAC 文件路径
        cover_data: 图片数据（bytes）
        metadata: 要写入的元数据，如 {"DATE": "2003"}
    """
    # 打开 FLAC 文件
    audio = FLAC(flac_path)

    # 清除现有图片
    audio.clear_pictures()

    # 创建图片对象
    image = Picture()
    image.type = 3  # 3 表示封面
    image.mime = "image/jpeg"
    image.width = 500
    image.height = 500
    image.data = cover_data

    # 添加图片
    audio.add_picture(image)

    # 添加元数据
    if metadata:
        for key, value in metadata.items():
            audio[key] = value

    # 保存
    audio.save()


def process_flac_file(flac_path):
    """
    处理 FLAC 文件：获取专辑封面并嵌入

    Args:
        flac_path: FLAC 文件路径
    """
    try:
        # 读取 FLAC 文件元数据
        audio = FLAC(flac_path)
        artist = audio.get("ARTIST", [None])[0]
        album = audio.get("ALBUM", [None])[0]

        if not artist or not album:
            print(f"⚠️  文件 {flac_path} 缺少 ARTIST 或 ALBUM 标签")
            return

        print(f"📝 处理文件：{flac_path}")
        print(f"   歌手：{artist}")
        print(f"   专辑：{album}")

        # 步骤 1：搜索专辑获取 ID
        print("🔍 搜索专辑...")
        album_info = search_album(artist, album)
        albummid = album_info["albummid"]

        # 步骤 2：获取专辑详细信息
        print("📋 获取专辑详细信息...")
        detailed_info = get_album_info(albummid)
        pub_year = detailed_info["pub_year"]

        # 步骤 3：获取专辑封面
        print("🖼️  获取专辑封面...")
        cover_data = get_album_cover(albummid, size="500x500")

        # 步骤 4：嵌入到 FLAC 文件
        print("💾 嵌入封面和元数据...")
        metadata = {}
        if pub_year:
            metadata["DATE"] = pub_year

        embed_cover_to_flac(flac_path, cover_data, metadata)

        print(f"✅ 完成！已嵌入封面和发行年份：{pub_year}")

        # 添加延迟，避免请求过快
        time.sleep(1)

    except Exception as e:
        print(f"❌ 处理失败：{e}")


# 批量处理示例
if __name__ == "__main__":
    import os
    import glob

    # 处理当前目录下所有 FLAC 文件
    flac_files = glob.glob("*.flac")

    print(f"📁 找到 {len(flac_files)} 个 FLAC 文件\n")

    for flac_file in flac_files:
        process_flac_file(flac_file)
        print()

    print("🎉 所有文件处理完成！")
```

**使用方法：**

1. 安装依赖：

```bash
pip install requests mutagen
```

2. 运行脚本：

```bash
python process_flac.py
```

**输出示例：**

```
📁 找到 3 个 FLAC 文件

📝 处理文件：song1.flac
   歌手：周杰伦
   专辑：叶惠美
🔍 搜索专辑...
📋 获取专辑详细信息...
🖼️  获取专辑封面...
💾 嵌入封面和元数据...
✅ 完成！已嵌入封面和发行年份：2003

📝 处理文件：song2.flac
   歌手：周杰伦
   专辑：七里香
🔍 搜索专辑...
📋 获取专辑详细信息...
🖼️  获取专辑封面...
💾 嵌入封面和元数据...
✅ 完成！已嵌入封面和发行年份：2004

🎉 所有文件处理完成！
```

---

## 注意事项

### Cookie 配置

部分高级功能需要配置 QQ Music Cookie。

**获取 Cookie 方法：**

1. 打开 [QQ音乐网页版](https://y.qq.com/)
2. 登录账号
3. 按 F12 打开开发者工具
4. 切换到 Network 标签
5. 刷新页面，找到任一 API 请求
6. 复制请求头中的 Cookie

**配置方法：**

编辑 `config/user-info.js`：

```javascript
const userInfo = {
	loginUin: '你的QQ号码',
	cookie: '复制的Cookie字符串',
};
```

**测试 Cookie：**

```bash
curl "http://localhost:3200/user/getCookie"
```

### API 限制

**请求频率限制：**

- QQ Music 可能有请求频率限制
- 建议每次请求之间添加延迟（如 1-2 秒）
- 避免短时间内大量请求

**示例：**

```python
time.sleep(1)  # 每次请求后延迟 1 秒
```

### ID 匹配问题

**搜索结果可能有多个匹配项：**

- 按相似度排序，通常第一个最准确
- 可以根据歌手名和专辑名进行二次筛选
- 考虑添加模糊匹配算法

**示例：**

```python
def find_best_match(songs, artist, album_name):
    """从搜索结果中找到最佳匹配"""
    for song in songs:
        if song.get("singername") == artist and song.get("albumname") == album_name:
            return song
    # 如果没有精确匹配，返回第一个
    return songs[0] if songs else None
```

### 封面尺寸选择

**推荐尺寸：**

- `300x300` - 默认，适合小图
- `500x500` - 推荐，平衡质量和大小
- `800x800` - 高清，适合大图
- `1000x1000` - 超高清，文件较大

**建议：**

- 如果存储空间充足，使用 `800x800` 或更高
- 如果需要快速加载，使用 `300x300` 或 `500x500`

### Docker 网络配置

**容器内访问：**

```bash
# 如果服务运行在容器内，使用容器名称
curl http://qq-music-api:3200/getHotkey
```

**外网访问：**

```bash
# 如果从宿主机或其他容器访问，使用宿主机 IP 或 localhost
curl http://localhost:3200/getHotkey
```

**端口映射：**

```bash
# 修改端口映射为 8080
docker run -d --name qq-music-api -p 8080:3200 qq-music-api

# 访问时使用 8080
curl http://localhost:8080/getHotkey
```

### 错误处理

**常见错误及处理：**

| 错误                 | 原因                       | 解决方法               |
| -------------------- | -------------------------- | ---------------------- |
| `no id~~`            | getImageUrl 缺少 id 参数   | 检查参数传递           |
| `no albummid`        | getAlbumInfo 缺少 albummid | 检查搜索结果           |
| `search key is null` | 搜索关键字为空             | 检查 FLAC 文件标签     |
| 超时                 | 网络问题或 API 响应慢      | 增加 timeout 或重试    |
| `未找到专辑`         | 搜索无结果                 | 检查歌手名和专辑名拼写 |

**错误处理示例：**

```python
try:
    album_info = search_album(artist, album)
except Exception as e:
    print(f"搜索失败：{e}")
    print(f"尝试简化搜索：只使用专辑名...")
    album_info = search_album("", album)  # 只搜索专辑名
```

### 资源清理

**清理无用的 Docker 资源：**

```bash
# 清理停止的容器
docker container prune

# 清理未使用的镜像
docker image prune

# 清理所有未使用的资源
docker system prune -a
```

**监控容器资源：**

```bash
# 实时查看资源使用
docker stats qq-music-api

# 查看容器日志
docker logs -f qq-music-api
```

---

## 相关文件位置

| 功能         | 文件路径                                 |
| ------------ | ---------------------------------------- |
| Docker 配置  | `Dockerfile:1-19`                        |
| 构建脚本     | `scripts/build-images.js:9-12`           |
| 搜索 API     | `routers/context/getSearchByKey.js:7-19` |
| 专辑信息 API | `routers/context/getAlbumInfo.js:5-11`   |
| 图片 URL API | `routers/context/getImageUrl.js:15-21`   |
| 歌手专辑 API | `routers/context/getSingerAlbum.js:5-22` |
| 用户信息配置 | `config/user-info.js:8-10`               |
| 路由定义     | `routers/router.js:14,83,53,106`         |

---

## 技术支持

- **项目地址**: [https://github.com/Rain120/qq-music-api](https://github.com/Rain120/qq-music-api)
- **问题反馈**: [https://github.com/Rain120/qq-music-api/issues](https://github.com/Rain120/qq-music-api/issues)
- **在线文档**: [https://rain120.github.io/qq-music-api](https://rain120.github.io/qq-music-api)

---

**文档版本:** 1.0
**最后更新:** 2026-01-28
