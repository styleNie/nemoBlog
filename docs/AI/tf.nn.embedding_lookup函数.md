embedding_lookup函数的实现：    
tf函数接口实现
```
import tensorflow as tf
import numpy as np

input_ids = tf.placeholder(dtype=tf.int32, shape=[None])

embedding = tf.Variable(np.identity(5, dtype=np.int32))
input_embedding = tf.nn.embedding_lookup(embedding, input_ids)

sess = tf.InteractiveSession()
sess.run(tf.global_variables_initializer())
print(embedding.eval())
print(sess.run(input_embedding, feed_dict={input_ids:[1, 2, 3, 0, 3, 2, 1]}))
```

矩阵乘法版本的实现    
```
import tensorflow as tf
import numpy as np

input_ids = tf.placeholder(dtype=tf.int32, shape=[None])

embedding = tf.Variable(np.identity(5, dtype=np.int32))
flat_input_ids = tf.reshape(input_ids, [-1])
one_hot_input_ids = tf.one_hot(flat_input_ids, depth=5,dtype=tf.int32)
output = tf.matmul(one_hot_input_ids, embedding)

print(sess.run(output, feed_dict={input_ids:[1, 2, 3, 0, 3, 2, 1]}))
```