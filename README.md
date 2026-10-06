# comfyui-kaloscope

### *「我这双眼睛，能将黑暗看得一清二楚」*

  —— 宇智波佐助

![阿妈特拉斯](https://github.com/user-attachments/assets/a9d16b72-b577-4458-bc10-604eb82fefea)

> “*Kaloscope*”（万花筒）致敬万花筒写轮眼，象征忍术(画风)复刻能力

> 该插件支持comfyui插件，webui插件和单独启动三种方式运行

## 核心能力

支持 *LSNet* 和 *DINOv3* 模型，本工具聚焦以下场景：

1. **画风分类**：识别单幅作品的风格属性，完成风格相似的标签匹配

2. **画风聚类**：自动对多组作品按风格特征进行归类聚合，筛选出风格相似的作品群体，实现批量风格整理与分析，提升处理效率

3. **特征提取**：输出整张图片、patch tokens、空间特征图或中间层特征，方便连接其他分析节点

4. **关系制图**：把多张图片的相似关系画成关系网络、距离热图或聚类散点，也可以查看特征统计、近邻排行和 patch 能量分布

## 第一步：下载必要文件

前往 Hugging Face 或 ModelScope 仓库，下载模型对应的文件：

* `best_checkpoint.pth` / `best.pt` / `model.safetensors`（*模型权重文件*，支持 `.pt`、`.pth`、`.ckpt`、`.safetensors`）

* `class_mapping.csv`（*风格类别映射配置文件*，可选，用于把类别编号转换成画师或风格名称）

* `config.json`（用于识别模型架构，如果有的话请下载）

### v3-preview版本

huggingface仓库地址：https://huggingface.co/heathcliff01/Kaloscope3.0-preview

或者在modelscope下载：https://www.modelscope.cn/models/Heathcliff02/Kaloscope3.0-preview

### v2版本

huggingface仓库地址：https://huggingface.co/heathcliff01/Kaloscope2.0

或者在modelscope下载：https://www.modelscope.cn/models/Heathcliff02/Kaloscope-2.0/summary

### v1版本

huggingface仓库地址：https://huggingface.co/heathcliff01/Kaloscope/tree/main

或者在modelscope下载：https://www.modelscope.cn/models/Heathcliff02/Kaloscope/files

## 第二步：文件放置与环境配置

### 1. 创建目录结构

在 ComfyUI 的`models`目录下，新建名为`kaloscope`的文件夹（存放模型文件）；

进入`kaloscope`文件夹后，可随意创建一个子文件夹（如 “checkpoints”“kaloscope” 等，名称无强制要求，用于归类核心文件）。

目录结构示例：


```
ComfyUI/

└── models/

       └── kaloscope/

           └── 子文件夹名称/  # 例："kaloscope-v2”

               ├── best_checkpoint.pth # 或 best.pt / model.safetensors

               ├── class_mapping.csv # 可选

               └── config.json # 填写模型架构
```

### DINOv3 分类头输入归一化

使用冻结主干训练的分类头时，推理必须沿用其训练时的特征预处理。例如 Kaloscope 3.0 Preview 的 v1 画师分类头使用以下配置：

```json
{
  "model": "dinov3_vitb16",
  "checkpoint": "model.safetensors",
  "pooling": "cls_mean",
  "feature_source": "projector",
  "input_size": 512,
  "classifier_input_normalization": "l2_sqrt_dim"
}
```

`l2_sqrt_dim` 在分类线性层之前，用 FP32 对池化特征做 L2 归一化，再乘以输入维数的平方根；对于 CLS + mean pooling，该维数为 1536。
归一化只影响分类分支，原始主干特征和风格投影输出保持不变。此字段也可放在 `model` 对象或检查点的 `model_config` 中；省略时默认 `none`，兼容已有模型。不要给未经此预处理训练的分类头开启该选项。

需使用支持该字段的插件版本；旧版本会忽略该字段并产生错误的分类 logits。
模型包还应包含按分类输出顺序排列的 `class_mapping.csv`（列名 `class_id,class_name`）。上例假设完整权重包含 temporal 模型的 `log_temperature` 和 `bias`，插件因此采用短边缩放至 512 后中心裁剪的预处理。

### 2. 安装依赖

将模型权重、配置和可选类别映射放入子文件夹后，在插件目录使用 ComfyUI 的 Python 环境安装依赖(webui插件可以跳过这一步，会自动安装依赖)：

```
python -m pip install -r requirements.txt
```

## 第三步：启动 ComfyUI 并使用

1. 按常规方式启动 ComfyUI

2. 在画风分析工作流中，调用 **Kaloscope** 分类下的节点，即可使用画风分类、特征提取、画风聚类和关系制图


### 使用示例

### 1. 分类与画风比较

* **Kaloscope Model Loader**：选择模型文件夹，输出模型

* **Kaloscope Artist Inference**：接图片和模型，输出标签与 JSON。无分类头时标签为空，JSON 输出特征

* **Kaloscope Artist Similarity**：比较一张查询图片与多张参考图片，输出余弦相似度

* **Kaloscope Common Features**：接一组参考图片，输出该组平均特征 `[D]`

* **Kaloscope Feature Comparison**：接查询图片与最多三组平均特征，输出各组相似度和最接近的组

* **Kaloscope Clustering**：接最多三组 `[B,D]` 特征，支持 KMeans、DBSCAN、hierarchical 聚类，可输出 PCA/t-SNE 图

* **Kaloscope Image Connector**：将三路同尺寸图片组成一个批次，每路若为批次则取第一张。更多图片可以使用 ComfyUI 的图片批次组合节点

分组比较时，每组图片分别接 **Common Features**，再把平均特征接到 **Feature Comparison** 的 `group_1` / `group_2` / `group_3`

聚类时把 **Extract Features** 的输出接到 **Clustering**；每张图片都需要保留自己的特征，不能用组均值替代。KMeans/hierarchical 的 `n_clusters` 要按图片数量设置，DBSCAN 使用 `eps` / `min_samples` 控制

### 2. 提取特征给下游使用

**Kaloscope Extract Features** 接图片批次和模型，最后输出一个 CPU float32 TENSOR。在 `output_type` 中选择需要的特征

| output_type | 输出形状 | 用途 |
| --- | --- | --- |
| `default` | `[B,F]` | 按模型配置选择骨干或投影特征，适合先做画风比较 |
| `backbone` | `[B,D]` 或 `[B,2D]` | 按模型池化配置输出骨干特征 |
| `cls` | `[B,D]` | CLS token，全局特征 |
| `mean` | `[B,D]` | patch tokens 的均值 |
| `cls_mean` | `[B,2D]` | CLS 与 patch 均值拼接 |
| `projector` | `[B,P]` | 模型投影层输出 |
| `patch_tokens` | `[B,N,D]` | 局部 patch 特征 |
| `patch_map` | `[B,D,H,W]` | 空间特征图 |
| `storage_tokens` | `[B,R,D]` | storage/register tokens |
| `all_tokens` | `[B,1+R+N,D]` | CLS、storage、patch tokens 按顺序拼接 |
| `prenorm` | `[B,1+R+N,D]` | 最后一层 LayerNorm 之前的完整 tokens |
| `intermediate_cls` | `[B,L,D]` | 中间层 CLS |
| `intermediate_mean` | `[B,L,D]` | 中间层 patch 均值 |
| `intermediate_cls_mean` | `[B,L,2D]` | 中间层 CLS 与 patch 均值拼接 |
| `intermediate_patch_tokens` | `[B,L,N,D]` | 中间层 patch tokens |
| `intermediate_patch_map` | `[B,L,D,H,W]` | 中间层空间特征图 |
| `intermediate_storage_tokens` | `[B,L,R,D]` | 中间层 storage tokens |
| `intermediate_all_tokens` | `[B,L,1+R+N,D]` | 中间层完整 tokens |
| `intermediate_prenorm` | `[B,L,1+R+N,D]` | 中间层未归一化 tokens |

`B` 是图片数，`D` 是通道数，`N` 是 patch 数，`R` 是 storage token 数，`L` 是选取的层数，`P` 是投影维度，`F` 是模型默认特征维度。表中的 token 形状以 ViT 为例，实际维度随架构和输入尺寸变化

* `layers`：填写中间层编号，例如 `-1` 为最后一层，`8,9,10,11` 或 `-4,-3,-2,-1` 为 ViT-B 的最后四层。层编号从 0 开始，输出顺序与填写顺序一致，单层也保留 `L=1`

* `intermediate_norm`：是否对中间层应用模型的 LayerNorm，默认开启；`intermediate_prenorm` 始终不应用 LayerNorm

* LSNet 支持 `default` / `backbone`；没有 projector 或 storage tokens 的模型不能选择对应输出

* ConvNeXt 的 CLS 表示全局池化，`prenorm` 为未归一化的空间 tokens。不同 stage 的形状可能不同，请一次选择一个 stage

> ps:分类头的池化方式不受这里的选择影响。同一轮图片比较请使用同一个模型、特征类型和预处理设置。

### 3. 从特征生成关系图和分析图

想看几张图片之间的关系，可以按这个方式连接：

```text
图片批次 + Kaloscope Model Loader
              ↓
    Kaloscope Extract Features
              ↓ TENSOR
    Kaloscope Feature Analysis
              ↓ IMAGE
      Preview Image / Save Image
```

**Kaloscope Feature Analysis** 直接使用已有特征，不再执行推理，输出

* `visualization`：图像，可以连接预览或保存节点

* `analysis_json`：距离、相似度、近邻、簇标签、投影坐标和统计结果，方便下游读取

* `distance_matrix`：`[B,B]` 距离 TENSOR

节点在 **Kaloscope/Analysis** 下。同一份特征可以接多个分析节点，分别生成不同图表。可选 `images` 只用来显示缩略图，图片数量和顺序要与特征一致

也可以用 **Kaloscope Image Analysis** 直接接图片与模型；它提取特征后制图，同时输出 `features`，可以再连接其他分析节点

> ps:ComfyUI 中组成图片批次前需要统一尺寸。想看类似 KMeans 的关系分组，先选择 `relationship_graph`，再设置 `cluster_method=kmeans` 和 `n_clusters`

| chart_type | 图表用途 |
| --- | --- |
| `relationship_graph` | 近邻关系网络，MDS 布局、聚类颜色/标记，连线数字表示原始特征距离 |
| `distance_heatmap` | 两两距离热图 |
| `similarity_heatmap` | 两两余弦相似度热图 |
| `pca_scatter` | PCA 二维散点，显示解释方差比例 |
| `mds_scatter` | 近似保持距离的二维散点 |
| `tsne_scatter` | t-SNE 邻域结构散点 |
| `dendrogram` | 层次聚类关系树 |
| `nearest_neighbors` | 指定图片的最近邻排行 |
| `distance_distribution` | 图片对的距离分布与簇内/簇间距离 |
| `silhouette` | 各图片轮廓系数及平均值 |
| `cluster_sizes` | 各簇数量，包含 DBSCAN 噪声 |
| `pca_variance` | PCA 方差解释率、累计比例与有效秩 |
| `feature_statistics` | 特征范数、平均绝对值和标准差 |
| `feature_heatmap` | 图片与高方差特征维度的数值热图 |
| `dimension_correlation` | 特征维度之间的 Pearson 相关 |
| `cluster_centroid_heatmap` | 各簇平均特征热图 |
| `outlier_scores` | 最近邻距离均值，用来查看孤立程度 |
| `patch_energy` | patch 特征 L2 范数的空间分布 |

常用参数：

* `metric`：`cosine`、`euclidean`、`manhattan`。余弦距离为 `1-cosine_similarity`，相似度热图始终显示余弦相似度；

* `normalize`：是否在距离和聚类前按图片做 L2 归一化，默认开启。原始特征统计与 patch 能量使用归一化之前的输入；

* `cluster_method`：`kmeans`、`agglomerative`、`dbscan`、`none`。KMeans 使用欧氏向量目标，agglomerative/DBSCAN 使用所选距离；

* `n_clusters`：KMeans/agglomerative 的簇数；图片或不同向量数量不足时自动减少。`dbscan_eps` / `dbscan_min_samples` 为 DBSCAN 参数；

* `top_k`：关系连线、近邻排行和孤立得分的邻居数；`reference_index` 选择查询图片，从 0 开始；

* `labels`：每行一个名字或 JSON 数组，顺序与输入图片一致；

* `seed` / `perplexity`：随机种子与 t-SNE 参数；

* `max_dimensions`：特征热图最多显示多少个高方差维度；`heatmap_order` 选择 `cluster` 或 `input` 排序；

* `width` / `height`：输出分辨率，范围 512–4096 像素；`grid_width` 指定 patch 网格列数，0 自动推断正方形网格

局部和多层特征需要按输出形状选择 `tensor_layout`：

| 特征类型 | tensor_layout |
| --- | --- |
| 全局向量 `[B,D]` | `vectors` 或 `auto` |
| patch/storage/all tokens `[B,N,D]` | `tokens` |
| patch_map `[B,D,H,W]` | `spatial` |
| 中间层全局向量 `[B,L,D]` | `layer_vectors` |
| 中间层 tokens `[B,L,N,D]` | `layer_tokens` |
| 中间层 patch_map `[B,L,D,H,W]` | `layer_spatial` |

`layer_index` 选择输入特征中的层位置，默认最后一层；`layer_pooling=mean` 对选取的层取均值。`token_pooling` 可选 `mean` 或 `flatten`，`tensor_layout=flatten` 可将每张图片的其余维度直接展平

Image Analysis 和界面缓存会根据特征类型处理布局。直接传 TENSOR 时，四维输入请明确选择 `spatial` 或 `layer_tokens`

## 第四步：作为 WebUI 插件或单独启动

### 1. WebUI 插件

将插件放到 WebUI 的 `extensions/comfyui-kaloscope/`，重启后打开 **Kaloscope** 页签。支持 AUTOMATIC1111 及兼容其扩展接口的 WebUI

模型放在 WebUI 的 `models/kaloscope/<子文件夹>/`，权重、配置和类别映射的放法与 ComfyUI 相同

### 2. 单独启动

在项目根目录安装界面依赖，然后启动：

```bash
python -m pip install -r requirements-webui.txt
python -m scripts.app
```

浏览器打开 `http://127.0.0.1:7860`。也可以运行 `python scripts/app.py` 或双击 `单独启动.bat`。

模型默认放在项目根目录的 `models/kaloscope/<子文件夹>/`。需要更改模型目录或端口时：

```bash
python -m scripts.app --models-dir D:/models --host 127.0.0.1 --port 7860
```

这里的 `D:/models` 是包含 `kaloscope/` 的根目录，也可通过环境变量 `KALOSCOPE_MODELS_DIR` 指定。

### 3. 界面里怎么用

WebUI 和独立启动使用同一套界面：

1. **Inference**：上传单张图片，选择模型、设备、Top K 和阈值，点击 Infer。有分类头输出分类，无分类头输出特征

2. **Features & Analysis**：上传多张图片，选择特征类型、中间层和批次大小，点击「提取并缓存特征」

3. 选择图表和分析参数，点击「从缓存生成图表」，可以反复换图表，不需要重新提取

4. 下载 PNG、分析 JSON、距离矩阵 CSV，也可以下载 `features.npz`，下次直接导入缓存

5. 「共同特征 / 相似度 / 分组比较」使用同一份缓存，选择查询图片编号；分组比较时，每张图片填写一行组名

`mode=auto` 自动选择分类或特征；`cluster` 只提取特征；`classify` / `both` 分别用于分类、分类加特征，需要模型有分类头

## 第五步：命令行和 API 用法

### 1. 命令行推理

在项目根目录运行，例如模型放在 `models/kaloscope/sharingan/`：

```bash
python inference_artist.py --checkpoint models/kaloscope/sharingan/best.pt --input example.png --device cuda --mode auto --output output
```

`--input` 也可以填写图片目录。`--mode cluster` 提取特征，`--output-type` 选择特征类型，`--layers` 选择中间层，`--no-intermediate-norm` 关闭中间层 LayerNorm。

分类结果保存为 JSON；提取特征时同时保存 `features.npz`，批量提取还会保存 `features.npy` 与图片名称列表

### 2. 命令行制图

```bash
# 从一组图片提取一次特征，再生成关系图
python analysis_cli.py --input images --model-dir models/kaloscope/sharingan --output outputs --device cuda

# 用缓存改画距离热图，不加载模型
python analysis_cli.py --features outputs/features.npz --chart-type distance_heatmap --output outputs

# 使用 patch 特征，一次提取后生成全部 18 类图
python analysis_cli.py --input images --model-dir models/kaloscope/sharingan --output-type patch_tokens --all-charts --output outputs
```

每种图会保存 PNG、JSON 和距离矩阵 CSV，参数与界面对应，例如 `--metric euclidean --no-normalize --cluster-method dbscan --dbscan-eps 0.5`。

还可以从缓存获取共同特征、相似度或分组比较：

```bash
python analysis_cli.py --features outputs/features.npz --operation common_features --output outputs
python analysis_cli.py --features outputs/features.npz --operation similarity --reference-index 0 --output outputs
python analysis_cli.py --features outputs/features.npz --operation compare_groups --groups groups.txt --output outputs
```

`groups.txt` 每行一个组名，与图片顺序一致。`--options-json options.json` 可以读取分析参数，命令行显式填写的参数优先；所有参数可通过 `python analysis_cli.py --help` 查看

> ps:界面、API 和命令行共用 `.npz` 缓存；`.npy` 只包含数组。生成全部图表需要 patch tokens/patch_map，因为全局向量不能生成 patch 能量图

### 3. API

WebUI 和独立服务都提供 `/kaloscope/v1/` API。独立服务打开 `/docs` 可以查看请求参数

| 接口 | 用途 |
| --- | --- |
| `GET /kaloscope/v1/models` | 查看模型目录、特征类型和图表类型 |
| `POST /kaloscope/v1/infer` | 单图分类或提取特征 |
| `POST /kaloscope/v1/features` | 批量提取，返回可复用的特征缓存 |
| `POST /kaloscope/v1/analyze` | 从特征、缓存或图片批次生成图表 |
| `POST /kaloscope/v1/feature-tools` | 共同特征、相似度和分组比较 |

单图推理：

```json
{
  "input_image": "<图片的Base64>",
  "model_name": "sharingan",
  "device": "cuda",
  "mode": "auto",
  "top_k": 5,
  "threshold": 0.0
}
```

`/infer` 返回 `results` 和 `info`。分类列表包含 `class_id`、`class_name`、`probability`；特征保存在 `results.features`。可以填写 `output_type`、`layers`、`intermediate_norm` 选择特征

批量提取 `/features`：

```json
{
  "input_images": ["<图片1的Base64>", "<图片2的Base64>"],
  "model_name": "sharingan",
  "device": "cuda",
  "output_type": "patch_tokens",
  "batch_size": 2,
  "labels": ["image1", "image2"]
}
```

返回 `cache_base64`、`shape`、`labels`、`output_type`。把 `cache_base64` 传给 `/analyze` 就可以制图：

```json
{
  "cache_base64": "<上一步返回的特征缓存>",
  "chart_type": "relationship_graph",
  "options": {
    "metric": "cosine",
    "cluster_method": "kmeans",
    "n_clusters": 2,
    "top_k": 1
  }
}
```

返回 PNG 的 `image_base64`、分析对象 `analysis`、`distance_matrix` 和可复用缓存。`cache_base64` 解码后是完整的 `features.npz` 文件，可以导入界面或用于命令行

`/analyze` 也可以接 `features: [[...], [...]]` 数组，填写对应 `output_type` 和 `labels`；或者接 `image_batch`，结构与 `/features` 的请求相同。三种输入选择一种即可，`thumbnail_images` 是可选缩略图，不参与特征推理

共同特征、相似度和分组比较可以这样调用 `/feature-tools`：

```json
{
  "cache_base64": "<特征缓存>",
  "operation": "compare_groups",
  "reference_index": 0,
  "groups": ["artist_a", "artist_a", "artist_b", "artist_b"]
}
```

`groups` 数量与缓存图片数一致。`common_features` 返回平均向量与样本数；`similarity` 返回查询图片与批次内所有图片的相似度，包含自身；`compare_groups` 返回组名、相似度和最接近的组

> ps:图片 Base64 不带 `data:image/...;base64,` 前缀。`options` 使用分析节点的同名参数，API 的 `labels` 填字符串数组；共同特征和分组比较也支持 `tensor_layout`、`layer_index`、`layer_pooling`、`token_pooling`

### 致谢

感谢 [@heathcliff01](https://huggingface.co/heathcliff01) 训练模型

### lsnet训练代码

https://github.com/spawner1145/lsnet-test.git

### dinov3训练代码

https://github.com/Chenkin-x/kaloscope-dinov3.git

## Citation

```BibTeX
@misc{wang2025lsnetlargefocussmall,
    title={LSNet: See Large, Focus Small},
    author={Ao Wang and Hui Chen and Zijia Lin and Jungong Han and Guiguang Ding},
    year={2025},
    eprint={2503.23135},
    archivePrefix={arXiv},
    primaryClass={cs.CV},
    url={https://arxiv.org/abs/2503.23135},
}

@misc{simeoni2025dinov3,
    title={{DINOv3}},
    author={Sim{\'e}oni, Oriane and Vo, Huy V. and Seitzer, Maximilian and Baldassarre, Federico and Oquab, Maxime and Jose, Cijo and Khalidov, Vasil and Szafraniec, Marc and Yi, Seungeun and Ramamonjisoa, Micha{\"e}l and Massa, Francisco and Haziza, Daniel and Wehrstedt, Luca and Wang, Jianyuan and Darcet, Timoth{\'e}e and Moutakanni, Th{\'e}o and Sentana, Leonel and Roberts, Claire and Vedaldi, Andrea and Tolan, Jamie and Brandt, John and Couprie, Camille and Mairal, Julien and J{\'e}gou, Herv{\'e} and Labatut, Patrick and Bojanowski, Piotr},
    year={2025},
    eprint={2508.10104},
    archivePrefix={arXiv},
    primaryClass={cs.CV},
    url={https://arxiv.org/abs/2508.10104},
}

```
