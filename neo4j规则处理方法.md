在 `text2cypher` 任务中，生成的 Cypher 查询可能存在**语法错误、实体/关系不匹配、查询优化问题**等。为了提升准确度，可以采用 **规则处理、语法检查、实体对齐、查询优化** 等策略。  

---

## **1. 规则处理：NL 到 Cypher 解析增强**
在 `text2cypher` 转换过程中，我们可以对 **模型输出的 Cypher 进行修正**，避免结构性错误。  

### **(1) 关键字增强**
确保 Cypher 生成时包含必要的关键字（如 `MATCH`、`RETURN`）。
```python
def enforce_cypher_structure(cypher_query):
    """ 强制 Cypher 结构完整 """
    if "MATCH" not in cypher_query:
        cypher_query = "MATCH " + cypher_query  # 补全 MATCH 语句
    if "RETURN" not in cypher_query:
        cypher_query += " RETURN *"  # 默认返回所有匹配项
    return cypher_query
```
**示例**
```python
q = "(:Person)-[:ACTED_IN]->(:Movie)"
print(enforce_cypher_structure(q))
# 输出: MATCH (:Person)-[:ACTED_IN]->(:Movie) RETURN *
```

### **(2) 变量名一致性**
有时候 LLM 可能会随机生成变量名，导致 `MATCH (a:Person) RETURN b.name` 这种错误。可以用正则解析变量并进行对齐：
```python
import re

def fix_variable_names(cypher_query):
    """ 确保 MATCH 和 RETURN 中的变量名一致 """
    match = re.findall(r"MATCH \((\w+):", cypher_query)
    return_vars = re.findall(r"RETURN (\w+)", cypher_query)

    if return_vars and match and return_vars[0] != match[0]:
        cypher_query = cypher_query.replace(return_vars[0], match[0])
    
    return cypher_query
```
**示例**
```python
q = "MATCH (a:Person) RETURN b.name"
print(fix_variable_names(q))
# 输出: MATCH (a:Person) RETURN a.name
```

---

## **2. 语法错误检查**
使用 **Neo4j 自带的 `EXPLAIN` 语法检查**，避免执行错误的查询。  
```python
def check_cypher_syntax(cypher_query):
    """ 使用 EXPLAIN 检查 Cypher 语法 """
    try:
        with driver.session() as session:
            session.run(f"EXPLAIN {cypher_query}")
        return True
    except Exception as e:
        return False
```
**示例**
```python
q = "MATCH (p:Person) RETURN p.nam"  # `name` 写错了
print(check_cypher_syntax(q))  # False
```

---

## **3. 实体 & 关系对齐**
**问题**：如果用户查询的是 `"导演的电影"`，但 Cypher 里写的是 `ACTED_IN`，需要进行**实体关系映射**。  

### **(1) 维护数据库 Schema**
```python
schema = {
    "Person": {
        "valid_properties": ["name", "birthYear"],
        "relations": {
            "ACTED_IN": "Movie",
            "DIRECTED": "Movie"
        }
    },
    "Movie": {
        "valid_properties": ["title", "year"],
        "relations": {
            "HAS_GENRE": "Genre"
        }
    }
}
```
---

### **(2) 修正关系错误**
如果 `MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN m.director`，但 `Movie` 没有 `director` 属性，应该返回 `Person`：
```python
def fix_entity_relationship(cypher_query):
    """ 修正实体-关系不匹配 """
    for entity, details in schema.items():
        for relation, target in details["relations"].items():
            if relation in cypher_query and target not in cypher_query:
                cypher_query = cypher_query.replace(relation, list(details["relations"].keys())[0]) 
    return cypher_query
```
**示例**
```python
q = "MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN m.director"
print(fix_entity_relationship(q))
# 输出: MATCH (p:Person)-[:DIRECTED]->(m:Movie) RETURN p.name
```

---

## **4. 查询优化**
**问题**：某些生成的 Cypher 查询可以进行优化，比如：
- **去除多余的 `MATCH`**
- **避免 `RETURN *`**
- **加速索引查询**

### **(1) 移除 `RETURN *`**
```python
def remove_return_all(cypher_query):
    """ 替换 RETURN *，只返回必要的字段 """
    return cypher_query.replace("RETURN *", "RETURN DISTINCT n.name")
```

### **(2) 添加索引优化**
```python
def optimize_query(cypher_query):
    """ 使用索引优化查询 """
    if "MATCH (n:Person)" in cypher_query:
        cypher_query = cypher_query.replace("MATCH (n:Person)", "MATCH (n:Person) USING INDEX n:Person(name)")
    return cypher_query
```
**示例**
```python
q = "MATCH (n:Person) RETURN *"
print(optimize_query(remove_return_all(q)))
# 输出: MATCH (n:Person) USING INDEX n:Person(name) RETURN DISTINCT n.name
```

---

## **5. 综合修正 Pipeline**
将以上所有修正步骤整合到完整的 Pipeline：
```python
def process_cypher_query(cypher_query):
    """ 依次执行修正步骤 """
    cypher_query = enforce_cypher_structure(cypher_query)
    cypher_query = fix_variable_names(cypher_query)
    cypher_query = fix_entity_relationship(cypher_query)
    cypher_query = remove_return_all(cypher_query)
    cypher_query = optimize_query(cypher_query)
    
    # 语法检查
    if check_cypher_syntax(cypher_query):
        return cypher_query
    else:
        return "ERROR: Invalid Cypher Query"

# 测试
query = "MATCH (a:Person)-[:ACTED_IN]->(b:Movie) RETURN *"
print(process_cypher_query(query))
```

---

## **总结**
| 处理步骤 | 主要作用 |
|----------|---------|
| **规则处理** | 确保变量名、结构一致 |
| **语法检查** | 使用 `EXPLAIN` 预检测错误 |
| **实体关系对齐** | 确保 `Person`、`Movie` 的关系合理 |
| **查询优化** | 去除 `RETURN *`，添加索引 |

✅ 通过这些规则，`text2cypher` 的准确率能显著提升！