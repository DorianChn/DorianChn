# 地图查询工具 (maps)

## 这个skill能做什么
**免费查询地理位置信息** —— 搜索地名、查坐标、找附近餐厅/医院、算驾车距离、查时区，全用Python自带库，不需要API密钥。

## 使用场景
- 想去吃饭，搜附近有什么餐厅
- 拿到一个地址想知道坐标
- 从A地到B地开车/走路多远
- 旅游时查当地时区
- 做数据分析需要地理编码

## 前置要求
```bash
# 只需要Python 3.8+，不需要装任何第三方库！
python --version
```

## 快速开始

### 1. 保存脚本
把下面的Python代码保存为 `maps.py`

### 2. 运行示例
```bash
# 搜索地名
python maps.py search "北京天安门"

# 坐标转地址
python maps.py reverse 39.9042 116.3974

# 找附近餐厅
python maps.py nearby 39.9042 116.3974 restaurant

# 算距离
python maps.py distance "北京" --to "上海"

# 查时区
python maps.py timezone 39.9042 116.3974
```

## 完整代码

```python
#!/usr/bin/env python3
"""
maps.py - 免费地图查询工具
使用 OpenStreetMap 数据，无需 API 密钥
功能：地名搜索、反向地理编码、附近POI、距离计算、时区查询
"""

import json, sys, time, urllib.request, urllib.parse, math

# ============================================================
# 配置
# ============================================================
USER_AGENT = "MapsTool/1.0 (learning-python)"
# API 地址
NOMINATIM   = "https://nominatim.openstreetmap.org"
OVERPASS    = "https://overpass-api.de/api/interpreter"
OSRM        = "https://router.project-osrm.org/route/v1"
TIMEAPI     = "https://timeapi.io/api/timezone/coordinate"

# POI 分类（OSM 标签映射）
CATEGORIES = {
    "restaurant":  ("amenity", "restaurant"),   # 餐厅
    "cafe":        ("amenity", "cafe"),          # 咖啡馆
    "hospital":    ("amenity", "hospital"),      # 医院
    "pharmacy":    ("amenity", "pharmacy"),      # 药店
    "hotel":       ("tourism", "hotel"),         # 酒店
    "supermarket": ("shop", "supermarket"),      # 超市
    "gas_station": ("amenity", "fuel"),          # 加油站
    "parking":     ("amenity", "parking"),       # 停车场
    "bank":        ("amenity", "bank"),          # 银行
    "atm":         ("amenity", "atm"),           # ATM
    "school":      ("amenity", "school"),        # 学校
    "park":        ("leisure", "park"),          # 公园
    "museum":      ("tourism", "museum"),        # 博物馆
    "cinema":      ("amenity", "cinema"),        # 电影院
    "gym":         ("leisure", "fitness_centre"),# 健身房
    "library":     ("amenity", "library"),       # 图书馆
}

# ============================================================
# 网络请求
# ============================================================
def http_get(url, params=None):
    """发送 HTTP GET 请求，返回解析后的 JSON 数据"""
    if params:
        url += "?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
    try:
        with urllib.request.urlopen(req, timeout=10) as resp:
            return json.loads(resp.read().decode("utf-8"))
    except Exception as e:
        return {"error": str(e)}

def http_post(url, data):
    """发送 HTTP POST 请求（用于 Overpass API）"""
    body = urllib.parse.urlencode({"data": data}).encode("utf-8")
    req = urllib.request.Request(url, data=body,
        headers={"User-Agent": USER_AGENT, "Content-Type": "application/x-www-form-urlencoded"})
    try:
        with urllib.request.urlopen(req, timeout=20) as resp:
            return json.loads(resp.read().decode("utf-8"))
    except Exception as e:
        return {"error": str(e)}

# ============================================================
# 工具函数
# ============================================================
def haversine(lat1, lon1, lat2, lon2):
    """计算两点间直线距离（米），使用 Haversine 公式"""
    R = 6371000  # 地球半径（米）
    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)
    a = (math.sin(dlat/2)**2 +
         math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) *
         math.sin(dlon/2)**2)
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

def fmt_dist(meters):
    """格式化距离显示"""
    if meters >= 1000:
        return f"{meters/1000:.1f} km"
    return f"{meters:.0f} m"

def fmt_time(seconds):
    """格式化时间显示"""
    h, r = divmod(int(seconds), 3600)
    m, s = divmod(r, 60)
    if h > 0:  return f"{h}小时{m}分钟"
    if m > 0:  return f"{m}分钟{s}秒"
    return f"{s}秒"

# ============================================================
# 核心功能
# ============================================================

# ----- 1. 搜索地名（地理编码）-----
def search(query):
    """搜索地名，返回坐标和详细信息"""
    params = {"q": query, "format": "json", "limit": 5, "accept-language": "zh"}
    time.sleep(1)  # Nominatim 限流要求
    data = http_get(f"{NOMINATIM}/search", params)
    if "error" in data:
        return {"error": data["error"]}
    results = []
    for item in data:
        results.append({
            "name": item.get("display_name", ""),
            "lat": float(item["lat"]),
            "lon": float(item["lon"]),
            "type": item.get("type", ""),
            "category": item.get("category", ""),
        })
    return {"results": results, "count": len(results)}

# ----- 2. 坐标转地址（反向地理编码）-----
def reverse(lat, lon):
    """将经纬度坐标转换为人类可读的地址"""
    params = {"lat": lat, "lon": lon, "format": "json", "accept-language": "zh"}
    time.sleep(1)
    data = http_get(f"{NOMINATIM}/reverse", params)
    if "error" in data:
        return {"error": data["error"]}
    addr = data.get("address", {})
    return {
        "address": data.get("display_name", ""),
        "lat": float(data["lat"]),
        "lon": float(data["lon"]),
        "details": {
            "country": addr.get("country", ""),
            "city": addr.get("city") or addr.get("town") or addr.get("village", ""),
            "road": addr.get("road", ""),
            "postcode": addr.get("postcode", ""),
        }
    }

# ----- 3. 找附近POI -----
def nearby(lat, lon, category, radius=500, limit=10):
    """查找附近的兴趣点（餐厅、医院等）"""
    if category not in CATEGORIES:
        cats = ", ".join(CATEGORIES.keys())
        return {"error": f"未知分类 '{category}'，可选：{cats}"}
    tag_key, tag_val = CATEGORIES[category]
    # Overpass QL 查询语句
    overpass_query = f"""
        [out:json][timeout:15];
        node[{tag_key}={tag_val}](around:{radius},{lat},{lon});
        out center {limit};
    """
    data = http_post(OVERPASS, overpass_query)
    if "error" in data:
        return {"error": data["error"]}
    places = []
    for elem in data.get("elements", []):
        p_lat = elem.get("lat") or elem.get("center", {}).get("lat", lat)
        p_lon = elem.get("lon") or elem.get("center", {}).get("lon", lon)
        tags = elem.get("tags", {})
        dist = haversine(lat, lon, p_lat, p_lon)
        places.append({
            "name": tags.get("name", "（无名）"),
            "lat": p_lat, "lon": p_lon,
            "distance": fmt_dist(dist),
            "distance_m": round(dist, 1),
            "address": tags.get("addr:full", ""),
            "phone": tags.get("phone", ""),
        })
    # 按距离排序
    places.sort(key=lambda x: x["distance_m"])
    return {"category": category, "count": len(places), "results": places[:limit]}

# ----- 4. 计算驾车距离和时间 -----
def calc_distance(origin, destination, mode="driving"):
    """计算两地之间的驾车/步行/骑行距离和时间"""
    # 先查坐标
    o = search(origin)
    d = search(destination)
    if "error" in o or "error" in d:
        return {"error": "无法找到地点"}
    if o["count"] == 0 or d["count"] == 0:
        return {"error": "无法找到地点"}
    o_loc = o["results"][0]
    d_loc = d["results"][0]
    # OSRM 路由查询
    url = f"{OSRM}/{mode}/{o_loc['lat']},{o_loc['lon']};{d_loc['lat']},{d_loc['lon']}"
    data = http_get(url, {"overview": "false"})
    if "error" in data:
        return {"error": data["error"]}
    routes = data.get("routes", [])
    if not routes:
        return {"error": "未找到路线"}
    route = routes[0]
    dist_m = route.get("distance", 0)
    dur_s = route.get("duration", 0)
    return {
        "origin": o_loc["name"],
        "destination": d_loc["name"],
        "mode": mode,
        "distance": fmt_dist(dist_m),
        "distance_m": round(dist_m),
        "duration": fmt_time(dur_s),
        "duration_s": round(dur_s),
    }

# ----- 5. 查时区 -----
def get_timezone(lat, lon):
    """根据坐标查询时区"""
    if not (-90 <= lat <= 90) or not (-180 <= lon <= 180):
        return {"error": "坐标超出范围"}
    params = {"latitude": lat, "longitude": lon}
    data = http_get(TIMEAPI, params)
    if "error" in data:
        # 备用方案：用经度估算时区
        offset = round(lon / 15)
        sign = "+" if offset >= 0 else "-"
        return {"timezone": f"UTC{sign}{abs(offset):02d}:00",
                "source": "经度估算（longitude/15）"}
    return {
        "timezone": data.get("timeZone", "未知"),
        "utc_offset": data.get("currentUtcOffset", {}).get("hours", 0),
        "current_time": data.get("currentLocalTime", ""),
        "source": "timeapi.io",
    }

# ============================================================
# 命令行入口
# ============================================================
def main():
    if len(sys.argv) < 2:
        print("用法：python maps.py <命令> [参数...]")
        print("命令：search, reverse, nearby, distance, timezone")
        print("示例：")
        print("  python maps.py search \"北京天安门\"")
        print("  python maps.py reverse 39.9042 116.3974")
        print("  python maps.py nearby 39.9042 116.3974 restaurant")
        print("  python maps.py distance \"北京\" --to \"上海\"")
        print("  python maps.py timezone 39.9042 116.3974")
        return

    cmd = sys.argv[1]

    if cmd == "search":
        # python maps.py search "地名"
        query = " ".join(sys.argv[2:])
        if not query:
            print("请提供地名，例如：python maps.py search \"北京天安门\"")
            return
        result = search(query)
        print(json.dumps(result, ensure_ascii=False, indent=2))

    elif cmd == "reverse":
        # python maps.py reverse 纬度 经度
        if len(sys.argv) < 4:
            print("请提供纬度和经度")
            return
        lat, lon = float(sys.argv[2]), float(sys.argv[3])
        result = reverse(lat, lon)
        print(json.dumps(result, ensure_ascii=False, indent=2))

    elif cmd == "nearby":
        # python maps.py nearby 纬度 经度 分类
        if len(sys.argv) < 5:
            print("请提供纬度、经度和分类")
            return
        lat, lon = float(sys.argv[2]), float(sys.argv[3])
        category = sys.argv[4]
        result = nearby(lat, lon, category)
        print(json.dumps(result, ensure_ascii=False, indent=2))

    elif cmd == "distance":
        # python maps.py distance "起点" --to "终点" [--mode driving]
        if "--to" not in sys.argv:
            print("请使用 --to 指定目的地")
            return
        idx = sys.argv.index("--to")
        origin = " ".join(sys.argv[2:idx])
        dest = " ".join(sys.argv[idx+1:idx+2]) if idx+2 < len(sys.argv) else ""
        mode = "driving"
        if "--mode" in sys.argv:
            mode = sys.argv[sys.argv.index("--mode") + 1]
        result = calc_distance(origin, dest, mode)
        print(json.dumps(result, ensure_ascii=False, indent=2))

    elif cmd == "timezone":
        # python maps.py timezone 纬度 经度
        if len(sys.argv) < 4:
            print("请提供纬度和经度")
            return
        lat, lon = float(sys.argv[2]), float(sys.argv[3])
        result = get_timezone(lat, lon)
        print(json.dumps(result, ensure_ascii=False, indent=2))

    else:
        print(f"未知命令：{cmd}")
        print("可用命令：search, reverse, nearby, distance, timezone")

if __name__ == "__main__":
    main()
```

## 常见问题

**Q: 报错 `Connection timeout`？**
A: 国内访问 OpenStreetMap 可能较慢，多试几次。或者检查网络连接。

**Q: 搜索英文地名没有结果？**
A: 可以试试英文搜索，如 `python maps.py search "Times Square"`

**Q: 附近搜索返回空结果？**
A: 把半径调大，或者换一个分类。比如 `restaurant` 可能比 `museum` 多。

**Q: 需要 API Key 吗？**
A: 不需要！所有数据来自 OpenStreetMap 等免费公开 API。

**Q: Nominatim 限流怎么办？**
A: 代码已内置 1 秒延时，连续搜索不要太快。

## 进阶用法
- 修改 `CATEGORIES` 字典添加更多 POI 分类
- 结合 `pandas` 批量处理地址列表
- 用 `matplotlib` 在地图上标注查询结果
- 添加 `--radius` 参数控制搜索半径

## 参考资源
- [OpenStreetMap 官网](https://www.openstreetmap.org/)
- [Nominatim 文档](https://nominatim.org/release-docs/develop/api/Overview/)
- [Overpass API 指南](https://wiki.openstreetmap.org/wiki/Overpass_API)
- [OSRM 路由服务](https://project-osrm.org/)
- [TimeAPI 时区服务](https://timeapi.io/)