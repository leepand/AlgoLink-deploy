# 机器学习模型专题

## 概况
构建一个强大的、可扩展的机器学习平台是一项艰巨的工作。随着数据规模的增加，对更多计算能力的需求也在增加。那么如何构建一个只需添加更多硬件就能扩展的系统，而不用担心每次都要更改太多代码呢？答案就是模块化你的算法。


我们能提供什么：

- 从0到1快速搭建推荐引擎
- 流程化管理离线模型流程
- 在线学习模块-mab

<p class="note">
  <b>NOTE:</b> 封装的<code>推荐引擎</code>数据侧封装交互、特征标准化处理流程和ID mapping，模型侧封装基于<code>lightfm</code>的训练、基础预测和基于最近邻的快速预测类。
</p>

## Tutorial 1: 从0到1快速搭建推荐引擎
媒体上可以找到很多关于训练和评估推荐系统的文章，但很少有人解释如何克服建立一个完整的系统所带来的挑战。
大多数开源库并不支持开箱即用的可扩展生产系统。这些挑战通常是：

- **动态预测** – 当你有一个非常大的用户/物品维度时，预先计算所有的建议可能会非常低效--或者说不可能。
- **优化响应时间** – 当你动态创建预测时，你需要的检索时间是非常重要的。
- **频繁更新模型** – 当系统需要加入新的数据时，经常更新模型显得非常关键。
- **根据未知数据的预测** – 这意味着要处理模型未见过的用户或物品，并不断地改变功能。
- **线下与线上的综合系统** – 生产系统中需要模型复杂度与性能间平衡。
- **支持实时检索与实时计算** – 实时特征的处理和最终的排序需要线上进行。优化，模型驱动的标配。

我们使用的是LightFM模型，这是一个非常流行的Python推荐库，它实现了一个混合模型。它最适合于小型到中型的推荐器项目-- 针对这类规模的场景你不需要分布式训练。
**简述不同的推荐方法**
协同式 — 模型只使用协作信息--用户与物品的隐性或显性互动（如观看的电影、评分或喜欢的电影）。他们不使用任何关于实际物品的信息（如电影类别、类型等）。
协同模型可以用很少的数据实现高精度，但无法处理未知的用户或项目（冷启动问题）。
基于内容的 — 模型的工作原理完全是基于关于物品或用户的现有数据--完全忽略了用户和物品之间的交互。- 因此，它们的推荐方式与协同模型非常不同。
基于内容的模型通常：
需要更多的训练数据（几乎每一个用户/物品组合都需要有用户/物品的例子），以及比协同模式更难调参。
但它们可以对物品的项目进行预测，而且与协同模型相比，通常有更好的覆盖面。
混合型推荐器 — 像LightFM一样，将这两种方法结合在一起，克服了很多单独的方法所带来的挑战。他们可以处理新物品或新用户。当你将协同模式部署到生产中时，你经常会遇到这样一个问题，那就是你需要对未见过的用户或物品进行预测--比如当一个新用户注册或访问你的网站时，或者你的内容团队发布了一篇新的文章。通常情况下，你至少要等到下一个训练周期，或者用户与某个物品互动时，你才能对这些用户进行推荐。但混合模型即使在这种情况下也能做出预测。它将简单地使用部分可用的特征来计算推荐。
**混合模型还可以处理缺失的特征。**
有时某些用户和物品的特征缺失了（只是因为你还没能收集到这些特征），如果你依赖的是基于内容的模型，这就是一个问题。
混合推荐器对老用户（那些从训练中知道的用户）和新用户/物品都有作用，只要你有关于他们的特征。这对物品特别有用，但对新用户也很有用（当用户第一次访问你的网站时，你可以询问他们对什么感兴趣）。
另外：关于是否使用深度学习模型，这个问题主要取决软硬两个方面，硬的是你的计算装备，软的是你的业务发展阶段，我们都知道深度学习（神经网络）对稀疏特征的表达方面和记忆非常的优异，也就是说如果你的业务只是处在初中期，特征和交互都没有很丰富，直接上深度学习可能并不能达到很好的效果。深度方法未来一定是趋势，但在工业实践上是不是一个最优的方案，需要根据自己具体情况进行选择和迭代。
系统组件：
![ML Model #1](../assets/images/how-to/recom-sys.jpg)


## Tutorial 2: 流程化管理离线模型流程
In this tutorial, we will demonstrate how to port the [SqueezeNet](https://arxiv.org/abs/1512.03385) computer vision model to RunwayML. We also provide a [RunwayML Model Template](https://github.com/runwayml/model-template) repository that contains boilerplate code for porting a new model.

We recommend forking the [`runwayml/model-squeezenet`](https://github.com/runwayml/model-squeezenet) GitHub repository to your own GitHub user account so that you can follow along with the tutorial.

### Step 1. Create a `runway_model.py` model server file

SqueezeNet is a neural network architecture used for computer vision tasks, optimized for mobile/embedded devices. [PyTorch](https://github.com/pytorch/vision) includes an implementation of SqueezeNet pre-trained on ImageNet that can be used out-of-the-box for object recognition.

Here's the code to classify a single local image (via a hard-coded path) with pre-trained SqueezeNet:

```python
# runway_model.py
import json
from PIL import Image
from torchvision import models, transforms
from torch.autograd import Variable

# Read the index -> label mapping from a file
labels = json.load(open('labels.json'))

# set up preprocessing transformations

normalize = transforms.Normalize(
   mean=[0.485, 0.456, 0.406],
   std=[0.229, 0.224, 0.225]
)

preprocess = transforms.Compose([
   transforms.Scale(256),
   transforms.CenterCrop(224),
   transforms.ToTensor(),
   normalize
])

# initialize the model and download pre-trained weights

model = models.squeezenet1_1(pretrained=True)

# open a local image
img = Image.open('still_life.jpg')

# preprocess the image and convert it into a tensor
img_tensor = preprocess(img)
img_tensor.unsqueeze_(0)
img_variable = Variable(img_tensor)

# do a forward pass and classify the image
fc_out = model(img_variable)
label = labels[str(fc_out.data.numpy().argmax())]
print(label)
# lemon
```

And here's the modified code for the same model wrapped in a server so that RunwayML can access it:


```diff
import json
from PIL import Image
from torchvision import models, transforms
from torch.autograd import Variable
+ import runway
+ from runway.data_types import image, text

labels = json.load(open('labels.json'))

normalize = transforms.Normalize(
   mean=[0.485, 0.456, 0.406],
   std=[0.229, 0.224, 0.225]
)

preprocess = transforms.Compose([
   transforms.Scale(256),
   transforms.CenterCrop(224),
   transforms.ToTensor(),
   normalize
])

+ @runway.setup
+ def setup():
+     return models.squeezenet1_1(pretrained=True)

+ @runway.command('classify', inputs={ 'photo': image() }, outputs={ 'label': text() })
+ def classify(model, input):
+     img = input['photo']
      img_tensor = preprocess(img)
      img_tensor.unsqueeze_(0)
      img_variable = Variable(img_tensor)
      fc_out = model(img_variable)
      label = labels[str(fc_out.data.numpy().argmax())]
+     return { 'label': label }

+ if __name__ == '__main__':
+     runway.run()
```
<p class='subtitle'>Contents of model.py</p>

Behind the scenes, the RunwayML Python SDK uses the `@runway.command()` decorator to set up a server with a `/classify` endpoint that accepts a base64-encoded image. The SDK then converts this image into a `PIL.Image` before passing it to the `classify()` function, which classifies it and returns a text label.

<p class="note"><b>NOTE:</b> You don't have to use the RunwayML Python SDK to set up the server. You can just use any server framework (e.g. Flask) and set up the classification endpoint manually. The SDK just makes things easier by taking care of parsing/serializing inputs/outputs and setting up the endpoints for you.</p>

#### Testing your `runway_model.py` locally

Before you configure your build environment and add your model to RunwayML, you should test it locally to make sure that it works as expected. Using RunwayML's Develop Mode, you can connect directly to your model server in order to test your model before building it remotely. Open your command line and type:

```bash
## Optionally create and activate a Python 3 virtual environment
# virtualenv -p python3 venv && source venv/bin/activate

pip install -r requirements.txt

# Run the entrypoint script
python RunwayML_model.py
```

You should see an output similar to this, indicating your model is running.

```
Setting up model...
Starting model server at http://0.0.0.0:9000...
```

You can now open RunwayML, and click the "From Server" button under "Create New Model."

![Click "From Server"](assets/images/how-to/github-link/from-server.png)

Choose a name for your workspace, and click "Create." 

After making sure that the Port under Server Options matches the port that your model server is running at, click "Connect" to connect to your model server.

![Connect to model server](assets/images/how-to/github-link/dev-model-connect.png)

You can now interact with your model using RunwayML. For example, try dragging an image in the Input area to see your model classify it:

![Classify using your model server](assets/images/how-to/github-link/dev-model-classify.png)

Once you've confirmed your model works correctly locally, you can create a `runway.yml` config file before building the model remotely with RunwayML + GitHub.

### Step 2. Write a `runway.yml` config file

Next you need to write a config file that defines the environment, dependencies, and build steps required to build and run our model. This file is written in [YAML](https://learnxinyminutes.com/docs/yaml/), a human-readable superset of JSON. Below is an example of the `runway.yml` file for Squeezenet:

```yaml
python: 3.6
cuda: 9.2
entrypoint: python runway_model.py
build_steps:
  - pip install -r requirements.txt
```

This file specifies the Python and CUDA versions to use, the entry point command which launches the RunwayML model and HTTP server, and a build step which installs the Python dependencies required to run our model. See the [RunwayML YAML reference page](https://sdk.runwayml.com/en/latest/runway_yaml_file.html) for a full list of config values supported in the `runway.yml` config file.

Peeking at the `requirements.txt` file reveals that the only dependencies for the SqueezeNet RunwayML model are PyTorch and PyTorch Vision, as well as the RunwayML Model SDK itself.

```
torch==1.0.0
torchvision==0.2.1
runway-python
```

<p class="note">
  <b>NOTE:</b> The <code>runway-python</code> module must always be installed via the <code>build_steps</code>. Failing to install this required dependency via the <code>runway.yml</code> file will cause all model builds to fail.
</p>

### Step 3. Upload your code to a GitHub repository

Once your model has a `runway_model.py` and `runway.yml` config file it's time to upload your repository to GitHub. If you forked the `runway/model-squeezenet` repo at the beginning of this tutorial, you should already have a `git remote` that points to the forked repository on your GitHub user account. If you instead cloned the repo, or you started creating a model from scratch instead of using the `runwayml/model-squeezenet` repo, you should publish your code to GitHub as an open source repository now. For the remainder of this tutorial, we will assume the repository being ported is located at `https://github.com/YOUR_USERNAME/runway-model-tutorial`.

### Step 4. Import the model

Once you've pushed your model to GitHub, it's time to import your model in the RunwayML. Open RunwayML and select the _Create New Model From GitHub_ button on the lower left.

![Import Model #1](assets/images/views/model-directory.png)

Authorize RunwayML to access public data on your GitHub account.

![Import Model #2](assets/images/how-to/github-link/github-01.png)

Back in RunwayML, select the repository that contains your RunwayML model; `runway-model-tutorial` in our case.

![Import Model #4](assets/images/how-to/github-link/github-03.png)

### Step 5. Push a new commit to trigger a build

Once you've imported your model to RunwayML, you must push a new commit to GitHub to trigger a model build. When you do so, RunwayML will clone the code from your repository and use its `runway.yml` file to build a Docker image from your model. Each new commit pushed to the `master` branch of GitHub repository linked to a model will trigger a new model version to be built by RunwayML. Select the "Versions" tab in your model page to view all builds, past and present.

![Import Model #6](assets/images/how-to/github-link/model-versions.png)

Click on a model version to view details about the version builds. You will notice that the `default` switch is flipped automatically for the first build. This means that the model is now published by your RunwayML users and is available for others to use. You can set any successfully built version to be the `default` version, but you must have at least one default version. This is an intentional decision for the time being, as _Private Models_ is a feature that is still to come.

![Import Model #7](assets/images/how-to/github-link/logs.png)

You can view model logs during or after a build to debug your model build process.

![Import Model #8](assets/images/how-to/github-link/logs2.png)

Once a model version has been successfully built you can add it to a workspace. You can also add any successful model versions to your personal workspace by hovering over the model version check mark in the "Versions" panel until it becomes a "+" icon, and then selecting it. This allows you to test new model versions before making them the default model version that is published to all RunwayML users.

### Step 6. Add information to your model

Once you've imported your model, you can add additional information to help others understand the model. As the publisher of the model, you can do this from the model page using the **Edit Info** button. We recommend adding these fields for your model:

* **Model Name**: The name of the model. Alphanumeric values, underscores, and hyphens are allowed here. No spaces.
* **Tagline**: A short one-line description of what your model does.
* **Description**: A long description of what your model does. The description can be a paragraph or more and is interpreted as Markdown.
* **Images**: A collection of images showcasing your model. The first image you add in the "Gallery" tab will be used as the thumbnail for the image.
* **Attributions**: A list of names and respective roles for people or organizations that significantly contributed to a model. This usually includes authors or publishers of machine learning papers or frameworks. Be generous with attributions; give credit where credit is due. We intentionally differentiate between a model publisher (the RunwayML user that adds/publishes a model) and the original authors of the model code or paper. This provides flexibility and encourages remix, experimentation, and "forking" of models.
* **LICENSE**: The license defining how the model can be used.
* **Keywords**: A list of tags that can be used to find your model in RunwayML.
* **Performance Notes**: How can users expect the model to perform on GPU or CPU environments?
* **Code, Paper, and More**: A list of links to resources related to your model, such as source code, arXiv papers, blog posts, or more.

### Step 7. (Optional) Add files to your model

Some models may require files that are too large to include in the Github repository, such model checkpoints. RunwayML provides a space to upload large files to include with your model. 

To upload a model file, you need to specify a setup option in `runway_model.py` that has the [`runway.file`](https://sdk.runwayml.com/en/latest/data_types.html#runway.data_types.file) data type. For instance, if your model supports loading checkpoints in a Python pickle format (`.pkl`), you can add a setup option to your model for accepting pickle files in the following way:

```python
# runway_model.py
@runway.setup(options={'checkpoint': runway.file(extension='.pkl')})
def setup(opts):
   checkpoint_path = opts['checkpoint']
   model = load_model_from_checkpoint(checkpoint_path)
   # ...
```

Once you've added a file option to your model and built your model successfully, you can upload files to associate with that option by selecting the "Files" tab in your model page and clicking "Upload File."

![Import Model #9](assets/images/how-to/github-link/model-files.png)

(If you are seeing a "No File Options in Model" message, that means that you either have not added a `file` option, or your model has not built successfully since you added the option. Check the "Versions" tab to see the status of your builds.)

Click "Choose File..." and select the file you want to upload, provide a name for your file, and click "Upload File" to start the uploading process.

![Import Model #10](assets/images/how-to/github-link/model-file-dialog.png)

Once you have uploaded your file, you can now use that file with your model in RunwayML. Add your model to your workspace, and your file should appear in Options on the right hand side:

![Import Model #11](assets/images/how-to/github-link/file-in-options.png)

### Step 8. (Optional) Make your model public!

Once you've successfully built and tested your model, and are satisfied with its results, you can make the model public, allowing anyone in the RunwayML community to use it and make projects with it! 

To make your model public, select the "Settings" tab in your model page, and click on the "Make Public" button.

![Import Model #12](assets/images/how-to/github-link/make-public.png)

## Tutorial 3: Importing the Progressive Growing of GANs (PGAN) Model

This tutorial is posted on the RunwayML Medium blog: [Porting a machine learning model from GitHub to RunwayML in 5 minutes](https://medium.com/runwayml/porting-a-machine-learning-model-from-github-to-runway-in-5-minutes-555c5c9310af)

## RunwayML Python SDK
Developer documentation for the Python SDK to import models: [RunwayML Model SDK Docs](https://sdk.runwayml.com)

## RunwayML Model Template
A boilerplate model template that you can use as a starting point to import models: [RunwayML Model Template](https://github.com/runwayml/model-template)



---
Related Technical Support Resource: [Add Your Own Models](https://support.runwayml.com/en/articles/3037632-add-your-own-models)

