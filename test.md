import numpy as np
import pandas as pd

# 创建数组
arr = np.array([[1, 2, 3], [4, 5, 6]])

# 将每行转换为列表
data = [row.tolist() for row in arr]

# 创建 DataFrame
df = pd.DataFrame(data, columns=['Column1'])

print(df)
