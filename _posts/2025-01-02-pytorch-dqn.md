---
title: 使用PyTorch实现强化学习
layout: post
---

> 一直以为强化学习在各种神经网络框架中会有单独的模块，然而试了才发现其用的就是一般的神经网络，只是在使用上有一套独特的逻辑——强化学习的核心是通过神经网络对行动价值函数Q(s,a)进行预测，进而选择价值函数最高的行动从而实现行动的最优化。

#### 1. 强化学习的基本概念

> 强化学习相对一般类型的机器学习，其特点是通过和环境的交互中收集数据作为输入，并据此获得实现奖励最大化。
>
> 收益：t时刻后的奖励总和，其中每经理一次行动都会将之前每次的奖励都乘以折现率（0~1之间），避免出现溢出问题，同时也使得近期能够获得的奖励权重更高。

\\[G\_t = R\_t + \gamma R\_{t+1} + \gamma^2 R\_{t+2} + ...\\ = R\_t + \gamma G\_{t+1}\\]

> 状态价值函数V：在某状态S下以策略P行动获得收益的期望值。
>
> 行动价值函数Q：在某状态S下采取采取A行动，此后以策略P行动获得收益的期望。
>
> 强化学习的过程为通过蒙特卡洛方法对行动价值函数Q进行估计，从而能够在选择行动是选取使Q最大的行动A。

#### 2. 构建强化学习的数据结构
>
> 首先需要使用一个队列来对每一次行动进行记录，从而以此来训练对应的神经网络，可以借助Python中的dqueue，对应的功能代码如下
>
```python
class ReplayMemory():
    def __init__(self, capacity):
        self.memory = collections.deque(maxlen=capacity);
>
    def add(self, data):
        self.memory.append(data);
>
    def sample(self, batch_size):
        return random.sample(self.memory, batch_size);
>
    def __len__(self):
        return len(self.memory);
```
> 提供了增加一项数据和从中抽取一定量数据两个方法，同时还实现了计算加入的数据总量的方法。
>
> 然后将设计几个进行数据预测的神经网络结构，这里使用三层全连接网络，其结构如下
>
```python
class QNet(torch.nn.Module):
    def __init__(self, n_observations, n_action):
        super(QNet, self).__init__();
        self.layer1 = torch.nn.Linear(n_observations, 128);
        self.layer2 = torch.nn.Linear(128, 128);
        self.layer3 = torch.nn.Linear(128, n_action);
>
    def forward(self, x):
        x = torch.nn.functional.relu(self.layer1(x));
        x = torch.nn.functional.relu(self.layer2(x));
        return self.layer3(x);
```
> 其输入数据的维度为状态参数的维度，对应相关的状态参数，输出数据维度为行动种类的维度，对应每种行动的行动价值函数值。相当于拟合函数Q(S,A)
> 
> 一般在进行强化学习时会使用两个神经网络，一个通过数据进行参数的优化，并使用改神经网络决定行动，同时将参数定期将参数同步给另一个（不太明白问什么需要这样操作，感觉可能是为了提高收敛性）。
```python
#当前神经网络
policy_net = QNet(n_observations, n_actions).to(device);
#目标神经网络
target_net = QNet(n_observations, n_actions).to(device);
```

#### 3. 强化学习的流程
> 进行episodes轮的行动，并在每轮中以一定的概率根据预测到的Q最大的行动进行行动，一定的概率随机行动，并记录每次行动的状态和对应的奖励为一组数据，并放置到队列中，当队列有足够的长度后，每次行动后抽取一定的数据样本，通过计算和神经网络分别得到行动价值函数，并通过优化神经网络当前神经网络减两者之间的误差。在完成一次当前神经网络的训练后，将当前神经网络和目标神经网络进行加权平均作为新的目标神经网络，如此完成一轮行动。
```python
for i in range(0, n_episodes):
    state, info = env.reset();
    state = torch.tensor(state, dtype=torch.float32, device=device).unsqueeze(0);
    for t in itertools.count():
        with torch.no_grad():
        #根据当前网络得到行动最大价值的行动，并以一定概率选择该行动，一定概率随机行动
            action = policy_net(state).max(1).indices[0].item() \
                if random.random() > EPS_END else numpy.random.choice(n_actions);
        #获取行动后的需结果
        observation, reward, terminated, truncated, _ = env.step(action);
        next_state = None if terminated \
            else torch.tensor(observation, dtype=torch.float32, device=device).unsqueeze(0);
>
        #将结果记录到记忆中，并更新当前状态
        memory.add((state, action, next_state, reward));
        state = next_state;
>
        #当记忆中的数据大于一次训练所需要的数据，进行一次神经网络的训练
        if len(memory) >= BATCH_SIZE:
            #抽取数据，并进行封装
            batch = memory.sample(BATCH_SIZE);
            state_batch = [];
            action_batch = [];
            reward_batch = [];
            next_state_batch = [];
 >           
            for item in batch:
                state_batch.append(item[0]);
                next_state_batch.append(item[2]);
                action_batch.append(torch.tensor([[item[1]]], device=device));
                reward_batch.append(torch.tensor([[item[3]]], device=device));
            state_batch = torch.cat(state_batch);
            action_batch = torch.cat(action_batch);
            reward_batch = torch.cat(reward_batch);
>
            #剔除为终止位置的数据(终止位置行动价值函数为0)
            non_final_mask = torch.tensor(tuple(map(lambda s: s is not None, next_state_batch)), \
                device=device, dtype=torch.bool);
            non_final_next_states = torch.cat([s for s in next_state_batch if s is not None]);
>
            #使用当前神经网络预测行动价值函数
            state_action_values = policy_net(state_batch).gather(1, action_batch);
            next_state_values = torch.zeros(BATCH_SIZE, device=device);
>
            with torch.no_grad():
                #使用目标神经网络预测下一状态的价值函数
                next_state_values[non_final_mask] = target_net(non_final_next_states).max(1).values;
            expected_state_action_values = (next_state_values.unsqueeze(1) * GAMMA) + reward_batch;
>
            #对当前网络的预测进行优化
            loss_fn = torch.nn.SmoothL1Loss();
            loss = loss_fn(state_action_values, expected_state_action_values);
            optim.zero_grad();
            loss.backward();
            torch.nn.utils.clip_grad_value_(policy_net.parameters(), 100);
            optim.step();
>       
        #将当前网络同步到目标网络，这里使用了当前网络和目标网络的加权平均进行同步，一定程度提高收敛性 
        target_net_state_dict = target_net.state_dict();
        policy_net_state_dict = policy_net.state_dict();
        for key in policy_net_state_dict:
            target_net_state_dict[key] = policy_net_state_dict[key] * \
                TAU + target_net_state_dict[key] * (1 - TAU);
        target_net.load_state_dict(target_net_state_dict);
>
        if terminated or truncated == True:
            print(i, t);
            break;
```

#### 4. 总结

>  强化学习的过程是通过蒙特卡洛方法对于行动价值函数Q(s,a)进行拟合的过程，因为一旦能够准确预测行动价值函数，就很容易选择行动价值最大化的行为进行行动，同时由于对于行动价值函数的预估可能存在偏差，可始终保留一定概率继续通过蒙特卡洛方法修正行动价值函数，使得得到的模型可以不断的提升性能。
