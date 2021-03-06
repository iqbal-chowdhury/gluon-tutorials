# Dropout regularization with ``gluon``

In [the previous chapter](./mlp-dropout-scratch.ipynb), 
we introduced Dropout regularization, implementing the algorithm from scratch. 
As a reminder, Dropout is a regularization technique 
that zeroes out some fraction of the nodes during training. 
Then at test time, we use all of the nodes, but scale down their values,
essentially averaging the various dropped out nets. 
If you're approaching this chapter out of sequence,
and aren't sure how Dropout works, it's best to take a look at the implementation by hand
since ``gluon`` will manage the low-level details for us.

Dropout is a special kind of layer because it behaves differently 
when training and predicting. 
We've already seen how ``gluon`` can keep track of when to record vs not record the computation graph.
Since this is a ``gluon`` implementation chapter,
let's get intro the thick of things by importing our dependencies and some toy data.

```{.python .input}
from __future__ import print_function
import mxnet as mx
import numpy as np
from mxnet import nd, autograd
from mxnet import gluon
import sys
sys.path.append('..')
import utils

ctx = mx.cpu()

```

## The MNIST dataset

```{.python .input}
batch_size = 64
num_outputs = 10
train_data, test_data = utils.load_mnist(batch_size)
```

## Define the model

Now we can add Dropout following each of our hidden layers.

```{.python .input}
num_hidden = 256
net = gluon.nn.Sequential()
with net.name_scope():
    ###########################
    # Adding first hidden layer
    ###########################
    net.add(gluon.nn.Dense(num_hidden, activation="relu"))
    ###########################
    # Adding dropout with rate .5 to the first hidden layer
    ###########################
    net.add(gluon.nn.Dropout(.5))
    
    ###########################
    # Adding first hidden layer
    ###########################
    net.add(gluon.nn.Dense(num_hidden, activation="relu")) 
    ###########################
    # Adding dropout with rate .5 to the second hidden layer
    ###########################
    net.add(gluon.nn.Dropout(.5))
    
    ###########################
    # Adding the output layer
    ###########################
    net.add(gluon.nn.Dense(num_outputs))
```

## Parameter initialization

Now that we've got an MLP with dropout layers, let's register an initializer 
so we can play with some data.

```{.python .input}
net.collect_params().initialize(mx.init.Xavier(magnitude=2.24), ctx=ctx)
```

## Train mode and predict mode

Let's grab some data and pass it through the network.
To see what effect dropout is having on our predictions, 
it's instructive to pass the same example through our net multiple times.

```{.python .input}
for x, _ in train_data:
    x = x.as_in_context(ctx)
    break
print(net(x[0:1]))
print(net(x[0:1]))
```

Note that we got the exact same answer on both forward passes through the net!
That's because by, default, ``mxnet`` assumes that we are in predict mode. 
We can explicitly invoke this scope by placing code within a ``with autograd.predict_mode():`` block.

```{.python .input}
with autograd.predict_mode():
    print(net(x[0:1]))
    print(net(x[0:1]))
```

Unless something's gone horribly wrong, you should see the same result as before. 
We can also run the code in *train mode*.
This tells MXNet to run our Blocks as they would run during training.

```{.python .input}
with autograd.train_mode():
    print(net(x[0:1]))
    print(net(x[0:1]))
```

## Accessing ``is_training()`` status

You might wonder, how precisely do the Blocks determine 
whether they should run in train mode or predict mode?
Basically, autograd maintains a Boolean state 
that can be accessed via ``autograd.is_training()``. 
By default this falue is ``False`` in the global scope.
This way if someone just wants to make predictions and 
doesn't know anything about training models, everything will just work.
When we enter a ``train_mode()`` block, 
we create a scope in which ``is_training()`` returns ``True``.

```{.python .input}
with autograd.predict_mode():
    print(autograd.is_training())
    
with autograd.train_mode():
    print(autograd.is_training())
```

## Integration with ``autograd.record``

When we train neural network models,
we nearly always enter ``record()`` blocks.
The purpose of ``record()`` is to build the computational graph.
And the purpose of ``train`` is to indicate that we are training our model.
These two are highly correlated but should not be confused.
For example, when we generate adversarial examples (a topic we'll investigate later)
we may want to record, but for the model to behave as in predict mode.
On the other hand, sometimes, even when we're not recording,
we still want to evaluate the model's training behavior.

A problem then arises. Since ``record()`` and ``train_mode()``
are distinct, how do we avoid having to declare two scopes every time we train the model?

```{.python .input}
##########################
#  Writing this every time could get cumbersome
##########################
with autograd.record():
    with autograd.train_mode():
        yhat = net(x)
```

To make our lives a little easier, record() takes one argument, ``train_mode``,
which has a default value of True.
So when we turn on autograd, this by default turns on train_mode
(``with autograd.record()`` is equivalent to
``with autograd.record(train_mode=True):``).
To change this default behavior
(as when generating adversarial examples),
we can optionally call record via
(``with autograd.record(train_mode=False):``).

## Softmax cross-entropy loss

```{.python .input}
softmax_cross_entropy = gluon.loss.SoftmaxCrossEntropyLoss()
```

## Optimizer

```{.python .input}
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': .1})
```

## Evaluation metric

```{.python .input}
def evaluate_accuracy(data_iterator, net):
    acc = mx.metric.Accuracy()
    for i, (data, label) in enumerate(data_iterator):
        data = data.as_in_context(ctx).reshape((-1, 784))
        label = label.as_in_context(ctx)
        output = net(data)
        predictions = nd.argmax(output, axis=1)
        acc.update(preds=predictions, labels=label)
    return acc.get()[1]
```

## Training loop

```{.python .input  n=17}
epochs = 10
smoothing_constant = .01

for e in range(epochs):
    for i, (data, label) in enumerate(train_data):
        data = data.as_in_context(ctx).reshape((-1, 784))
        label = label.as_in_context(ctx)
        with autograd.record():
            output = net(data)
            loss = softmax_cross_entropy(output, label)
            loss.backward()
        trainer.step(data.shape[0])

        ##########################
        #  Keep a moving average of the losses
        ##########################
        curr_loss = nd.mean(loss).asscalar()
        moving_loss = (curr_loss if ((i == 0) and (e == 0)) 
                       else (1 - smoothing_constant) * moving_loss + (smoothing_constant) * curr_loss)

    test_accuracy = evaluate_accuracy(test_data, net)
    train_accuracy = evaluate_accuracy(train_data, net)
    print("Epoch %s. Loss: %s, Train_acc %s, Test_acc %s" %
          (e, moving_loss, train_accuracy, test_accuracy))
```

## Conclusion

Now let's take a look at how to build convolutional neural networks.

## Next
[Introduction to ``gluon.Block`` and ``gluon.nn.Sequential``](../chapter03_deep-neural-networks/plumbing.ipynb)

For whinges or inquiries, [open an issue on  GitHub.](https://github.com/zackchase/mxnet-the-straight-dope)
