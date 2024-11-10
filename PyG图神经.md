# Data类

为了构建图，需要两个元素：节点和边。PyG 提供了`torch_geometric.data.Data` (下面简称`Data`) 用于构建图，包括 5 个属性：

- x：存储节点特征，大小为`[num_nodes, num_node_features]`，即行数为节点数，列数为特征维度
- edge_index：存储边信息，大小为`[2, num_edges]`
- y：存储样本标签，大小为`[num_nodes, *]`，*代表维度可以是任意的。对于整张图而言的训练，维度为`[1, *]`。也就是说，前者是节点分类，后者是图分类
- edge_attr：存储边的特征，大小是`[num_edges, num_edge_features]`
- pos：存储节点坐标

## 边信息

对于edge_index，通过COO格式进行表示。对于每一条边，edge_index中对应有一个两节点元祖，第一个值为源节点index，第二个值为目标节点索引。例如，一个`(0 -> 1), (1 -> 2), (2 -> 1), (2 -> 3)`的图，edge_index的表示为

```python
edge_index = torch.tensor([[0, 1, 2, 2],
                           [1, 2, 1, 3]], dtype=torch.long)
```

因此可以得知，PyG中会将无向图视作有向图的特例。



# Dataset

可以继承`torch_geometric.data.Dataset`使用自己的数据集。有以下两种`Dataset`：

- InMemoryDataset：一次性把数据全部加载到内存中。
- Dataset：每次加载一个数据到内存中，比较常用。

## 文件夹与文件

PyG需要初始化`Dataset`时传入数据存放的路径，并这个路径下再划分 2 个文件夹：

- raw_dir：存放原始数据的路径，一般是 csv、mat 等格式
- processed_dir：存放处理后的数据，一般是 pt 格式，重写`process()`方法实现。

## 类方法

- raw_file_names()：

返回数据集原始文件的文件名列表，需要能在`raw_dir`对应文件夹找到。

- processed_file_names()：

返回存储处理过的数据文件的文件名列表，需要能在`processed_dir`文件夹中找到

### process()：

调用读取函数，将数据包装成Data类，处理数据，保存处理好的数据到`processed_dir`下。数据集原始的格式可能是 csv 或者 mat，在`process()`函数里可以转化为 pt 格式的文件，这样在`get()`方法中就可以直接使用`torch.load()`函数读取 pt 格式文件。

官网文档的示例代码（新版本PyG）为
```python
def process(self):
        # Data类的list，每个Data即为封装的图
        data_list = [...]

        if self.pre_filter is not None:
            data_list = [data for data in data_list if self.pre_filter(data)]

        if self.pre_transform is not None:
            data_list = [self.pre_transform(data) for data in data_list]

        self.save(data_list, self.processed_paths[0])
```
其中，`pre_filter`和`pre_transform`是初始化dataset时传入的预处理方法。

当PyG版本>=2.4时，torch.save和torch.load被封装为Dataset.save和Dataset.load




# 其他

## 图可视化

通过`networkx`库来实现可视化，例子如下

需要注意的是，data成员属性edge_index需要表示为[2, edge_num]的COO张量格式

```python
from torch_geometric.utils import to_networkx
import networkx as nx
import matplotlib.pyplot as plt

G = to_networkx(data, to_undirected=True)
nx.draw(G)
plt.show()
```

nx.draw参数：

- node_color：颜色、RGB值或者列表（根据列表中的值区分不同节点颜色）
- with_labels：标注出每个节点的编号

## mini-batch

使用`Dataloader`可以将数据集分为多个mini-batch，示例代码
```python
from torch_geometric.data import DataLoader

loader = DataLoader(dataset, batch_size=32, shuffle=True)

for batch in loader:
    batch
    # Batch(batch=[1082], edge_index=[2, 4066], x=[1082, 21], y=[32])
```
`Batch`继承自`Data`，多了个列向量属性`batch`，将每个元素映射到mini-batch的相应图