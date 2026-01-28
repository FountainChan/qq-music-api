# QQ Music API 调用指南 - 获取专辑封面与元数据

本文档提供完整的 API 调用说明，用于从 QQ Music 获取专辑封面和元数据信息。

## 目录

- [实现流程](#实现流程)
- [通过搜索获取 ID](#通过搜索获取-id)
- [获取专辑详细信息](#获取专辑详细信息)
- [获取专辑封面](#获取专辑封面)
- [获取歌手专辑列表](#获取歌手专辑列表)
- [完整实现方案](#完整实现方案)
- [注意事项](#注意事项)

---

## 实现流程

```
FLAC 文件 → 提取歌手名/专辑名 → 搜索 API → 获取 ID
  ↓
获取专辑信息 → 下载封面 → 嵌入 FLAC 文件
  ↓
写入发行年份
```

---

## 通过搜索获取 ID

### API 端点

```
GET /getSearchByKey
```

### 路由定义

`routers/router.js:14`

### 参数说明

| 参数          | 类型   | 必填 | 说明                         | 默认值 | 示例     |
| ------------- | ------ | ---- | ---------------------------- | ------ | -------- |
| `key`         | string | 是   | 搜索关键字（歌手名或专辑名） | -      | "叶惠美" |
| `page`        | number | 否   | 页码                         | 1      | 1        |
| `limit`       | number | 否   | 每页数量                     | 10     | 10       |
| `remoteplace` | string | 否   | 搜索类型                     | song   | album    |
| `catZhida`    | number | 否   | 保留参数                     | 1      | 1        |

### remoteplace 选项

| 值         | 说明    | 适用场景               |
| ---------- | ------- | ---------------------- |
| `song`     | 单曲    | 搜索歌曲               |
| `album`    | 专辑 ⭐ | **获取专辑封面时使用** |
| `mv`       | MV      | 搜索 MV                |
| `playlist` | 歌单    | 搜索歌单               |
| `user`     | 用户    | 搜索用户               |
| `lyric`    | 歌词    | 搜索歌词               |

### 请求示例

```bash
# 搜索专辑（推荐）
curl "http://localhost:3200/getSearchByKey?key=叶惠美&remoteplace=album&page=1&limit=10"

# 搜索歌手
curl "http://localhost:3200/getSearchByKey?key=周杰伦&remoteplace=song&page=1&limit=10"

# 搜索歌手名和专辑名组合
curl "http://localhost:3200/getSearchByKey?key=周杰伦+叶惠美&remoteplace=album&page=1&limit=10"
```

### 响应数据

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

### 关键字段说明

| 字段          | 说明     | 用途                        |
| ------------- | -------- | --------------------------- |
| `albummid`    | 专辑 ID  | ⭐ **用于获取封面**         |
| `singermid`   | 歌手 ID  | ⭐ **用于获取歌手专辑列表** |
| `albumname`   | 专辑名称 | 显示和匹配                  |
| `singername`  | 歌手名称 | 显示和匹配                  |
| `time_public` | 发行时间 | 可选的年份信息              |

### Python 调用示例

```python
import requests

BASE_URL = "http://localhost:3200"

def search_album(artist, album_name):
    """搜索专辑获取 ID"""
    search_key = f"{artist} {album_name}".strip()

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

# 使用示例
result = search_album("周杰伦", "叶惠美")
print(f"专辑 ID: {result['albummid']}")
print(f"歌手 ID: {result['singermid']}")
```

---

## 获取专辑详细信息

### API 端点

```
GET /getAlbumInfo
```

### 路由定义

`routers/router.js:83`

### 参数说明

| 参数       | 类型   | 必填 | 说明    | 示例             |
| ---------- | ------ | ---- | ------- | ---------------- |
| `albummid` | string | 是   | 专辑 ID | "000MkMni19ClKG" |

### 请求示例

```bash
curl "http://localhost:3200/getAlbumInfo?albummid=000MkMni19ClKG"
```

### 响应数据

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

### 关键字段说明

| 字段         | 说明         | 格式转换                      |
| ------------ | ------------ | ----------------------------- |
| `pub_time`   | 发行时间戳   | ⭐ **转换为 YYYY-MM-DD 格式** |
| `albumname`  | 专辑名称     | 直接使用                      |
| `singername` | 歌手名称     | 直接使用                      |
| `genre`      | 音乐类型     | 直接使用                      |
| `language`   | 语言         | 直接使用                      |
| `songlist`   | 专辑歌曲列表 | 可选使用                      |

### 时间格式转换

```python
# pub_time: 20030731 -> 2003-07-31
pub_time = 20030731
pub_time_str = str(pub_time)
pub_date = f"{pub_time_str[:4]}-{pub_time_str[4:6]}-{pub_time_str[6:8]}"
pub_year = pub_time_str[:4]  # 提取年份: 2003
```

### Python 调用示例

```python
def get_album_info(albummid):
    """获取专辑详细信息"""
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

# 使用示例
info = get_album_info("000MkMni19ClKG")
print(f"专辑: {info['albumname']}")
print(f"发行年份: {info['pub_year']}")
print(f"类型: {info['genre']}")
```

---

## 获取专辑封面

### 方案 A：使用 getImageUrl 接口 ⭐ **推荐**

#### API 端点

```
GET /getImageUrl
```

#### 路由定义

`routers/router.js:106`

#### 参数说明

| 参数     | 类型   | 必填 | 说明              | 默认值  | 示例             |
| -------- | ------ | ---- | ----------------- | ------- | ---------------- |
| `id`     | string | 是   | 专辑 ID 或歌曲 ID | -       | "000MkMni19ClKG" |
| `size`   | string | 否   | 图片尺寸          | 300x300 | 500x500          |
| `maxAge` | number | 否   | 缓存时间（毫秒）  | 2592000 | 86400000         |

#### size 支持的尺寸

| 尺寸        | 说明       | 适用场景              |
| ----------- | ---------- | --------------------- |
| `300x300`   | 默认尺寸   | 小图预览              |
| `500x500`   | 推荐尺寸   | ⭐ **平衡质量和大小** |
| `800x800`   | 高清尺寸   | 大图显示              |
| `1000x1000` | 超高清尺寸 | 高质量展示            |

#### 请求示例

```bash
# 获取默认尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG"

# 获取 500x500 尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG&size=500x500"

# 获取 800x800 尺寸封面
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG&size=800x800"

# 自定义缓存时间
curl "http://localhost:3200/getImageUrl?id=000MkMni19ClKG&size=500x500&maxAge=86400000"
```

#### 响应数据

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

#### Python 调用示例

```python
def get_album_cover(albummid, size="500x500"):
    """获取专辑封面"""
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

# 使用示例
cover_data = get_album_cover("000MkMni19ClKG", size="500x500")

# 保存到文件
with open("cover.jpg", "wb") as f:
    f.write(cover_data)
```

---

### 方案 B：使用 QQ Music 图片 URL 格式

#### URL 格式

```
http://i.gtimg.cn/music/photo/mid_album_500/7/a/专辑ID.jpg
```

#### 示例

```
http://i.gtimg.cn/music/photo/mid_album_500/7/a/000MkMni19ClKG.jpg
```

#### Python 调用示例

```python
def get_album_cover_direct(albummid):
    """使用直接 URL 格式获取封面"""
    image_url = f"http://i.gtimg.cn/music/photo/mid_album_500/7/a/{albummid}.jpg"

    response = requests.get(image_url, timeout=10)
    response.raise_for_status()

    return response.content

# 使用示例
cover_data = get_album_cover_direct("000MkMni19ClKG")
```

#### 两种方案对比

| 方案             | 优点                       | 缺点                 | 推荐场景          |
| ---------------- | -------------------------- | -------------------- | ----------------- |
| getImageUrl 接口 | 支持自定义尺寸，带缓存参数 | 需要调用 API         | ⭐ **大多数场景** |
| 直接 URL 格式    | 简单直接                   | 尺寸固定，无缓存控制 | 快速测试          |

---

## 获取歌手专辑列表

### API 端点

```
GET /getSingerAlbum
```

### 路由定义

`routers/router.js:53`

### 参数说明

| 参数        | 类型   | 必填 | 说明     | 默认值 | 示例             |
| ----------- | ------ | ---- | -------- | ------ | ---------------- |
| `singermid` | string | 是   | 歌手 ID  | -      | "0025NhlN2yWrP4" |
| `limit`     | number | 否   | 每页数量 | 5      | 20               |
| `page`      | number | 否   | 页码     | 0      | 1                |

### 请求示例

```bash
# 获取歌手的前 20 张专辑
curl "http://localhost:3200/getSingerAlbum?singermid=0025NhlN2yWrP4&limit=20&page=0"

# 获取第二页
curl "http://localhost:3200/getSingerAlbum?singermid=0025NhlN2yWrP4&limit=20&page=1"
```

### 响应数据

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

### Python 调用示例

```python
def get_singer_albums(singermid, limit=20, page=0):
    """获取歌手专辑列表"""
    response = requests.get(
        f"{BASE_URL}/getSingerAlbum",
        params={
            "singermid": singermid,
            "limit": limit,
            "page": page
        },
        timeout=10
    )

    data = response.json()

    if "response" not in data or "data" not in data["response"]:
        raise Exception("获取歌手专辑失败")

    albums = data["response"]["data"]["list"]

    return [
        {
            "albumname": album.get("albumname"),
            "albummid": album.get("albummid"),
            "pub_time": album.get("pub_time"),
            "public_time": album.get("public_time")
        }
        for album in albums
    ]

# 使用示例
albums = get_singer_albums("0025NhlN2yWrP4", limit=20)
for album in albums:
    print(f"{album['albumname']} - {album['public_time']}")
```

---

## 完整实现方案

### Python 完整脚本

```python
import requests
import time
from mutagen.flac import FLAC, Picture
import os
import glob

# 配置
BASE_URL = "http://localhost:3200"

def search_album(artist, album_name):
    """搜索专辑获取 ID"""
    search_key = f"{artist} {album_name}".strip()

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

    if "response" not in data or "data" not in data["response"]:
        raise Exception("搜索失败：无返回数据")

    songs = data["response"]["data"]["song"]["list"]
    if not songs:
        raise Exception(f"未找到专辑：{search_key}")

    first_match = songs[0]
    return {
        "albummid": first_match.get("albummid"),
        "singermid": first_match.get("singermid"),
        "albumname": first_match.get("albumname"),
        "singername": first_match.get("singername")
    }


def get_album_info(albummid):
    """获取专辑详细信息"""
    response = requests.get(
        f"{BASE_URL}/getAlbumInfo",
        params={"albummid": albummid},
        timeout=10
    )

    data = response.json()

    if "response" not in data or "data" not in data["response"]:
        raise Exception("获取专辑信息失败")

    album_info = data["response"]["data"]["list"][0]

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
    """获取专辑封面"""
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

    image_response = requests.get(image_url, timeout=10)
    image_response.raise_for_status()

    return image_response.content


def embed_cover_to_flac(flac_path, cover_data, metadata=None):
    """将封面嵌入到 FLAC 文件"""
    audio = FLAC(flac_path)

    audio.clear_pictures()

    image = Picture()
    image.type = 3  # 3 表示封面
    image.mime = "image/jpeg"
    image.width = 500
    image.height = 500
    image.data = cover_data

    audio.add_picture(image)

    if metadata:
        for key, value in metadata.items():
            audio[key] = value

    audio.save()


def process_flac_file(flac_path):
    """处理 FLAC 文件：获取专辑封面并嵌入"""
    try:
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

        time.sleep(1)

    except Exception as e:
        print(f"❌ 处理失败：{e}")


if __name__ == "__main__":
    flac_files = glob.glob("*.flac")

    print(f"📁 找到 {len(flac_files)} 个 FLAC 文件\n")

    for flac_file in flac_files:
        process_flac_file(flac_file)
        print()

    print("🎉 所有文件处理完成！")
```

### 使用方法

1. 安装依赖：

```bash
pip install requests mutagen
```

2. 运行脚本：

```bash
python process_flac.py
```

### 输出示例

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

---

## 相关文件位置

| 功能         | 文件路径                            | 行号            |
| ------------ | ----------------------------------- | --------------- |
| 搜索 API     | `routers/context/getSearchByKey.js` | 7-19            |
| 专辑信息 API | `routers/context/getAlbumInfo.js`   | 5-11            |
| 图片 URL API | `routers/context/getImageUrl.js`    | 15-21           |
| 歌手专辑 API | `routers/context/getSingerAlbum.js` | 5-22            |
| 路由定义     | `routers/router.js`                 | 14, 83, 53, 106 |
| 用户信息配置 | `config/user-info.js`               | 8-10            |

---

## 快速参考

### API 端点速查

| 功能         | 端点              | 必需参数               | 可选参数     |
| ------------ | ----------------- | ---------------------- | ------------ |
| 搜索专辑     | `/getSearchByKey` | key, remoteplace=album | page, limit  |
| 获取专辑信息 | `/getAlbumInfo`   | albummid               | -            |
| 获取封面     | `/getImageUrl`    | id                     | size, maxAge |
| 获取歌手专辑 | `/getSingerAlbum` | singermid              | limit, page  |

### 完整流程速查

```python
# 1. 搜索专辑
album_info = search_album("周杰伦", "叶惠美")
albummid = album_info["albummid"]

# 2. 获取专辑信息
detailed_info = get_album_info(albummid)
pub_year = detailed_info["pub_year"]

# 3. 获取封面
cover_data = get_album_cover(albummid, size="500x500")

# 4. 嵌入 FLAC
embed_cover_to_flac("song.flac", cover_data, {"DATE": pub_year})
```

---

**文档版本:** 1.0
**最后更新:** 2026-01-28
