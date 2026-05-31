# OneForAll-
OneForAll-0.4.5在python高版本无法使用，会产生报错，我在使用的时候被折磨坏了，找了半天没找到，我不如就自己写一个
# OneForAll v0.4.5 Python 3.14 兼容性修复

## 概述

OneForAll v0.4.5 在 Python 3.14 环境下无法运行，主要原因是多个依赖库和源码中使用了已在新版 Python 中移除的模块和 API。

## 修复内容

### 1. 升级不兼容的依赖包

| 包           | 旧版本 | 新版本 | 原因                             |
| ------------ | ------ | ------ | -------------------------------- |
| `fire`       | 0.4.0  | 0.7.1  | 依赖已移除的 `pipes` 模块        |
| `SQLAlchemy` | 1.3.22 | 2.0.50 | 旧版 API 与 Python 3.14 不兼容   |
| `dnspython`  | 2.2.1  | 2.8.0  | `resolver.query()` 已废弃        |
| `exrex`      | 0.10.5 | 0.12.0 | 依赖已移除的 `sre_parse` 模块    |
| `packaging`  | (新增) | 26.2   | 替代已移除的 `distutils.version` |

### 2. 修复源码中的废弃模块引用

#### oneforall.py

- 移除无用的 `import sre_parse`（Python 3.13 已移除该模块）

#### common/utils.py

- `from distutils.version import LooseVersion` → `from packaging.version import Version`
- `LooseVersion(version) < LooseVersion('3.6')` → `Version(version) < Version('3.6')`
- `resolver.query(qname, qtype)` → `resolver.resolve(qname, qtype)`

#### brute.py

- `resolver.query(ns, 'A')` → `resolver.resolve(ns, 'A')`
- `resolver.query(domain, 'NS')` → `resolver.resolve(domain, 'NS')`

#### takeover.py

- `resolver.query(subdomain, 'CNAME')` → `resolver.resolve(subdomain, 'CNAME')`

#### modules/wildcard.py

- `resolver.query(subdomain, 'A')` → `resolver.resolve(subdomain, 'A')`
- `resolver.query(domain, 'A')` → `resolver.resolve(domain, 'A')`

#### modules/check/axfr.py

- `resolver.query(self.domain, "NS")` → `resolver.resolve(self.domain, "NS")`

### 3. 修复 SQLAlchemy 2.0 API 兼容性

#### common/records.py

**查询结果处理：**

```python
# 旧代码（SQLAlchemy 1.x）
row_gen = (Record(cursor.keys(), row) for row in cursor)

# 新代码（SQLAlchemy 2.0）
row_gen = (Record(list(row._mapping.keys()), list(row._mapping.values())) for row in cursor)
```

**批量查询参数：**

```python
# 旧代码
self._conn.execute(text(query), *multiparams)

# 新代码
if multiparams:
    self._conn.execute(text(query), multiparams[0])
else:
    self._conn.execute(text(query))
```

**DDL/DML 语句兼容：**

```python
# 新增 returns_rows 检查，避免非 SELECT 语句报错
if cursor.returns_rows:
    row_gen = (Record(...) for row in cursor)
    results = RecordCollection(row_gen)
else:
    results = RecordCollection(iter([]))
```

## 测试命令

```
python oneforall.py --target example.com run
```

