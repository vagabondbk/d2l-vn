<!-- ===================== Bắt đầu dịch Phần 1 ===================== -->
<!-- ========================================= REVISE PHẦN 1 - BẮT ĐẦU =================================== -->

<!--
# Training on Multiple GPUs
-->

# Huấn luyện trên nhiều GPU
:label:`sec_multi_gpu`

<!--
So far we discussed how to train models efficiently on CPUs and GPUs.
We even showed how deep learning frameworks such as MXNet (and TensorFlow) allow one to parallelize computation and communication automatically between them in :numref:`sec_auto_para`.
Lastly, we showed in :numref:`sec_use_gpu` how to list all available GPUs on a computer using `nvidia-smi`.
What we did *not* discuss is how to actually parallelize deep learning training 
(we omit any discussion of *inference* on multiple GPUs here as it is a rather rarely used and advanced topic that goes beyond the scope of this book).
Instead, we implied in passing that one would somehow split the data across multiple devices and make it work.
The present section fills in the details and shows how to train a network in parallel when starting from scratch.
Details on how to take advantage of functionality in Gluon is relegated to :numref:`sec_multi_gpu_gluon`.
We assume that the reader is familiar with minibatch SGD algorithms such as the ones described in :numref:`sec_minibatch_sgd`.
-->

Tới giờ ta đã thảo luận về cách huấn luyện mô hình trên CPU và GPU một cách hiệu quả.
Trong :numref:`sec_auto_para`, ta còn xem cách mà các framework học sâu như MXNet (và TensorFlow) cho phép ta tự động song song hoá việc tính toán và giao tiếp giữa chúng.
Cuối cùng, :numref:`sec_use_gpu` chỉ cách liệt kê toàn bộ các GPU có trong máy bằng lệnh `nvidia-smi`.
Điều mà ta *không* thảo luận là cách để thực sự song song hoá việc huấn luyện mô hình học sâu.
(Ta bỏ qua vấn để thực hiện *suy luận* trên nhiều GPU ở đây vì nó khá ít khi được sử dụng và là một chủ đề nâng cao nằm ngoài phạm vi của cuốn sách này.)
Chúng ta mới chỉ ngầm hiểu rằng bằng cách nào đó có thể chia dữ liệu cho các thiết bị khác nhau và huấn luyện mạng.
Phần này sẽ bổ sung những chi tiết còn thiếu ấy và chỉ cách huấn luyện song song một mạng học sâu từ đầu.
Chi tiết về cách tận dụng các tính năng của Gluon sẽ nằm ở :numref:`sec_multi_gpu_gluon`.
Chúng tôi giả định rằng bạn đọc đã quen với các thuật toán SGD minibatch được mô tả ở :numref:`sec_minibatch_sgd`.

<!--
## Splitting the Problem
-->

## Chia vấn đề


<!--
Let us start with a simple computer vision problem and a slightly archaic network, e.g., with multiple layers of convolutions, pooling, and possibly a few dense layers in the end.
That is, let us start with a network that looks quite similar to LeNet :cite:`LeCun.Bottou.Bengio.ea.1998` or AlexNet :cite:`Krizhevsky.Sutskever.Hinton.2012`.
Given multiple GPUs (2 if it is a desktop server, 4 on a g4dn.12xlarge, 8 on an AWS p3.16xlarge, or 16 on a p2.16xlarge), 
we want to partition training in a manner as to achieve good speedup while simultaneously benefitting from simple and reproducible design choices.
Multiple GPUs, after all, increase both *memory* and *compute* ability.
In a nutshell, we have a number of choices, given a minibatch of training data that we want to classify.
-->

Ta hãy bắt đầu bằng một bài toán thị giác máy tính đơn giản cùng một kiến trúc mạng cổ xưa, với vài tầng tích chập, tầng gộp và có thể thêm vài tầng dày đặc ở cuối.
Như vậy, mạng này sẽ trông khá tương tự như LeNet :cite:`LeCun.Bottou.Bengio.ea.1998` hoặc AlexNet :cite:`Krizhevsky.Sutskever.Hinton.2012`.
Với nhiều GPU (máy chủ để bàn thường có 2, máy chủ g4dn.12xlarge thì có 4, AWS p3.16xlarge có 8, hoặc là 16 trên p2.16xlarge),
ta muốn phân chia việc huấn luyện theo một cách mà vừa tăng tốc độ lại tận dụng được lợi ích từ các thiết kế đơn giản và tái tạo được.
Nhiều GPU, sau tất cả, là để tăng cả *bộ nhớ* và năng lực *tính toán*.
Tóm lại, với một minibatch dữ liệu huấn luyện để phân loại, ta có một vài phương án phân tách khác nhau.

<!--
![Model parallelism in the original AlexNet design due to limited GPU memory.](../img/alexnet-original.svg)
-->

![Song song hoá mô hình trong thiết kế AlexNet gốc do giới hạn bộ nhớ trên GPU.](../img/alexnet-original.svg)
:label:`fig_alexnet_original`

<!-- ===================== Kết thúc dịch Phần 1 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 2 ===================== -->


<!--
* We could partition the network layers across multiple GPUs. 
That is, each GPU takes as input the data flowing into a particular layer,processes data across a number of subsequent layers and then sends the data to the next GPU. 
    * This allows us to process data with larger networks when compared to what a single GPU could handle. 
    * Memory footprint per GPU can be well controlled (it is a fraction of the total network footprint)
    * The interface between layers (and thus GPUs) requires tight synchronization. 
    This can be tricky, in particular if the computational workloads are not properly matched between layers. The problem is exacerbated for large numbers of GPUs.
    * The interface between layers requires large amounts of data transfer (activations, gradients). This may overwhelm the bandwidth of the GPU buses.
    * Compute intensive, yet sequential operations are nontrivial to partition. 
    See e.g., :cite:`Mirhoseini.Pham.Le.ea.2017` for a best effort in this regard. 
    It remains a difficult problem and it is unclear whether it is possible to achieve good (linear) scaling on nontrivial problems. 
    We do not recommend it unless there is excellent framework / OS support for chaining together multiple GPUs.
* We could split the work required by individual layers. 
For instance, rather than computing 64 channels on a single GPU we could split up the problem across 4 GPUs, each of which generate data for 16 channels. 
Likewise, for a dense layer we could split the number of output neurons. 
:numref:`fig_alexnet_original` illustrates this design. 
The figure is taken from :cite:`Krizhevsky.Sutskever.Hinton.2012` where this strategy was used to deal with GPUs that had a very small memory footprint (2GB at the time). 
    * This allows for good scaling in terms of computation, provided that the number of channels (or neurons) is not too small. 
    * Multiple GPUs can process increasingly larger networks since the memory available scales linearly.
    * We need a *very large* number of synchronization / barrier operations since each layer depends on the results from all other layers.
    * The amount of data that needs to be transferred is potentially even larger than when distributing layers across GPUs. We do not recommend this approach due to its bandwidth cost and complexity.
* Lastly we could partition data across multiple GPUs. This way all GPUs perform the same type of work, albeit on different observations. Gradients are aggregated between GPUs after each minibatch.
    * This is the simplest approach and it can be applied in any situation.
    * Adding more GPUs does not allow us to train larger models.
    * We only need to synchronize after each minibatch. That said, it is highly desirable to start exchanging gradients parameters already while others are still being computed. 
    * Large numbers of GPUs lead to very large minibatch sizes, thus reducing training efficiency.
-->

* Chúng ta có thể phân chia các tầng mạng cho các GPU.
Cụ thể, mỗi GPU lấy một luồng dữ liệu đưa vào từ một tầng xác định, xử lý dữ liệu truyền qua một số tầng kế tiếp nhau rồi gửi dữ liệu tới GPU kế tiếp. 
   * Điều này cho phép ta xử lý dữ liệu với các mạng lớn hơn khi so sánh với những gì một GPU có thể làm được.
   * Bộ nhớ bị chiếm dụng trên mỗi GPU có thể được kiểm soát dễ dàng (mỗi GPU sẽ chỉ chiếm một phần tổng dung lượng bộ nhớ cấp phát cho cả mạng).
   * Giao tiếp giữa các tầng (cũng như giữa các GPU) đòi hỏi tính đồng bộ chặt chẽ.
   Điều này có thể sẽ rất khó, đặc biệt nếu khối lượng tính toán không được phân chia hợp lý cho các tầng. 
   Vấn đề sẽ trở nên nghiêm trọng với một số lượng lớn GPU.
   * Giao tiếp giữa các tầng yêu cầu một lượng lớn việc truyền dữ liệu (các hàm kích hoạt, các gradient). Điều này có thể vượt quá mức băng thông các bus của GPU.
   * Các phép toán phức tạp nhưng tuần tự cũng sẽ tốn công sức nhất định để phân chia.
   :cite:`Mirhoseini.Pham.Le.ea.2017` là một cố gắng tốt nhất cho vấn đề này.
   Nó vẫn còn là một vấn đề khó và chưa rõ ràng liệu có thể mở rộng tốt (tuyến tính) cho các bài toán không tầm thường. 
   Chúng tôi không khuyến khích cách làm này trừ phi có một framework xuất sắc hay hệ điều hành hỗ trợ cho việc xâu chuỗi nhiều GPU lại với nhau.
* Chúng ta có thể phân chia công việc của các tầng đơn lẻ.
Chẳng hạn, thay vì tính toán 64 kênh trên một GPU, ta có thể tách công việc này cho 4 GPU, mỗi GPU sẽ sinh dữ liệu cho 16 kênh. 
Tương tự, với một tầng kết nối đặc ta có thể tách số neuron đầu ra.
:numref:`fig_alexnet_original` mô tả thiết kế kiểu này. 
Hình này lấy từ :cite:`Krizhevsky.Sutskever.Hinton.2012`, ở đây chiến lược này được sử dụng để làm việc với nhiều GPU mà có mức chiếm dụng bộ nhớ rất nhỏ (2GB ở thời điểm đó).
   * Điều này cho phép việc điều chỉnh kích thước tính toán tốt, với điều kiện là số kênh (hoặc số nơ-ron) không quá nhỏ.
   * Dùng nhiều GPU có thể xử lý nhiều mạng ngày một lớn hơn vì dung lượng bộ nhớ khả dụng cũng tăng tuyến tính.
   * Chúng ta cần một lượng *rất lớn* các phép toán đồng bộ / rào cản vì mỗi tầng tùy thuộc vào các kết quả từ tất cả các tầng khác.
   * Lượng dữ liệu cần được truyền thậm chí có thể lớn hơn khi chia các tầng cho các GPU.
   Chúng tôi không khuyến khích cách tiếp cận này do tính phức tạp và chiếm dụng băng thông của nó.
* Sau cùng, ta có thể phân chia dữ liệu cho nhiều GPU. 
Cách này cho phép tất cả GPU thực hiện cùng một công việc, mặc dù từ các vùng quan sát khác nhau. 
Các gradient được tổng hợp lại trên các GPU sau mỗi minibatch.
   * Đây là phương pháp đơn giản nhất và có thể sử dụng cho bất cứ tình huống nào.
   * Gắn thêm nhiều GPU không cho phép chúng ta huấn luyện mô hình lớn hơn.
   * Chúng ta chỉ cần đồng bộ hóa sau mỗi minibatch. 
   Như đã nói, ta có thể bắt đầu thực hiện trao đổi các gradient đã tính xong trong khi vẫn đang tính các gradient khác.
   * Số lượng GPU lớn dẫn tới kích thước minibatch rất lớn, do đó giảm hiệu quả huấn luyện.
   
<!--
![Parallelization on multiple GPUs. From left to right - original problem, network partitioning, layer partitioning, data parallelism.](../img/splitting.svg)
-->

![Song song hóa trên nhiều GPU. Từ trái sang phải - bài toán ban đầu, phân tách mạng, phân tách tầng, song song hóa dữ liệu](../img/splitting.svg)
:label:`fig_splitting`


<!--
By and large, data parallelism is the most convenient way to proceed, provided that we have access to GPUs with sufficiently large memory.
See also :cite:` Li.Andersen.Park.ea.2014` for a detailed description of partitioning for distributed training.
GPU memory used to be a problem in the early days of deep learning.
By now this issue has been resolved for all but the most unusual cases.
We focus on data parallelism in what follows.
-->

Nhìn chung việc song song hóa dữ liệu là cách thuận tiện nhất để xử lý, với điều kiện là ta có thể truy xuất tới các GPU với bộ nhớ đủ lớn.
Xem thêm :cite:` Li.Andersen.Park.ea.2014` để biết chi tiết cách phân chia cho việc huấn luyện phân tán.
Bộ nhớ GPU từng là một vấn đề trong những ngày đầu của học sâu.
Đến thời điểm này thì vấn đề đã được giải quyết cho hầu hết trừ một số trường hợp ít gặp nhất.
Chúng ta tập trung vào việc song song hóa dữ liệu ở phần kế tiếp sau.

<!-- ===================== Kết thúc dịch Phần 2 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 3 ===================== -->

<!--
## Data Parallelism
-->

## *dịch tiêu đề phía trên*


<!--
Assume that there are $k$ GPUs on a machine.
Given the model to be trained, each GPU will maintain a complete set of model parameters independently.
Training proceeds as follows (see :numref:`fig_data_parallel` for details on data parallel training on two GPUs):
-->

*dịch đoạn phía trên*


<!--
* In any iteration of training, given a random minibatch, we split the examples in the batch into $k$ portions and distribute them evenly across the GPUs. 
* Each GPU calculates loss and gradient of the model parameters based on the minibatch subset it was assigned and the model parameters it maintains. 
* The local gradients of each of the $k$ GPUs are aggregated to obtain the current minibatch stochastic gradient. 
* The aggregate gradient is re-distributed to each GPU. 
* Each GPU uses this minibatch stochastic gradient to update the complete set of model parameters that it maintains. 
-->

*dịch đoạn phía trên*

<!--
![Calculation of minibatch stochastic gradient using data parallelism and two GPUs. ](../img/data-parallel.svg)
-->

![*dịch chú thích ảnh phía trên*](../img/data-parallel.svg)
:label:`fig_data_parallel`


<!--
A comparison of different ways of parallelization on multiple GPUs is depicted in :numref:`fig_splitting`.
Note that in practice we *increase* the minibatch size $k$-fold when training on $k$ GPUs such that each GPU has the same amount of work to do as if we were training on a single GPU only.
On a 16 GPU server this can increase the minibatch size considerably and we may have to increase the learning rate accordingly.
Also note that :numref:`sec_batch_norm` needs to be adjusted (e.g., by keeping a separate batch norm coefficient per GPU). 
In what follows we will use :numref:`sec_lenet` as the toy network to illustrate multi-GPU training. As always we begin by importing the relevant packages and modules.
-->

*dịch đoạn phía trên*


```{.python .input  n=2}
%matplotlib inline
from d2l import mxnet as d2l
from mxnet import autograd, gluon, np, npx
npx.set_np()
```


<!-- ===================== Kết thúc dịch Phần 3 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 4 ===================== -->


<!--
## A Toy Network
-->

## *dịch tiêu đề phía trên*


<!--
We use LeNet as introduced in :numref:`sec_lenet`. We define it from scratch to illustrate parameter exchange and synchronization in detail.
-->

*dịch đoạn phía trên*


```{.python .input  n=10}
# Initialize model parameters
scale = 0.01
W1 = np.random.normal(scale=scale, size=(20, 1, 3, 3))
b1 = np.zeros(20)
W2 = np.random.normal(scale=scale, size=(50, 20, 5, 5))
b2 = np.zeros(50)
W3 = np.random.normal(scale=scale, size=(800, 128))
b3 = np.zeros(128)
W4 = np.random.normal(scale=scale, size=(128, 10))
b4 = np.zeros(10)
params = [W1, b1, W2, b2, W3, b3, W4, b4]

# Define the model
def lenet(X, params):
    h1_conv = npx.convolution(data=X, weight=params[0], bias=params[1],
                              kernel=(3, 3), num_filter=20)
    h1_activation = npx.relu(h1_conv)
    h1 = npx.pooling(data=h1_activation, pool_type='avg', kernel=(2, 2),
                     stride=(2, 2))
    h2_conv = npx.convolution(data=h1, weight=params[2], bias=params[3],
                              kernel=(5, 5), num_filter=50)
    h2_activation = npx.relu(h2_conv)
    h2 = npx.pooling(data=h2_activation, pool_type='avg', kernel=(2, 2),
                     stride=(2, 2))
    h2 = h2.reshape(h2.shape[0], -1)
    h3_linear = np.dot(h2, params[4]) + params[5]
    h3 = npx.relu(h3_linear)
    y_hat = np.dot(h3, params[6]) + params[7]
    return y_hat

# Cross-entropy loss function
loss = gluon.loss.SoftmaxCrossEntropyLoss()
```

<!-- ========================================= REVISE PHẦN 1 - KẾT THÚC ===================================-->

<!-- ========================================= REVISE PHẦN 2 - BẮT ĐẦU ===================================-->

<!--
## Data Synchronization
-->

## *dịch tiêu đề phía trên*


<!--
For efficient multi-GPU training we need two basic operations: firstly we need to have the ability to distribute a list of parameters to multiple devices and to attach gradients (`get_params`).
Without parameters it is impossible to evaluate the network on a GPU.
Secondly, we need the ability to sum parameters across multiple devices, i.e., we need an `allreduce` function.
-->

*dịch đoạn phía trên*


```{.python .input  n=12}
def get_params(params, ctx):
    new_params = [p.copyto(ctx) for p in params]
    for p in new_params:
        p.attach_grad()
    return new_params
```


<!--
Let us try it out by copying the model parameters of lenet to gpu(0).
-->

*dịch đoạn phía trên*


```{.python .input  n=13}
new_params = get_params(params, d2l.try_gpu(0))
print('b1 weight:', new_params[1])
print('b1 grad:', new_params[1].grad)
```


<!--
Since we didn't perform any computation yet, the gradient with regard to the bias weights is still $0$.
Now let us assume that we have a vector distributed across multiple GPUs.
The following allreduce function adds up all vectors and broadcasts the result back to all GPUs.
Note that for this to work we need to copy the data to the device accumulating the results.
-->

*dịch đoạn phía trên*


```{.python .input  n=14}
def allreduce(data):
    for i in range(1, len(data)):
        data[0][:] += data[i].copyto(data[0].ctx)
    for i in range(1, len(data)):
        data[0].copyto(data[i])
```


<!--
Let us test this by creating vectors with different values on different devices and aggregate them.
-->

*dịch đoạn phía trên*


```{.python .input  n=16}
data = [np.ones((1, 2), ctx=d2l.try_gpu(i)) * (i + 1) for i in range(2)]
print('before allreduce:\n', data[0], '\n', data[1])
allreduce(data)
print('after allreduce:\n', data[0], '\n', data[1])
```

<!-- ===================== Kết thúc dịch Phần 4 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 5 ===================== -->

<!--
## Distributing Data
-->

## *dịch tiêu đề phía trên*


<!--
We need a simple utility function to distribute a minibatch evenly across multiple GPUs.
For instance, on 2 GPUs we'd like to have half of the data to be copied to each of the GPUs.
Since it is more convenient and more concise, we use the built-in split and load function in Gluon (to try it out on a $4 \times 5$ matrix).
-->

*dịch đoạn phía trên*


```{.python .input  n=8}
data = np.arange(20).reshape(4, 5)
ctx = [npx.gpu(0), npx.gpu(1)]
split = gluon.utils.split_and_load(data, ctx)
print('input :', data)
print('load into', ctx)
print('output:', split)
```


<!--
For later reuse we define a `split_batch` function which splits both data and labels.
-->

*dịch đoạn phía trên*


```{.python .input  n=9}
#@save
def split_batch(X, y, ctx_list):
    """Split X and y into multiple devices specified by ctx."""
    assert X.shape[0] == y.shape[0]
    return (gluon.utils.split_and_load(X, ctx_list),
            gluon.utils.split_and_load(y, ctx_list))
```

<!--
## Training 
-->

## *dịch tiêu đề phía trên*


<!--
Now we can implement multi-GPU training on a single minibatch.
Its implementation is primarily based on the data parallelism approach described in this section.
We will use the auxiliary functions we just discussed, `allreduce` and `split_and_load`, to synchronize the data among multiple GPUs.
Note that we do not need to write any specific code to achieve parallelism.
Since the compute graph does not have any dependencies across devices within a minibatch, it is executed in parallel *automatically*.
-->

*dịch đoạn phía trên*


```{.python .input  n=10}
def train_batch(X, y, ctx_params, ctx_list, lr):
    X_shards, y_shards = split_batch(X, y, ctx_list)
    with autograd.record():  # Loss is calculated separately on each GPU
        losses = [loss(lenet(X_shard, ctx_W), y_shard)
                  for X_shard, y_shard, ctx_W in zip(
                      X_shards, y_shards, ctx_params)]
    for l in losses:  # Back Propagation is performed separately on each GPU
        l.backward()
    # Sum all gradients from each GPU and broadcast them to all GPUs
    for i in range(len(ctx_params[0])):
        allreduce([ctx_params[c][i].grad for c in range(len(ctx_list))])
    # The model parameters are updated separately on each GPU
    for param in ctx_params:
        d2l.sgd(param, lr, X.shape[0])  # Here, we use a full-size batch
```

<!--
Now, we can define the training function.
It is slightly different from the ones used in the previous chapters: we need to allocate the GPUs and copy all the model parameters to all devices.
Obviously each batch is processed using `train_batch` to deal with multiple GPUs.
For convenience (and conciseness of code) we compute the accuracy on a single GPU (this is *inefficient* since the other GPUs are idle).
-->

*dịch đoạn phía trên*


```{.python .input  n=61}
def train(num_gpus, batch_size, lr):
    train_iter, test_iter = d2l.load_data_fashion_mnist(batch_size)
    ctx_list = [d2l.try_gpu(i) for i in range(num_gpus)]
    # Copy model parameters to num_gpus GPUs
    ctx_params = [get_params(params, c) for c in ctx_list]
    # num_epochs, times, acces = 10, [], []
    num_epochs = 10
    animator = d2l.Animator('epoch', 'test acc', xlim=[1, num_epochs])
    timer = d2l.Timer()
    for epoch in range(num_epochs):
        timer.start()
        for X, y in train_iter:
            # Perform multi-GPU training for a single minibatch
            train_batch(X, y, ctx_params, ctx_list, lr)
            npx.waitall()
        timer.stop()
        # Verify the model on GPU 0
        animator.add(epoch + 1, (d2l.evaluate_accuracy_gpu(
            lambda x: lenet(x, ctx_params[0]), test_iter, ctx[0]),))
    print(f'test acc: {animator.Y[0][-1]:.2f}, {timer.avg():.1f} sec/epoch '
          f'on {str(ctx_list)}')
```

<!-- ===================== Kết thúc dịch Phần 5 ===================== -->

<!-- ===================== Bắt đầu dịch Phần 6 ===================== -->

<!--
## Experiment
-->

## *dịch tiêu đề phía trên*


<!--
Let us see how well this works on a single GPU. We use a batch size of 256 and a learning rate of 0.2.
-->

*dịch đoạn phía trên*


```{.python .input  n=62}
train(num_gpus=1, batch_size=256, lr=0.2)
```

<!--
By keeping the batch size and learning rate unchanged and changing the number of GPUs to 2, 
we can see that the improvement in test accuracy is roughly the same as in the results from the previous experiment.
In terms of the optimization algorithms, they are identical.
Unfortunately there is no meaningful speedup to be gained here: the model is simply too small; 
moreover we only have a small dataset, where our slightly unsophisticated approach to implementing multi-GPU training suffered from significant Python overhead.
We will encounter more complex models and more sophisticated ways of parallelization going forward. Let us see what happens nonetheless for MNIST.
-->

*dịch đoạn phía trên*


```{.python .input  n=13}
train(num_gpus=2, batch_size=256, lr=0.2)
```


## Tóm tắt

<!--
* There are multiple ways to split deep network training over multiple GPUs. 
We could split them between layers, across layers, or across data. 
The former two require tightly choreographed data transfers. 
Data parallelism is the simplest strategy.
* Data parallel training is straightforward. However, it increases the effective minibatch size to be efficient. 
* Data is split across multiple GPUs, each GPU executes its own forward and backward operation and subsequently gradients are aggregated and results broadcast back to the GPUs. 
* Large minibatches may require a slightly increased learning rate.
-->

*dịch đoạn phía trên*


## Bài tập

<!--
1. When training on multiple GPUs, change the minibatch size from $b$ to $k \cdot b$, i.e., scale it up by the number of GPUs.
2. Compare accuracy for different learning rates. How does it scale with the number of GPUs. 
3. Implement a more efficient allreduce that aggregates different parameters on different GPUs (why is this more efficient in the first place). 
4. Implement multi-GPU test accuracy computation.
-->

*dịch đoạn phía trên*


<!-- ===================== Kết thúc dịch Phần 6 ===================== -->
<!-- ========================================= REVISE PHẦN 2 - KẾT THÚC ===================================-->

## Thảo luận
* [Tiếng Anh](https://discuss.mxnet.io/t/2383)
* [Tiếng Việt](https://forum.machinelearningcoban.com/c/d2l)


## Những người thực hiện
Bản dịch trong trang này được thực hiện bởi:
<!--
Tác giả của mỗi Pull Request điền tên mình và tên những người review mà bạn thấy
hữu ích vào từng phần tương ứng. Mỗi dòng một tên, bắt đầu bằng dấu `*`.
Tên đầy đủ của các reviewer có thể được tìm thấy tại https://github.com/aivivn/d2l-vn/blob/master/docs/contributors_info.md
-->

* Đoàn Võ Duy Thanh
<!-- Phần 1 -->
* Lê Khắc Hồng Phúc
* Nguyễn Văn Cường

<!-- Phần 2 -->
* Nguyễn Mai Hoàng Long
* Nguyễn Văn Cường

<!-- Phần 3 -->
* 

<!-- Phần 4 -->
* 

<!-- Phần 5 -->
* 

<!-- Phần 6 -->
* 
