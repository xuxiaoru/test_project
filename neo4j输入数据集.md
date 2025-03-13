现有数据库转换的方法主要是从 **已有的 Neo4j 数据库**（或其他图数据库）中 **自动提取 Cypher 查询**，然后**反向生成自然语言查询**，构建 `text2cypher` 训练数据集。  

---

## **步骤 1：连接 Neo4j 并获取模式信息**  

首先，需要连接到 Neo4j 数据库，并获取图的基本结构，包括 **节点标签（labels）、关系类型（relationships）、属性（properties）**。  

**Python 代码（使用 `neo4j` Python 驱动）**：
```python
from neo4j import GraphDatabase

# 连接到 Neo4j
uri = "bolt://localhost:7687"
driver = GraphDatabase.driver(uri, auth=("neo4j", "password"))

def get_schema():
    schema = {"nodes": {}, "relationships": []}
    
    with driver.session() as session:
        # 获取所有节点标签
        result = session.run("CALL db.labels()")
        labels = [record["label"] for record in result]
        
        # 获取所有关系类型
        result = session.run("CALL db.relationshipTypes()")
        relationships = [record["relationshipType"] for record in result]
        
        # 获取每种节点的属性
        for label in labels:
            result = session.run(f"MATCH (n:{label}) RETURN keys(n) AS properties LIMIT 1")
            properties = result.single()["properties"] if result.peek() else []
            schema["nodes"][label] = properties

        # 获取所有关系及其起点和终点
        result = session.run("MATCH (a)-[r]->(b) RETURN DISTINCT labels(a), type(r), labels(b)")
        for record in result:
            schema["relationships"].append({
                "start": record["labels(a)"][0], 
                "type": record["type(r)"], 
                "end": record["labels(b)"][0]
            })

    return schema

# 获取并打印数据库的模式
schema_info = get_schema()
print(schema_info)
```
---
## **步骤 2：从数据库自动生成 Cypher 查询**  

知道了 **节点类型、关系、属性** 之后，我们可以基于这些信息生成 Cypher 查询。例如：
- 电影数据库（`Movie`、`Person` 之间有 `ACTED_IN` 关系）
- 车辆数据库（`Car`、`Brand` 之间有 `PRODUCED_BY` 关系）

### **示例：生成查询**
```python
def generate_cypher_queries(schema):
    queries = []
    
    for label, properties in schema["nodes"].items():
        if "name" in properties:
            queries.append({
                "input": f"查询所有{label}的名称",
                "output": f"MATCH (n:{label}) RETURN n.name"
            })
        if "year" in properties:
            queries.append({
                "input": f"查询{label}的所有年份",
                "output": f"MATCH (n:{label}) RETURN DISTINCT n.year"
            })

    for relation in schema["relationships"]:
        queries.append({
            "input": f"查询所有{relation['start']} 和 {relation['end']} 之间的 {relation['type']} 关系",
            "output": f"MATCH (a:{relation['start']})-[r:{relation['type']}]->(b:{relation['end']}) RETURN a, b"
        })

    return queries

# 生成 Cypher 查询
query_dataset = generate_cypher_queries(schema_info)
for q in query_dataset[:5]:
    print(q)
```

---
## **步骤 3：反向生成自然语言查询（NL）**  

上一步我们只是自动生成了 Cypher 查询，现在需要将它们转换为更自然的用户查询。  

### **方法 1：规则映射**（简单但灵活性较低）  
可以用固定模板替换：
```python
def convert_cypher_to_nl(cypher_query):
    # 解析 Cypher 查询，提取实体和关系
    if "MATCH (n:Movie) RETURN n.name" in cypher_query:
        return "有哪些电影？"
    elif "MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN p.name" in cypher_query:
        return "哪些演员出演了电影？"
    elif "MATCH (c:Car) WHERE c.brand = 'Tesla' RETURN c.model" in cypher_query:
        return "特斯拉有哪些车型？"
    return None

# 示例转换
for query in query_dataset:
    nl_query = convert_cypher_to_nl(query["output"])
    if nl_query:
        print(f"自然语言: {nl_query} \nCypher: {query['output']}\n")
```

### **方法 2：利用 LLM（如 `Qwen-14B`）进行 NL 生成**  
如果有大量 Cypher 查询，可以用 **Qwen-14B** 让模型生成 **自然语言** 查询：
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline

model_name = "Qwen/Qwen1.5-14B"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name, device_map="auto")
generator = pipeline("text-generation", model=model, tokenizer=tokenizer)

def generate_nl_query(cypher_query):
    prompt = f"请将以下 Cypher 查询转换为自然语言问题：\n{cypher_query}\n"
    output = generator(prompt, max_length=100, num_return_sequences=1)
    return output[0]['generated_text']

# 示例转换
for query in query_dataset[:5]:
    nl_query = generate_nl_query(query["output"])
    print(f"自然语言: {nl_query} \nCypher: {query['output']}\n")
```

---
## **步骤 4：存储并构建数据集**  
最终，我们要把 `input`（自然语言查询） 和 `output`（Cypher 语句） 存入 JSON 文件：
```python
import json

with open("text2cypher_dataset.json", "w", encoding="utf-8") as f:
    json.dump(query_dataset, f, ensure_ascii=False, indent=4)

print("数据集构建完成！")
```

---
## **最终数据格式示例（JSON）**
```json
[
    {
        "input": "有哪些电影？",
        "output": "MATCH (n:Movie) RETURN n.name"
    },
    {
        "input": "哪些演员出演了电影？",
        "output": "MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN p.name"
    },
    {
        "input": "特斯拉有哪些车型？",
        "output": "MATCH (c:Car) WHERE c.brand = 'Tesla' RETURN c.model"
    }
]
```

---
## **总结**
| 步骤 | 关键操作 |
|------|---------|
| **1. 解析数据库模式** | 通过 Neo4j 获取 **节点、关系、属性** |
| **2. 自动生成 Cypher** | 基于数据库结构生成常见查询 |
| **3. 反向生成 NL 查询** | 用 **规则/LLM** 生成自然语言 |
| **4. 存储数据集** | 以 JSON 格式保存用于训练 |

### **适用场景**
✅ **已有 Neo4j 数据库**，希望从真实数据构建 `text2cypher` 训练数据  
✅ **已有 Cypher 查询日志**，想自动匹配自然语言  
✅ **希望用 LLM 辅助生成多样性 NL 查询**  

你可以用这个数据集 **微调 Qwen** 或 **训练自定义的 `text2cypher` 模型**！