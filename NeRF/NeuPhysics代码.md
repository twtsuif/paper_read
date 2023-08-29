# neuphysics代码

## 数据集

下载数据集到项目目录https://drive.google.com/drive/folders/1PGHkcNolUZ3ld_e5jFtr8OB5_EMWB4sR?usp=sharing



camera.npz相机参数	遵循IDR中的数据格式 每个图片对应4个矩阵  world_mat表示世界到图像投影矩阵，scale_mat表示归一化矩阵

sparse_points_interest.ply场景ROI  感兴趣区域

image每张图片

mask目标掩码  (应该是照搬的NeuS数据集)



### 自定义数据集

抄的NeuS



#### COLMAP

1. 运行COLMAP SfM

```shell
cd colmap_preprocess
python img2poses.py ${data_dir}
```

会得到稀疏点云

2. 定义ROI

原始稀疏点云可能有噪声，自己清理，比如使用Meshlab

另存为${data_dir}/sparse_points_interest.ply

```shell
python gen_cameras.py ${data_dir}
```



## 运行

```shell
--mode
train训练
validate_mesh 从单个帧或者时间步的训练模型中，提取表面
validate_mesh_sequence 从整个运动序列的训练模型中，提取表面
interpolate_<img_idx_0>_<img_idx_1> 视图插值  给两张视图索引，然后插值
train_physics_gravity_warp 训练物理参数
validate_image
validate_image_sequence
validate_image_sequence_original_motion
fore_back_ground 导出一个基于刚性蒙版阈值和场景边界框的前景与背景分离的网格

--conf配置文件
--val_frame_idx帧的索引
--val_rigidity_threshold
--mcube_threshold
--is_continue
--gpu是否使用GPU
--case哪一个运行示例
--seq
--edit_type 编辑类型 # move_fg delete_fg duplicate_fg
```



#### 运行模板

```shell
python run.py --mode validate_mesh_sequence --conf ./confs/hands.conf --case hands/preprocessed/ --is_continue

python run.py --mode validate_image --val_frame_idx 10 --conf ./confs/hands.conf --case hands/preprocessed/ --is_continue
```

#### 输出

```shell
Load data: Begin
Load data: End
[run.py:108 -             __init__() ] Find checkpoint: ckpt_300000.pth
[run.py:306 -      load_checkpoint() ] End

Validating mesh 0/227...
fore_back_ground
[load.py:222 -            load_mesh() ] loaded <trimesh.PointCloud(vertices.shape=(10429, 3))> using load_ply
[constants.py:135 -                timed() ] load_mesh executed in 0.0025 seconds.
threshold: 0.0
Bound Min: tensor([-0.4834, -0.6215, -0.4361])
Bound Max: tensor([0.4834, 0.6215, 0.4361])
len(X) 8
/root/anaconda3/envs/neuphysics/lib/python3.8/site-packages/torch/functional.py:568: UserWarning: torch.meshgrid: in an upcoming release, it will be required to pass the indexing argument. (Triggered internally at  ../aten/src/ATen/native/TensorShape.cpp:2228.)
  return _VF.meshgrid(tensors, **kwargs)  # type: ignore[attr-defined]
[export.py:72 -          export_mesh() ] Exporting 2105706 faces as PLY
[run.py:570 -     fore_back_ground() ] End
```





## torch

### torch.meshgrid

生成网格、生成坐标

输入：两个数据类型相同的一维张量

输出：两个张量，行数：第一个输入张量的元素个数，列数：第二个输入张量的元素个数

第一个输出张量填充第一个输入张量，各行相同；第二个输出张量填充第二个输入张量，各列相同



### torch.autograd.grad

Jacobi矩阵 函数的一阶偏导组成的矩阵

![image-20230715123343246](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-15/2cc9ee7587c893dc183d889ad3f00996--f80a--image-20230715123343246.png)

## numpy

### np.linalg.norm

线性代数求范数



## library

### trimesh

加载和保存ply文件



### PyMCubes

Marching Cubes算法
