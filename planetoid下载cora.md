Planetoid是一个用于图神经网络（GNN）任务的基准数据集库，其中包括经典的图结构数据集，如Cora、CiteSeer和PubMed。它们常被用于节点分类、链接预测等任务。Planetoid的`Cora`数据集包含关于科学出版物的节点和它们之间的引用关系。

### Cora数据的内容：
- **节点（Node）**：每个节点代表一篇论文。
- **边（Edges）**：每一条边表示两篇论文之间的引用关系。
- **节点特征（Node Features）**：每篇论文的文本内容被表示为一个1433维的稀疏向量（one-hot编码），其中每个维度对应一个词的出现与否。
- **类别标签（Labels）**：每篇论文被分配一个类别标签，表示该论文所属的主题类别，共有7个类别。

### 返回数据的格式：
1. **`edge_index`**：形状为`[2, num_edges]`的张量，表示图的边，第一行是起始节点，第二行是目标节点。
2. **`x`**：形状为`[num_nodes, num_features]`的稀疏矩阵，表示节点的特征（每个节点的1433维特征向量）。
3. **`y`**：形状为`[num_nodes]`的张量，表示每个节点的标签。
4. **`train_mask`**、**`val_mask`**、**`test_mask`**：布尔向量，指定哪些节点用于训练、验证或测试。

使用方式上，通常通过`torch_geometric.datasets.Planetoid`来加载Cora数据集，例如：

```python
from torch_geometric.datasets import Planetoid

# 加载 Cora 数据集
dataset = Planetoid(root='/tmp/Cora', name='Cora')

# 访问图数据
data = dataset[0]  # 图数据
print(data.edge_index)  # 边
print(data.x)  # 特征
print(data.y)  # 标签
print(data.train_mask, data.val_mask, data.test_mask)  # 掩码
```

这样得到的数据格式便是可供图神经网络处理的输入。