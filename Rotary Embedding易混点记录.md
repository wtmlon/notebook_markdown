# Rotary Embedding记录

公式：

<img width="532" alt="image" src="https://github.com/wtmlon/notebook_markdown/assets/37530985/102190cf-8b48-4102-b853-937ed9e26fb4">

- 这里的d指的是hidden_size那一维维度，这个公式指的是给位置m的一整个hidden_size长度的向量加上embedding
- 其中 $$\theta$$ 在这里是和hidden_size的序号强相关的，和seq_len序号没关系，定义如下：
<img width="157" alt="image" src="https://github.com/wtmlon/notebook_markdown/assets/37530985/92313cd3-9274-45e7-a29b-30bc0fd74dec">

- 其中i范围[0, (d/2 - 1)], 对应到代码中，这一块的计算如下：
```python
  self.inv_freq = 1.0 / (base ** (paddle.arange(0, dim, 2, dtype=paddle.float32) / dim))
```
- 在代码实现中，一般会先计算出 $$m\theta$$ 矩阵，这是一个[seq_len, d]的矩阵， 对应每一个seq idx的d维向量。
- 计算的代码如下，这里以千问大模型（llama-based）做示范,每一行都写了注释
```python
def update_rotary_pos_emb_cache(self, max_seq_len, offset=0, ntk_alpha=1.0):
        seqlen = max_seq_len + offset
        if seqlen > self._seq_len_cached or ntk_alpha != self._ntk_alpha_cached:
            base = self.base * ntk_alpha ** (self.dim / (self.dim - 2))
            # m * theta 矩阵计算， self.inv_freq: [d/2]
            self.inv_freq = 1.0 / (base ** (paddle.arange(0, self.dim, 2, dtype=paddle.float32) / self.dim))
            self._seq_len_cached = max(2 * seqlen, 16)
            self._ntk_alpha_cached = ntk_alpha
            # seq: [seq_len] ..[0, 1, 2, 3, 4, 5,...]...
            seq = paddle.arange(self._seq_len_cached)
            # 做外积运算，注意这里的顺序，第一个参数是seq, 第二个才是inv_freq
            # 这会生成一个[seq_len, d/2]矩阵，对应每个seq idx的d/2个 m * theta
            freqs = paddle.outer(seq.astype(self.inv_freq.dtype), self.inv_freq)
            # concat， 对于每个seq idx: m, 会有一个d维向量，注意，这里对应的hidden_size idx
            # 分布是[0, 2, 4,..., 2/d-1, 0, 2, 4, ..., 2/d-1]
            emb = paddle.concat([freqs, freqs], axis=-1)
            # 最后再将其unsqueeze，得到[bz, seq_len, num_heads, hidden]
            self._rotary_pos_emb_cache = emb.unsqueeze([0, 2])
```
## 嵌入环节
- 上面已经生成了 $$m*\theta$$，现在我们需要对query和key进行嵌入（因为只有他俩进行了内积，需要相对位置信息）
- 代码如下所示：
```python
def _rotate_half(x):
    # 注意，这里的维度从[bz, seq_len, n_head, d] == > [bz, seq_len, n_head, 2, d/2]
    # 这个做法是为了实现 x1, x0, x3, x2....这样的调换
    x = x.reshape(x.shape[:-1] + [2, -1])
    x1, x2 = x.unbind(axis=-2)
    return paddle.concat([-x2, x1], axis=-1)

# t: query, freqs: m * \theta
def apply_rotary_pos_emb(t, freqs):
    # 这里拿的是freqs最后一个维度，也就是d
    rot_dim = freqs.shape[-1]
    # 这里相当于复制了两份，不要被这个前后：绕晕
    t_, t_pass_ = t[..., :rot_dim], t[..., rot_dim:]
    t_ = t_.astype(paddle.float32)
    t_pass_ = t_pass_.astype(paddle.float32)
    # 这里实现的就是图一的计算：[bz, seq_len, n_head, d] * [bz, seq_len, n_head, d], 其中0，2维度broadcast
    t_ = (t_ * freqs.cos()) + (_rotate_half(t_) * freqs.sin())
    return paddle.concat([t_, t_pass_], axis=-1).astype(t.dtype)
```
