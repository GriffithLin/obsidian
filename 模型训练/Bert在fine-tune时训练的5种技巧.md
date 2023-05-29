[https://zhuanlan.zhihu.com/p/524036087](https://zhuanlan.zhihu.com/p/524036087)

  

## 1、Adam
其中提到现在的transformer已经更正了adam优化器的问题， 使用transformers.AdamW， 而不用原始的from pytorch_pretrained_bert.optimization import BertAdam。

DNABERT中使用了AdamW。

  

## 2、使用权重初始化。

[Revisiting Few-sample BERT Fine-tuning](https://link.zhihu.com/?target=https%3A//arxiv.org/pdf/2006.05987.pdf)

顶部层的权重（1~6 layers）重新初始化， 可以提高一些指标。  

## 3.weight-decay (L2正则化)——DNABERT已经使用

给出了方法，但是作者尝试了一下，效果不大。  这个我准备明显出现过拟合问题时候，再考虑
```python
param_optimizer = list(multi_classification_model.named_parameters())no_decay = ['bias', 'LayerNorm.bias', 'LayerNorm.weight']optimizer_grouped_parameters = [

    {'params': [p for n, p in param_optimizer

                if not any(nd in n for nd in no_decay)], 'weight_decay': 0.01},

    {'params': [p for n, p in param_optimizer

                if any(nd in n for nd in no_decay)], 'weight_decay': 0.0}]

optimizer = transformers.AdamW(optimizer_grouped_parameters, lr=config.lr, correct_bias=not config.bertadam)
```


### 4.冻结部分层参数（Frozen parameter） ——我的目的

没有给出具体的参考资料。但给出了冻结方法

### 方法1： 设置requires_grad = Falsefor 
```python
param in model.parameters():

    param.requires_grad = False
```
### 方法2： torch.no_grad()
```python
class net(nn.Module):

    def __init__():

        ......

    def forward(self.x):

        with torch.no_grad():  # no_grad下参数不会迭代

            x = self.layer(x)

            ......

        x = self.fc(x)

        return x
```
### 5.warmup & lr_decay  —— DNABERT已经改进使用了warmup

```python
num_train_optimization_steps = len(train_dataloader) * config.epochoptimizer = transformers.AdamW(optimizer_grouped_parameters, lr=config.lr)

scheduler = transformers.get_linear_schedule_with_warmup(optimizer,int(num_train_optimization_steps *0.1),num_train_optimization_steps)
```

[On Layer Normalization in the Transformer Architecture](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/2002.04745)