# Smart Stage
## 背景
在一个通常的TensorFlow训练任务中通常由样本数据的读取，图计算构成，样本数据的读取属于IO bound操作，在整个E2E的耗时占据表较大的百分比，从而拖慢训练任务，同时并不能高效的利用计算资源（CPU、GPU）。DeepRec已经提供了stage 功能，它的思想来源于TensorFlow的StagingArea功能，我们在DeepRec提供了API `tf.staged`，用户通过显示的指定图中哪一部分需要stage，以及在Session创建的时候加入`tf.make_prefetch_hook()`在TensorFlow runtime驱动异步执行，从而提高整张图的执行效率。

由于`tf.staged`需要用户指定stage的边界，一方面会增加使用难度，另一方面会导致stage颗粒度不够精细，难以做到更多op的异步执行。因此我们提出了SmartStage功能。用户不需要对TF Graph有OP级别理解的情况下，就可以使stage发挥最大的性能提升。

## 功能说明
在用户的原图中有stage阶段的前提下，通过开启smart stage功能，自动化的寻优最大可以stage的范围，修改实际物理计算图（不影响Graphdef图），从而提高性能。

**注意**：该功能的先决条件是，用户的原图中存在至少一个stage阶段

## 用户接口
`tf.staged`，对输入的 `features` 进行预取，返回预取后的 tensor。

| 参数 | 含义 | 默认值 |
| --- | --- | --- |
| features | 需要异步化执行的 op，可以是 tensor、list of tensor（list 中每一个元素都是 tensor ） 或者 dict of tensor （dict 中的 key 都是 string，value 都是 tensor） | 必选参数 |
|capacity| 缓存的 `items`异步化执行结果的最大个数。 | 1 |
| num_threads | 异步化执行 `items`的线程数。 |1|
|items| `items`依赖的 feed_dict 的 key 的列表 | None，即 `items`不依赖 feed_dict |
| feed_generator | `items`依赖的 feed_dict 的 value 的 generator 对象。Python 中一个 generator 对象是一种通过 yield 产生 list 的方法。通过这个 generator 对象，用户可以使用纯 Python 进行灵活的数据预处理，类似于 tensor_pack，接口与用法见示例。 |None，即 `features`不依赖 feed_dict|
|closed_exception_types| 被识别为正常退出的异常类型 | (`tf.errors.OutOfRangeError`, `errors.CancelledError`) |
| ignored_exception_types | 被识别可忽略跳过的异常类型 | () |

ConfigProro中定义了如下配置选项
```python
sess_config = tf.ConfigProto()
sess_config.graph_options.optimizer_options.do_smart_stage = True
```
Session中加入如下hook
```python
hooks=[tf.make_prefetch_hook()]
with tf.train.MonitoredTrainingSession(hooks=hooks, config=sess_config) as sess:
```
注意事项

- 待异步化的计算应该尽可能少和后续主体计算争抢资源（gpu、cpu、线程池等）
- `capacity` 更大会消耗更多的内存或显存，同时可能会抢占后续模型训练的 CPU 资源，建议设置为后续计算时间/待异步化时间。可以从 1 开始逐渐向上调整
- `num_threads` 并不是越大越好，只需要可以让计算和预处理重叠起来即可，数量更大会抢占模型训练的 CPU 资源。计算公式：num_threads >= 预处理时间 / 训练时间，可以从 1 开始向上调整
- `pai.data.make_prefetch_hook()`一定要加上，否则会hang住

## 代码示例
```python
import tensorflow as tf

filename_queue = tf.train.string_input_producer(['1.txt'])
reader = tf.TextLineReader()
k, v = reader.read(filename_queue)

var = tf.get_variable("var", shape=[100, 3], initializer=tf.ones_initializer())
v = tf.train.batch([v], batch_size=2, capacity=20 * 3)
v0, v1 = tf.decode_csv(v, record_defaults=[[''], ['']], field_delim=',')
xx = tf.staged([v0, v1])

xx[0]=tf.string_to_hash_bucket(xx[0],num_buckets=10)
xx[0] = tf.nn.embedding_lookup(var, xx[0])
xx[1]=tf.concat([xx[1], ['xxx']], axis = 0)
target = tf.concat([tf.as_string(xx[0]), [xx[1], xx[1]]], 0)

config = tf.ConfigProto()
# enable smart stage
config.graph_options.optimizer_options.do_smart_stage = True
# mark target 节点
tf.train.mark_target_node([target])

with tf.train.MonitoredTrainingSession(config=config,
                                       hooks=[tf.make_prefetch_hook()]) as sess:
  for i in range(5):
      print(sess.run([target]))
```
## 性能对比
在modelzoo中的DLRM模型中测试该功能
机型为Aliyun ECS 实例 ecs.hfg7.8xlarge

- Model name: Intel(R) Xeon(R) Platinum 8369HC CPU @ 3.30GHz
- CPU(s): 32
- Socket(s): 1
- Core(s) per socket: 16
- Thread(s) per core: 2
- Memory: 128G

|      |      case       | global steps/sec |
| :--: | :-------------: | :--------------: |
| DLRM | w/o smart stage |  201 (baseline)  |
| DLRM | w/o smart stage |  212 (+ 1.05x)   |



