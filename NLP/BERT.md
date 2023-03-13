## 微调
https://flyyufelix.github.io/2016/10/03/fine-tuning-in-keras-part1.html
## When to fine-tune Models?

In general, if our dataset is not drastically different in context from the dataset which the pre-trained model is trained on, we should go for fine-tuning. Pre-trained network on a large and diverse dataset like the ImageNet captures universal features like curves and edges in its early layers, that are relevant and useful to most of the classification problems.

Of course, if our dataset represents some very specific domain, say for example, medical images or Chinese handwritten characters, and that no pre-trained networks on such domain can be found, we should then consider training the network from scratch.

One other concern is that if our dataset is small, fine-tuning the pre-trained network on a small dataset might lead to overfitting, especially if the last few layers of the network are fully connected layers, as in the case for VGG network. ***Speaking from my experience, if we have a few thousand raw samples, with the common data augmentation strategies implemented (translation, rotation, flipping, etc), fine-tuning will usually get us a better result.***

If our dataset is really small, say less than a thousand samples, a better approach is to take the output of the intermediate layer prior to the fully connected layers as features (bottleneck features) and train a linear classifier (e.g. SVM) on top of it. SVM is particularly good at drawing decision boundaries on a small dataset.