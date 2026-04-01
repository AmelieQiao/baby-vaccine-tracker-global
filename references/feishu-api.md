# Feishu Bitable API Integration

This file tells the AI how to read from and write to the user's Feishu multi-dimensional table
(多维表格) as the persistent data source for the vaccine tracker.

---

## Setup Variables (User Must Provide Once)

Ask the user for these during onboarding if not already set:

```
FEISHU_APP_ID       = "cli_xxxxxxxxxxxx"       (from 飞书开放平台 → 凭证与基础信息)
FEISHU_APP_SECRET   = "xxxxxxxxxxxxxxxxxxxxxxxx" (same page)
FEISHU_APP_TOKEN    = "YcAWbqnmGaAcwPsuLdHcZ1hUnxe"  (from table URL: /base/{this})
FEISHU_TABLE_ID     = "tblXXXXXXXXXXXXXX"      (from table URL: ?table={this})
```

---

## Step 1: Get Access Token (every session)

Feishu tokens expire. Always get a fresh one at the start of any API operation.

```python
import requests

def get_feishu_token(app_id, app_secret):
    url = "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal"
    resp = requests.post(url, json={"app_id": app_id, "app_secret": app_secret})
    return resp.json()["tenant_access_token"]
```

---

## Step 2: Read All Records (用于从表格读取已有记录)

```python
def read_all_records(token, app_token, table_id):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records"
    headers = {"Authorization": f"Bearer {token}"}
    params = {"page_size": 500}
    
    all_records = []
    has_more = True
    page_token = None
    
    while has_more:
        if page_token:
            params["page_token"] = page_token
        resp = requests.get(url, headers=headers, params=params).json()
        items = resp.get("data", {}).get("items", [])
        all_records.extend(items)
        has_more = resp.get("data", {}).get("has_more", False)
        page_token = resp.get("data", {}).get("page_token")
    
    return all_records
```

Each record looks like:
```json
{
  "record_id": "recXXXXXX",
  "fields": {
    "疫苗名称": {"text": "乙肝疫苗 (HepB)"},
    "孩子": {"text": "小明"},
    "剂次": 1,
    "接种日期": 1678886400000,
    "品牌/厂家": {"text": "国药"},
    "状态": {"text": "已接种"},
    "计划接种日期": null
  }
}
```

Note: Dates in Feishu are Unix timestamps in milliseconds. Convert:
```python
from datetime import datetime
date_ms = 1678886400000
date_str = datetime.fromtimestamp(date_ms / 1000).strftime("%Y-%m-%d")
```

---

## Step 3: Write New Records (从疫苗本照片提取后写入)

```python
def create_record(token, app_token, table_id, fields_dict):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {"fields": fields_dict}
    resp = requests.post(url, headers=headers, json=payload)
    return resp.json()

# Example: writing a completed vaccine record
fields = {
    "疫苗名称": "乙肝疫苗 (HepB)",
    "孩子": "小明",
    "剂次": 1,
    "接种日期": 1678886400000,   # convert YYYY-MM-DD to ms timestamp
    "品牌/厂家": "国药",
    "状态": "已接种",
    "是否免费": True
}
create_record(token, APP_TOKEN, TABLE_ID, fields)
```

### Batch Write (more efficient for onboarding — writing many records at once)

```python
def batch_create_records(token, app_token, table_id, records_list):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/batch_create"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    # Max 500 per batch
    payload = {"records": [{"fields": r} for r in records_list]}
    resp = requests.post(url, headers=headers, json=payload)
    return resp.json()
```

---

## Step 4: Update a Record (更新单条记录，如状态从"待预约"→"已接种")

```python
def update_record(token, app_token, table_id, record_id, updated_fields):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/records/{record_id}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {"fields": updated_fields}
    resp = requests.patch(url, headers=headers, json=payload)
    return resp.json()
```

---

## Step 5: Create Views (视图) — Run Once During Onboarding

**Single child: group by vaccine name, sort by date within group, highlight TODO**

```python
def create_view_single_child(token, app_token, table_id):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/views"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {
        "view_name": "📋 疫苗总览（单孩）",
        "view_type": "grid",
        "property": {
            "group_by": [{"field_name": "疫苗名称", "desc": False}],
            "sort": [{"field_name": "接种日期", "desc": False}],
            "filter": {
                "conjunction": "and",
                "conditions": [
                    {"field_name": "状态", "operator": "isNot", "value": ["不适用"]}
                ]
            }
        }
    }
    return requests.post(url, headers=headers, json=payload).json()
```

**Multi-child: group by vaccine, sub-group by child, sort by date**

Note: Feishu API currently supports 1 group level in views via API. For multi-level grouping,
instruct the user to set it manually:
1. 打开视图 → 点顶部工具栏「分组」
2. 先按「疫苗名称」分组 → 再点「+添加分组」→ 按「孩子」分组
3. 再点「排序」→ 按「接种日期」升序

**Calendar view for scheduling:**

```python
def create_calendar_view(token, app_token, table_id):
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/views"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {
        "view_name": "📅 接种日历",
        "view_type": "calendar",
        "property": {
            "hierarchy_config": {
                "field_name": "计划接种日期"
            }
        }
    }
    return requests.post(url, headers=headers, json=payload).json()
```

---

## Date Conversion Utilities

```python
from datetime import datetime

def date_to_feishu_ms(date_str: str) -> int:
    """Convert '2024-03-15' to Feishu timestamp in milliseconds"""
    dt = datetime.strptime(date_str, "%Y-%m-%d")
    return int(dt.timestamp() * 1000)

def feishu_ms_to_date(ms: int) -> str:
    """Convert Feishu timestamp to '2024-03-15'"""
    return datetime.fromtimestamp(ms / 1000).strftime("%Y-%m-%d")
```

---

## Add a field (新增列)

Create with `POST` `.../apps/{app_token}/tables/{table_id}/fields`:

```python
requests.post(
    f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json; charset=utf-8"},
    json={"field_name": "地区标注", "type": 1},  # 1 = 文本
).json()

# 用户私有备注（AI 只读：写入记录时不要带此字段）
requests.post(
    f"https://open.feishu.cn/open-apis/bitable/v1/apps/{app_token}/tables/{table_id}/fields",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json; charset=utf-8"},
    json={"field_name": "人工备注", "type": 1},
).json()
```

New fields append at the end of the table; **reorder in Feishu UI** (e.g. **`地区标注`** 在 **`宝宝`** 与 **`疫苗名称`** 之间；**`人工备注`** 可放在表末或你顺手的位置)。

---

## Field Name Mapping (for writing records)

Use these EXACT Chinese field names when writing to the table:

| Field key to use in API | Type | Notes |
|------------------------|------|-------|
| `宝宝` | SingleSelect / text | User table may use 宝宝1/宝宝2 or nicknames |
| `地区标注` | text | **🇨🇳🇺🇸🇩🇪 flags only**; keep `疫苗名称` emoji-free for grouping in views |
| `疫苗名称` | text | **Vaccine generic name only** — no 「第X剂」/加强/首季等，no flag emoji；dose only in `剂次` |
| `人工备注` | text | **User-owned. AI: read OK; never include in POST/PUT/batch `fields`.** |
| `孩子` | text | Legacy template field; prefer `宝宝` if that is the table column |
| `剂次` | number | Integer: 1, 2, 3, 4 |
| `接种日期` | number | Unix ms timestamp |
| `品牌/厂家` | text | |
| `批号` | text | 规划行可用 **`todo`** 占位；**接种后**改为真实批号或清空，勿长期留 `todo`（便于视图筛选） |
| `接种机构` | text | |
| `状态` | text | Must be: 已接种 / 待预约 / 过期未打 / 不适用 |
| `计划接种日期` | number | Unix ms timestamp, or null |
| `备注` | text | |
| `是否免费` | boolean | true/false |
| `参考标准` | text | 国内标准 / 国际标准 / 两者 |

---

## When to Read vs Write

| Situation | Action |
|-----------|--------|
| User uploads vaccine photo but also shares Feishu link | Read table first → merge with photo data → resolve conflicts by asking user |
| User uploads photo only | Extract from photo → write ALL to Feishu (onboarding batch) |
| User fills table themselves → shares link | Read table → treat as ground truth → skip photo extraction |
| User says "we just got a shot" | Read current record for that vaccine → update 状态 to 已接种 + set 接种日期 |
| Generating a schedule | Read table → identify all 待预约 + 过期未打 rows → compute plan → write 计划接种日期 back |

---

## Error Handling

Common errors and what to do:

| Error code | Meaning | Action |
|------------|---------|--------|
| 99991671 | Token expired | Re-fetch token |
| 1254043 | No permission to table | Ask user to re-add the app to the table (Step 2 of setup) |
| 1254045 | Field name doesn't match | Check exact Chinese field name spelling |
| 400 | Bad field type | Check date fields are ms timestamps, not strings |
