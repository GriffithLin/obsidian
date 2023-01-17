## 找一个小的图像识别数据集
https://github.com/jindongwang/transferlearning/blob/master/data/dataset.md
![[Pasted image 20230113205744.png]]

使用Office+Caltech
mfsan的运行结果
```
Test set: Average loss: 0.0816, Accuracy: 872/958 (91%)

source1 accnum 872, source2 accnum 872
webcam dslr to amazon amazon max correct: 901
```



## 字符串数据的算法，如何改写





# 重写的意义

### 通过阅读代码看到的问题，能否解决？
1、当有多个源的时候，需要重写网络


### TLlib--dann的阅读
#### 1、torch.backends.cudnn.benchmark=True
https://zhuanlan.zhihu.com/p/73711222
设置 `torch.backends.cudnn.benchmark=True` 将会让程序在开始时花费一点额外时间，为整个网络的每个卷积层搜索最适合它的卷积实现算法，进而实现网络的加速。适用场景是网络结构固定（不是动态变化的），网络的输入形状（包括 batch size，图片大小，输入的通道）是不变的，其实也就是一般情况下都比较适用。反之，如果卷积层的设置一直变化，将会导致程序不停地做优化，反而会耗费更多的时间。

### 2、torchvision.transforms.RandomResizedCrop
https://blog.csdn.net/qq_36915686/article/details/122136299


example命令：
```
CUDA_VISIBLE_DEVICES=0 python dann.py data/office31 -d Office31 -s A -t W -a resnet50 --epochs 20 --seed 1 --log logs/dann/Office31_A2W
```
```
CUDA_VISIBLE_DEVICES=0 python dann.py data/OfficeCaltech -d OfficeCaltech -s A -t W -a resnet50 --epochs 20 --seed 1 --log logs/dann/OfficeCaltech_A2W
```

MyMfsan命令
```
CUDA_VISIBLE_DEVICES=0 python MyMfsan.py data1 -d OfficeCaltech -s W D -t A -a resnet50 --epochs 20 --seed 8 --log logs/mfsan/officecaltech_WD2A 

CUDA_VISIBLE_DEVICES=0 python MyMfsan.py data -d Office31 -s W D -t A -a resnet50 --epochs 20 --seed 8 --log logs/mfsan/office31_WD2A 

CUDA_VISIBLE_DEVICES=0 python MyMfsan.py data -d Office31 -s W D -t A -a resnet50 --epochs 20 --seed 8 --log logs/mfsan/office31_WD2A 


CUDA_VISIBLE_DEVICES=0 python MyMfsan.py /data3/linming/Transfer-Learning-Library/examples/domain_adaptation/image_classification/data/office31 -d Office31 -s W D -t A -a resnet50 --epochs 20 --seed 8 --log logs/mfsan/office31_WD2A
```

CUDA_VISIBLE_DEVICES=0 ？


3、遇到问题：RuntimeError: Input type (torch.cuda.FloatTensor) and weight type (torch.FloatTensor) should be the same
torch.cuda.FloatTensor和torch.FloatTensor，gpu版本的tensor和cpu版本的tensor。
意思是模型开启了gpu模式，要求输入为gpu版本的tensor 和 实际输入cpu版本的tensor不匹配。或者反之，模型没有开启gpu模式，输入的tensor为GPU版本。
需要检查模型和数据是否都为gpu模式。
model.cuda() 和source_data, source_label = source_data.cuda(), source_label.cuda()



## 运行参数

已经支持的
```python
# dataset parameters
    parser.add_argument('root', metavar='DIR',

                        help='root path of dataset')
                        
    parser.add_argument('-d', '--data', metavar='DATA', default='Office31', choices=utils.get_dataset_names(),help='dataset: ' + ' | '.join(utils.get_dataset_names()) +' (default: Office31)')
    
	parser.add_argument('-s', '--source', help='source domain(s)', nargs='+')

    parser.add_argument('-t', '--target', help='target domain(s)', nargs='+')

    parser.add_argument('--resize-size', type=int, default=224,

                        help='the image size after resizing')

  

     #action  就是--no-hflip 之后不用指定参数，  自动存为true  

    parser.add_argument('--no-hflip', action='store_true',

                        help='no random horizontal flipping during training')

	parser.add_argument('--norm-mean', type=float, nargs='+',

                        default=(0, 0, 0), help='normalization mean')

    parser.add_argument('--norm-std', type=float, nargs='+',

                        default=(1, 1, 1), help='normalization std')

    parser.add_argument('--train-resizing', type=str, default='ran.crop')

    parser.add_argument('--val-resizing', type=str, default='res.')

  

        #这两个参数 和原文不一样,但是指定train-resizing = ran.crop时，用不到

    parser.add_argument('--scale', type=float, nargs='+', default=[0.08, 1.0], metavar='PCT',

                        help='Random resize scale (default: 0.08 1.0)')

    parser.add_argument('--ratio', type=float, nargs='+', default=[3. / 4., 4. / 3.], metavar='RATIO',

                        help='Random resize aspect ratio (default: 0.75 1.33)')

	# model parameters

    parser.add_argument('-a', '--arch', metavar='ARCH', default='resnet18',

                        choices=utils.get_model_names(),

                        help='backbone architecture: ' +

                             ' | '.join(utils.get_model_names()) +

                             ' (default: resnet18)')

	parser.add_argument('--scratch', action='store_true', help='whether train from scratch.')

	# training parameters
	parser.add_argument('-b', '--batch-size', default=32, type=int,

					metavar='N',

					help='mini-batch size (default: 32)')

	parser.add_argument('--lr', '--learning-rate', default=0.01, type=float,

					metavar='LR', help='initial learning rate', dest='lr')

	parser.add_argument('--seed', default=None, type=int,

					help='seed for initializing training. ')
					
	parser.add_argument("--log", type=str, default='dann',

					help="Where to save logs, checkpoints and debugging images.")

```
还待支持的
```python
# model parameters

    parser.add_argument('--bottleneck-dim', default=256, type=int,

                        help='Dimension of bottleneck')

    parser.add_argument('--no-pool', action='store_true',

                        help='no pool layer after the feature extractor.')


    parser.add_argument('--trade-off', default=1., type=float,

                        help='the trade-off hyper-parameter for transfer loss')

# training parameters


    parser.add_argument('--lr-gamma', default=0.001, type=float, help='parameter for lr scheduler')

    parser.add_argument('--lr-decay', default=0.75, type=float, help='parameter for lr scheduler')

    parser.add_argument('--momentum', default=0.9, type=float, metavar='M',

                        help='momentum')

    parser.add_argument('--wd', '--weight-decay', default=1e-3, type=float,

                        metavar='W', help='weight decay (default: 1e-3)',

                        dest='weight_decay')

    parser.add_argument('-j', '--workers', default=2, type=int, metavar='N',

                        help='number of data loading workers (default: 2)')

    parser.add_argument('--epochs', default=20, type=int, metavar='N',

                        help='number of total epochs to run')

    parser.add_argument('-i', '--iters-per-epoch', default=1000, type=int,

                        help='Number of iterations per epoch')

    parser.add_argument('-p', '--print-freq', default=100, type=int,

                        metavar='N', help='print frequency (default: 100)')


    parser.add_argument('--per-class-eval', action='store_true',

                        help='whether output per-class accuracy during evaluation')



    parser.add_argument("--phase", type=str, default='train', choices=['train', 'test', 'analysis'],

                        help="When phase is 'test', only test the model."

                             "When phase is 'analysis', only analysis the model.")
```