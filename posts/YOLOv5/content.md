# Technology

我在此，把我大一一整年所学的技术知识做一个整理，无法面面俱到，只可供参考。

# YOLOv5

这是我接触到的最早一个算是实战的项目，当时很多都不懂，基本上是走一步学一步。从安装环境到最后的功能实现，再到后面自己对模型的改进，我将一一列在下面。

## 下载

YOLOv5 的项目地址：

https://github.com/ultralytics/yolov5

可以直接在网页中下载压缩包到本地电脑。

当然你也可以通过下面的方式拉取代码：

```bash
git clone https://github.com/ultralytics/yolov5
```

这种方法后续再作详细说明。

## 环境安装

这是我个人认为每个项目最难的一点，因为环境安装往往会遇到千奇百怪的错误。有可能别人遇到的问题你有，也有可能没有。合理地运用 AI 工具，加上足够的耐心，才可以顺利解决这类问题。

### miniconda 的安装

Miniconda 下载地址：

https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/

进入页面后，按 `Ctrl + F` 搜索对应的版本。这里我推荐下载：

`miniconda-py38_22.11.1-1-Windows-x86_64.exe`

详细安装步骤可以参考：

https://www.bilibili.com/video/BV1bg4y1R7cs

### 虚拟环境创建

`Windows + R` 打开运行，输入 `cmd` 进入终端命令行。

在命令行里面输入：

```bash
conda create -n yolov5 python=3.8
```

激活环境：

```bash
conda activate yolov5
```

换源这里我用的是清华源，用别的也可以：

https://mirrors.tuna.tsinghua.edu.cn/help/pypi/

## 安装 PyTorch

对于 PyTorch 安装，我使用的是轮子的安装方法：

https://download.pytorch.org/whl/cu118

进入这个网址后，找到对应 YOLOv5 所需要的 PyTorch 版本。依旧可以按 `Ctrl + F` 搜索，找到对应的 `torch`、`torchvision` 等文件。根据你模型所需要的版本和电脑对应的版本选择，一般结尾是 `win_amd64.whl`，CUDA 常见版本是 `11.8`，也就是 `cu118`。

下载到本地电脑后，进入创建好的虚拟环境，输入：

```bash
pip install E:\torch-2.4.0+cu118-cp39-cp39-win_amd64.whl
```

把路径替换成你本地实际的文件地址即可，不要带两头的双引号。

后面相应的 `torchvision`、`torchaudio` 也可以用类似的方法安装。这样下载速度通常会远快于直接在线安装。

## YOLOv5 所需的一些版本安装

YOLO 本身也需要一些库，例如 `numpy`、`pillow` 等。安装方式如下：

先进入下载好的 YOLOv5 代码中，打开 `requirements.txt`。

修改里面的参数，具体修改可以参考：

https://www.bilibili.com/video/BV1bg4y1R7cs

修改好之后，进入虚拟环境，输入：

```bash
pip install -r requirements.txt
```

不出意外的话，会出一些意外。这时候查询 AI 并且耐心一点，基本都能解决。

## YOLOv5 具体功能实现操作

### 标注工具下载

YOLOv5 需要手动进行人工标注，因此要用到标注工具。

打开终端，输入：

```bash
pip install labelimg
```

安装好之后就可以对图片进行标注。

### 数据集构建

首先我们先创建两个文件夹，一般是：

`images` 和 `labels`

`images` 中放图片，`labels` 中放标注好的标签文件。它们下面的子文件夹名字通常都为：

- `train`
- `val`

两个子文件中的图片要一一对应。

其中 `train` 和 `val` 中图片的比例大约为 `8:2`，可以根据模型情况进行调整。

运用 `labelimg` 进行标注时，要记得把格式从 VOC 调整为 YOLO 格式，也就是列表中的第六个。

然后把标注生成的 `.txt` 文件保存到上面创建的 `labels` 文件夹里。可以先全部标注完，再去划分训练集和验证集，常见比例是 `8:2` 或者 `7:3`。

### 配置文件参数修改

完成以上步骤之后，我们进入 YOLOv5 代码目录，找到 `data` 文件夹，复制其中一个 `.yaml` 文件。我这里改成 `bvn.yaml`，然后把里面的配置参数修改成自己标注时用到的类别信息。

比如你标注了篮球、羽毛球、足球等，就按照顺序排列好。

以下是一个典型示例，特别注意其中：

- `path: ./datasets`
- `train: images/train`
- `val: images/val`

这些路径必须和你实际数据集路径对应，这样才能保证程序正常运行。可以使用相对路径，也可以使用绝对路径，关键是不要写错。如果有问题，就根据报错慢慢改。

```yaml
# YOLOv5 🚀 by Ultralytics, GPL-3.0 license
# COCO128 dataset https://www.kaggle.com/ultralytics/coco128
# Example usage: python train.py --data coco128.yaml

path: ./datasets
train: images/train
val: images/val
test:

nc: 7
names:
  0: fish
  1: jellyfish
  2: penguin
  3: puffer_fish
  4: shark
  5: stingray
  6: starfish
```

进入具体代码中，在 `train.py` 里需要将以下参数进行修改，把上面的配置文件导入进去。

```python
def parse_opt(known=False):
    parser = argparse.ArgumentParser()
    parser.add_argument('--weights', type=str, default=ROOT / 'yolov5s.pt', help='initial weights path')
    parser.add_argument('--cfg', type=str, default='', help='model.yaml path')

    # 以下这一行就是将配置文件放进来，可以是相对路径，也可以是绝对路径
    parser.add_argument('--data', type=str, default=ROOT / 'E:\\yolov5-master\\data\\bvn.yaml', help='dataset.yaml path')

    parser.add_argument('--hyp', type=str, default=ROOT / 'data/hyps/hyp.scratch-low.yaml', help='hyperparameters path')

    # 以下是训练轮数的修改，可以根据模型反馈出来的收敛曲线进行调整
    parser.add_argument('--epochs', type=int, default=50, help='total training epochs')
    parser.add_argument('--batch-size', type=int, default=16, help='total batch size for all GPUs, -1 for autobatch')
    parser.add_argument('--imgsz', '--img', '--img-size', type=int, default=640, help='train, val image size (pixels)')
    parser.add_argument('--rect', action='store_true', help='rectangular training')
    parser.add_argument('--resume', nargs='?', const=True, default=False, help='resume most recent training')
    parser.add_argument('--nosave', action='store_true', help='only save final checkpoint')
    parser.add_argument('--noval', action='store_true', help='only validate final epoch')
    parser.add_argument('--noautoanchor', action='store_true', help='disable AutoAnchor')
    parser.add_argument('--noplots', action='store_true', help='save no plot files')
    parser.add_argument('--evolve', type=int, nargs='?', const=300, help='evolve hyperparameters for x generations')
    parser.add_argument('--bucket', type=str, default='', help='gsutil bucket')
    parser.add_argument('--cache', type=str, nargs='?', const='ram', help='image --cache ram/disk')
    parser.add_argument('--image-weights', action='store_true', help='use weighted image selection for training')
    parser.add_argument('--device', default='', help='cuda device, i.e. 0 or 0,1,2,3 or cpu')
    parser.add_argument('--multi-scale', action='store_true', help='vary img-size +/- 50%%')
    parser.add_argument('--single-cls', action='store_true', help='train multi-class data as single-class')
    parser.add_argument('--optimizer', type=str, choices=['SGD', 'Adam', 'AdamW'], default='SGD', help='optimizer')
    parser.add_argument('--sync-bn', action='store_true', help='use SyncBatchNorm, only available in DDP mode')
    parser.add_argument('--workers', type=int, default=1, help='max dataloader workers (per RANK in DDP mode)')
    parser.add_argument('--project', default=ROOT / 'runs/train', help='save to project/name')
    parser.add_argument('--name', default='exp', help='save to project/name')
    parser.add_argument('--exist-ok', action='store_true', help='existing project/name ok, do not increment')
    parser.add_argument('--quad', action='store_true', help='quad dataloader')
    parser.add_argument('--cos-lr', action='store_true', help='cosine LR scheduler')
    parser.add_argument('--label-smoothing', type=float, default=0.0, help='Label smoothing epsilon')
    parser.add_argument('--patience', type=int, default=100, help='EarlyStopping patience (epochs without improvement)')
    parser.add_argument('--freeze', nargs='+', type=int, default=[0], help='Freeze layers: backbone=10, first3=0 1 2')
    parser.add_argument('--save-period', type=int, default=-1, help='Save checkpoint every x epochs (disabled if < 1)')
    parser.add_argument('--seed', type=int, default=0, help='Global training seed')
    parser.add_argument('--local_rank', type=int, default=-1, help='Automatic DDP Multi-GPU argument, do not modify')
```

当我们训练完设定的轮数后，训练好的模型会放在 `runs/train` 中，可以根据对应路径去查看。

### 检测部分参数修改

在 `main.py` 函数中，我们需要对以下参数进行修改：

```python
def parse_opt():
    parser = argparse.ArgumentParser()

    # 以下这个文件要换成自己训练后的模型路径
    parser.add_argument('--weights', nargs='+', type=str, default=ROOT / 'E:\\yolov5-master\\runs\\train\\exp52\\weights\\best.pt', help='model path or triton URL')

    # 这里是我们想要检测的内容，可以是视频、图片
    # 设为 "1" 或 "0" 时可以表示摄像头，常见是 0
    parser.add_argument('--source', type=str, default=ROOT / '1', help='file/dir/URL/glob/screen/0(webcam)')

    # 这里也需要更换配置文件
    parser.add_argument('--data', type=str, default=ROOT / 'data/bvn.yaml', help='(optional) dataset.yaml path')

    parser.add_argument('--imgsz', '--img', '--img-size', nargs='+', type=int, default=[640], help='inference size h,w')
    parser.add_argument('--conf-thres', type=float, default=0.5, help='confidence threshold')
    parser.add_argument('--iou-thres', type=float, default=0.4, help='NMS IoU threshold')
    parser.add_argument('--max-det', type=int, default=1000, help='maximum detections per image')
    parser.add_argument('--device', default='', help='cuda device, i.e. 0 or 0,1,2,3 or cpu')
    parser.add_argument('--view-img', action='store_true', help='show results')
    parser.add_argument('--save-txt', action='store_true', help='save results to *.txt')
    parser.add_argument('--save-conf', action='store_true', help='save confidences in --save-txt labels')
    parser.add_argument('--save-crop', action='store_true', help='save cropped prediction boxes')
    parser.add_argument('--nosave', action='store_true', help='do not save images/videos')
    parser.add_argument('--classes', nargs='+', type=int, help='filter by class: --classes 0, or --classes 0 2 3')
    parser.add_argument('--agnostic-nms', action='store_true', help='class-agnostic NMS')
    parser.add_argument('--augment', action='store_true', help='augmented inference')
    parser.add_argument('--visualize', action='store_true', help='visualize features')
    parser.add_argument('--update', action='store_true', help='update all models')
    parser.add_argument('--project', default=ROOT / 'runs/detect', help='save results to project/name')
    parser.add_argument('--name', default='exp', help='save results to project/name')
    parser.add_argument('--exist-ok', action='store_true', help='existing project/name ok, do not increment')
    parser.add_argument('--line-thickness', default=3, type=int, help='bounding box thickness (pixels)')
    parser.add_argument('--hide-labels', default=False, action='store_true', help='hide labels')
    parser.add_argument('--hide-conf', default=False, action='store_true', help='hide confidences')
    parser.add_argument('--half', action='store_true', help='use FP16 half-precision inference')
    parser.add_argument('--dnn', action='store_true', help='use OpenCV DNN for ONNX inference')
    parser.add_argument('--vid-stride', type=int, default=1, help='video frame-rate stride')
```

基本如此，就可以跑通 YOLOv5 并实现目标检测任务。在学习过程中，**足够的耐心** 和 **AI 工具** 的正确使用，是解决问题的关键。

### 一些数据集

如果需要水下数据集，可以去官方常见数据集平台中寻找。很多数据集都自带标注，可以直接使用。

Kaggle 数据集入口：

https://www.kaggle.com/datasets

可以从上面寻找自己需要的数据集。

## 树莓派使用

在我们应用算法的时候，常常会搭载到机器人身上。这时候，树莓派作为一个载体，就显得尤为关键。下面的内容，我将从常用的两个系统，说一些配置、指令，以及常见的一些问题。

## VSCode 远程连接

### 扩展下载

打开 VSCode，进入扩展界面，搜索 SSH 服务，下载 `Remote - SSH`。

然后打开扩展，新建远程，输入：

```bash
ssh <主机名>@IP
```

这里的 `IP` 就是树莓派或者服务器对应的地址。

输入对应的密码，如果是 Linux 系统，就是输入对应账户的密码。等待一会即可远程连接。

## 如今 Ultralytics 的 YOLO 训练方式

这一部分是我后来补充的内容。早期我主要接触的是 YOLOv5 原仓库的训练方式，但如今 Ultralytics 官方已经提供了更统一、更简洁的训练接口。

根据 Ultralytics 官方文档，目前可以直接使用 `ultralytics` 包来训练模型。官方首页示例给出的写法如下。

### 官方 Python 训练方式

```python
from ultralytics import YOLO

# 加载预训练模型
model = YOLO("yolo26n.pt")

# 开始训练
model.train(data="path/to/dataset.yaml", epochs=100, imgsz=640)
```

### 官方 CLI 训练方式

```bash
yolo detect train data=path/to/dataset.yaml epochs=100 imgsz=640
```

### 这一套方式的理解

- `YOLO("yolo26n.pt")` 表示先加载一个官方预训练模型
- `data` 对应你自己的数据集配置文件，也就是 `.yaml`
- `epochs` 是训练轮数
- `imgsz` 是训练图像尺寸
- `detect` 表示当前任务是目标检测

如果你后面不再继续用老版 YOLOv5 仓库的 `train.py` 和 `detect.py` 手改参数方式，那么这一套官方接口其实会更加统一，也更适合后期迁移和维护。

## 最后

无论是早期 YOLOv5 的方式，还是如今 Ultralytics 更统一的官方接口，本质上都离不开三件事：

- 环境要配对
- 数据集要规范
- 报错要耐心看

很多时候，项目能不能真正跑起来，不只是代码问题，更是环境、路径、版本和耐心共同作用的结果。
