---
title: SlotWeight
date: 2017-12-13 18:08:39
tags:
 - 深度学习
layout: post
---

今天看到一篇文章[Continual Learning with Deep Generative Replay](https://arxiv.org/abs/1705.08690)提到多任务学习时，如果按顺序一个任务一个任务学的话，已经学会的任务会很快被忘记。联想到人学习东西的方式，似乎是有不同的“区块”，在应用不同技能的时候调用不同区块的记忆，而不是把所有的技能混在一起的。那么我们训练神经网络的时候能不能也引入这样的机制呢？让神经网络学习一些不同的规律，或者拥有多个记忆，并且根据面对的数据来选择调用哪段记忆。

<!-- more -->

## 实验场景
假设数据有两（多）种不同的分布，

$$ P(y|x) = \sum P_i(y|x)I_{x\in S_i} $$
其中$$\cup^iS_i=U, \cap^iS_i=\emptyset$$，I是示性函数.

我们希望神经网络自动根据数据学习这两种分布，并学习自动选择正确的分布。

## 生成数据
简单起见，我们只用线性模型生成数据，$$X \sim N(0, 1)^2$$, $$Y=W^TX$$.当$$x[0] > x[1]$$时$$W=(1, -1)$$;当$$x[1] \ge x[0]$$时$$W=(-1, 1)$$.

```python
def generate_data(n):
    w1 = torch.FloatTensor([[1.0, -1.0]]).expand(n, -1)
    w2 = torch.FloatTensor([[-1.0, 1.0]]).expand(n, -1)
    x = torch.randn(n, 2)
    p = (x[:, 0] > x[:, 1]).float().unsqueeze(1).expand(-1, 2)
    w = (1-p) * w1 + p * w2
    y = (x * w).sum(1).unsqueeze(1)
    return TensorDataset(x, y)
```

## 模型

模型有两个基本组件：一个是Linear+Softmax，用注意力机制选择使用哪一个weight来计算，另一个就是简单的Linear，使用选出来的weight来进行线性变换。

```python
class SlotWeight(nn.Module):
	def __init__(self, num_slots, input_size, output_size):
		self.input_size = input_size
		self.output_size = output_size
		super(SlotWeight, self).__init__()
		self.weight = nn.Parameter(torch.randn(num_slots, input_size * output_size).float())
		self.bias = nn.Parameter(torch.randn(num_slots, output_size).float())
		self.selector = nn.Sequential(OrderedDict([
			('linear', nn.Linear(input_size, num_slots)), 
			('softmax', nn.Softmax())
		]))

	def forward(self, x):
		batch_size = x.size(0)
		selector_weight = self.selector(x)
		weight = torch.mm(selector_weight, self.weight).view(batch_size, self.input_size, self.output_size)
		bias = torch.mm(selector_weight, self.bias).view(batch_size, 1, self.output_size)
		y = torch.baddbmm(bias, x.unsqueeze(1), weight).squeeze(1)
		return y
```

## 实验

```python
train_set = generate_data(10000)
validate_set = generate_data(1000)

model1 = nn.Linear(2, 1)
model2 = SlotWeight(2, 2, 1)

def run(model):
    loss_fn = nn.MSELoss()
    optimizer = optim.RMSprop(model.parameters(), lr=0.01)
    dl = DataLoader(train_set, 64, shuffle=True)
    for epoch in range(100):
        for x, y in dl:
            x, y = autograd.Variable(x), autograd.Variable(y)
            y_ = model(x)
            loss = loss_fn(y_, y)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
    valid_x = autograd.Variable(validate_set.data_tensor, volatile=True)
    valid_y = autograd.Variable(validate_set.target_tensor, volatile=True)
    valid_y_ = model(valid_x)
    loss = loss_fn(valid_y_, valid_y)
    print(loss.data[0])

def main():
    run(model1)
    run(model2)
    print(model2.selector.linear.weight.data)
    print(model2.selector.linear.bias.data)

```
使用简单的线性单元作为对照，结果线性单元的loss为`0.767`，而SlotWeight的loss只有`2.719e-5`，显然SlotWeight单元学习到了两种不同的分布。我们来看picker的weight长成什么样子：
$$
\begin{pmatrix}
16.8119 & -16.5320 \\\\
-16.4574 & 16.9866 \\\\
\end{pmatrix}
$$

显然这个矩阵学到了要计算X[0]和X[1]的差，这是一个好事。另外，权重的绝对值非常大，说明它不仅要计算这个差，还要把它映射到{-1, 1}上面，成为sign函数，这也和真实的分布是一致的。至此我们的SlotWeigth完美学到了数据的真实分布！

## 思考
SlotWeight只是一个非常简单的单元而已，在我们的实验数据上效果又如此好，为什么没有得到大家的青睐呢？有以下几个原因：

- 通过设置多组权重来学习不同的规律，会大大增加参数数量，也增加过拟合风险
- 每组权重互相独立，如果设置N组参数，则每组参数平均只有1/N的数据可以用来训练，样本的利用率太低
- 这一点是我认为最重要的，也就是对于大多数图像识别、语音识别等问题，数据并不具备“多种不同分布”这个性质，我们总是假设数据是采样自相同分布的，那么使用SlotWeight意义就不大。但是对于金融市场，在不同的时期可能有不同的规律，如果这些规律是重复的并且可识别的，SlotWeight模型就有可能得到利用。

接下去我会考虑把类似的机制放到LSTM里，让LSTM能选择在多个slot中读写信息，而不是把所有记忆放在同一个向量中。