# LLM学习框架

<show-structure depth="3"/>

## 1. LLM工作原理

《Build a Large Language Model (From Scratch)》通过一步步亲手编写代码，深入理解并掌握大语言模型（LLM）的内部运作机制。

[GitHub 源码](https://github.com/rasbt/LLMs-from-scratch)

<procedure>
<img src="build-a-large-language-model.jpeg"  thumbnail="true"/>
</procedure>

## 2. LLM物理组网

[Alibaba HPN- A Data Center Network for Large Language Model  Training](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final878-acmpaginated.pdf)

[Rail-only: A Low-Cost High-Performance Network for Training LLMs with Trillion Parameters](https://arxiv.org/pdf/2307.12169)

[Optimized Network Architectures for Training Large Language Models With Billions of Parameters](https://people.csail.mit.edu/ghobadi/papers/rail_llm_hotnets_2023.pdf)

[RailX-A Flexible, Scalable, and Low-Cost Network Architecture for Hyper-Scale LLM Training Systems](https://arxiv.org/abs/2507.18889)

## 3. LLM推理优化

### 3.1 关键性能指标

- **Time to First Token (TTFT)**: This measures the responsiveness of the system, defined as the time elapsed from when a user's request arrives to when the first output token is generated. It is primarily a function of the scheduling delay (time spent waiting in a queue) and the prefill phase latency. Low TTFT is essential for interactive applications like chatbots to feel responsive.
- **Time Per Output Token (TPOT) / Inter-Token Latency (ITL)**: This measures the speed of generation after the first token, defined as the average time to generate each subsequent token. It is determined by the decode phase latency. TPOT dictates the fluency of the output stream, which is critical for applications generating long responses.
- **Total Latency**: This is the total time required to generate a complete response for a single user. It can be calculated as: **Latency = TTFT + (TPOT × (number of output tokens − 1))**.
- **Throughput**: This measures the overall capacity and efficiency of the inference server. It is typically defined as the total number of output tokens generated per second across all concurrent users and requests. Maximizing throughput is the key to lowering the operational cost per token served.

### 3.2 LLM推理特征及优化

[P/D-Serve- Serving Disaggregated Large Language Model at Scale](https://arxiv.org/abs/2408.08147)

[Characterizing Communication Patterns in Distributed Large Language Model Inference](https://www.arxiv.org/abs/2507.14392)

## 4. LLM集合通信

[Doubling all2all Performance with NVIDIA Collective Communication Library 2.12](https://developer.nvidia.com/blog/doubling-all2all-performance-with-nvidia-collective-communication-library-2-12/)

## 5. LLM高可用

[Fault Tolerant Llama: training with 2000 synthetic failures every ~15 seconds and no checkpoints on Crusoe L40S](https://pytorch.org/blog/fault-tolerant-llama-training-with-2000-synthetic-failures-every-15-seconds-and-no-checkpoints-on-crusoe-l40s/)

[Revisiting Reliability in Large-Scale Machine Learning Research Clusters](https://arxiv.org/abs/2410.21680)

