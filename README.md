# 前言
本章介绍如何使用Tensorflow实现简单的声纹识别模型，首先你需要熟悉音频分类，没有了解的可以查看这篇文章[《基于Tensorflow实现声音分类》](https://blog.doiduoyi.com/articles/1587654005620.html) 。基于这个知识基础之上，我们训练一个声纹识别模型，通过这个模型我们可以识别说话的人是谁，可以应用在一些需要音频验证的项目。

使用环境：

 - Python 3.7
 - Tensorflow 2.3.0

# 模型下载
| 数据集 | 准确率 | 下载地址 |
| :---: | :---: | :---: |
| [中文语音语料数据集](https://github.com/KuangDD/zhvoice) | 训练中 | [训练中]() |

# 安装环境
1. 安装Tensorflow，如果已经安装过Tensorflow，测无需再次安装。
```shell
pip install tensorflow==2.3.0 -i https://mirrors.aliyun.com/pypi/simple/
```

2. 安装其他依赖库，命令如下。
```shell
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
```

**注意：** [libsora和pyaudio安装出错解决办法](docs/faq.md)

# 创建数据
本教程笔者使用的是[中文语音语料数据集](https://github.com/KuangDD/zhvoice) ，这个数据集一共有3242个人的语音数据，有1130000+条语音数据。如果读者有其他更好的数据集，可以混合在一起使用，但要用python的工具模块aukit处理音频，降噪和去除静音。

首先是创建一个数据列表，数据列表的格式为`<语音文件路径\t语音分类标签>`，创建这个列表主要是方便之后的读取，也是方便读取使用其他的语音数据集，语音分类标签是指说话人的唯一ID，不同的语音数据集，可以通过编写对应的生成数据列表的函数，把这些数据集都写在同一个数据列表中。

在`create_data.py`写下以下代码，因为[中文语音语料数据集](https://github.com/KuangDD/zhvoice) 这个数据集是mp3格式的，作者发现这种格式读取速度很慢，所以笔者把全部的mp3格式的音频转换为wav格式，在创建数据列表之后，可能有些数据的是错误的，所以我们要检查一下，将错误的数据删除。执行下面程序完成数据准备。
```shell
python create_data.py
```

# 数据读取
有了上面创建的数据列表和均值标准值，就可以用于训练读取。主要是把语音数据转换短时傅里叶变换的幅度谱，使用librosa可以很方便计算音频的特征，如梅尔频谱的API为`librosa.feature.melspectrogram()`，输出的是numpy值，可以直接用PaddlePaddle训练和预测。跟梅尔频谱同样很重要的梅尔倒谱（MFCCs）更多用于语音识别中，对应的API为`librosa.feature.mfcc()`。在本项目中使用的API分别是`librosa.stft()`和`librosa.magphase()`。在训练时，使用了数据增强，如随机翻转拼接，随机裁剪。经过处理，最终得到一个`257*257`的短时傅里叶变换的幅度谱。
```python
wav, sr_ret = librosa.load(audio_path, sr=sr)
linear = librosa.stft(extended_wav, n_fft=n_fft, win_length=win_length, hop_length=hop_length)
linear_T = linear.T
mag, _ = librosa.magphase(linear_T)
mag_T = mag.T
freq, freq_time = mag_T.shape
spec_mag = mag_T[:, :spec_len]
mean = np.mean(spec_mag, 0, keepdims=True)
std = np.std(spec_mag, 0, keepdims=True)
spec_mag = (spec_mag - mean) / (std + 1e-5)
```


# 训练模型
创建`train.py`开始训练模型，使用的是经过修改过的`resnet34`模型，数据输入层设置为`[None, 1, 257, 257]`，这个大小就是短时傅里叶变换的幅度谱的shape，如果读者使用了其他的语音长度，也需要修改这个值。每训练一轮结束之后，执行一次模型评估，计算模型的准确率，以观察模型的收敛情况。同样的，每一轮训练结束保存一次模型，分别保存了可以恢复训练的模型参数，也可以作为预训练模型参数。还保存预测模型，用于之后预测。
```shell
python train.py
```

训练过程中，会使用tensorboard保存训练日志，通过启动tensorboard可以随时查看训练结果，启动命令`tensorboard --logdir=log --host 0.0.0.0`

# 声纹对比
下面开始实现声纹对比，创建`infer_contrast.py`程序，编写`infer()`函数，在编写模型的时候，模型是有两个输出的，第一个是模型的分类输出，第二个是音频特征输出。所以在这里要输出的是音频的特征值，有了音频的特征值就可以做声纹识别了。我们输入两个语音，通过预测函数获取他们的特征数据，使用这个特征数据可以求他们的对角余弦值，得到的结果可以作为他们相识度。对于这个相识度的阈值`threshold`，读者可以根据自己项目的准确度要求进行修改。
```shell
python infer_contrast.py --audio_path1=audio/a_1.wav --audio_path2=audio/b_2.wav
```


# 声纹识别
在上面的声纹对比的基础上，我们创建`infer_recognition.py`实现声纹识别。同样是使用上面声纹对比的`infer()`预测函数，通过这两个同样获取语音的特征数据。 不同的是笔者增加了`load_audio_db()`和`register()`，以及`recognition()`，第一个函数是加载声纹库中的语音数据，这些音频就是相当于已经注册的用户，他们注册的语音数据会存放在这里，如果有用户需要通过声纹登录，就需要拿到用户的语音和语音库中的语音进行声纹对比，如果对比成功，那就相当于登录成功并且获取用户注册时的信息数据。第二个函数`register()`其实就是把录音保存在声纹库中，同时获取该音频的特征添加到待对比的数据特征中。最后`recognition()`函数中，这个函数就是将输入的语音和语音库中的语音一一对比。
有了上面的声纹识别的函数，读者可以根据自己项目的需求完成声纹识别的方式，例如笔者下面提供的是通过录音来完成声纹识别。首先必须要加载语音库中的语音，语音库文件夹为`audio_db`，然后用户回车后录音3秒钟，然后程序会自动录音，并使用录音到的音频进行声纹识别，去匹配语音库中的语音，获取用户的信息。通过这样方式，读者也可以修改成通过服务请求的方式完成声纹识别，例如提供一个API供APP调用，用户在APP上通过声纹登录时，把录音到的语音发送到后端完成声纹识别，再把结果返回给APP，前提是用户已经使用语音注册，并成功把语音数据存放在`audio_db`文件夹中。
```shell
python infer_recognition.py
```
