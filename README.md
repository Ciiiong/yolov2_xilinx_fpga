# yolov2_xilinx_fpga
A demo for accelerating YOLOv2 in xilinx's fpga PYNQ  
You can follow the step: HLS -> VIVADO -> PYNQ or just jump to PYNQ
Every repo has some steps to help further evaluate or study.  

# Design and Optimization of YOLOv2 Accelerator Based on FPGA  
According to the analysis of the YOLOv2 network, most layers are serially processed, except for the routing layer. The routing layer can be implemented by setting a specific address in advance.   
From an accelerator perspective, the work required is to interact with memory in order (reading memory data, processing data, and then writing back memory data). Since the amount of data input and output is very large, loop tiling technique is always applied to reuse data and reduce memory access times, which tiles the convolution loop R, C, M, N to Tr, Tc, Tm ,Tn[8].  
The overall architecture of the accelerator is shown below:  
![overview](https://github.com/dhm2013724/yolov2_xilinx_fpga/blob/master/overview.png)

Similar to [4,5,8], the accelerator has two AXI4 master interfaces and one AXI4-Lite slave interface. AXI-Lite slave interface is responsible for reading and writing control, data and status register sets. The input feature maps and weights are read concurrently by two master interfaces, and the output feature maps are written back simultaneously through write channel.   
The Data Scatter module is designed to generate the corresponding write address and distribute the data read from the DRAM to the on-chip buffers. The Data Gather module is designed to generate the DRAM write-back address and write the data in the output buffer back to the DRAM. The other red modules are responsible for the processing of the convolutional layer (Conv and Leaky ReLU), the maximum pooling layer (Pool) and the reorg layer (Reorg).  
1292/5000
与[4,5,8]类似，加速器有两个AXI4主接口和一个AXI4-Lite从接口。 AXI-Lite从接口负责读写控制，数据和状态寄存器组。输入要素图和权重由两个主接口同时读取，输出要素图通过写入通道同时写回。Data Scatter模块用于生成相应的写地址，并将从DRAM读取的数据分配到片上缓冲区。 Data Gather模块用于生成DRAM回写地址，并将输出缓冲区中的数据写回DRAM。其他红色模块负责处理卷积层（Conv和Leaky ReLU），最大池层（Pool）和reorg层（Reorg）。

## Weight Arrangement   
The effective FPGA bandwidth goes up with the increase of burst length and finally flattens out above some burst length threshold[7]. The data tiling technique usually results in a discontinuous DRAM access for the row-major data layout in DRAM. To reduce the number of memory accesses and increase the effective memory bandwidth, we arrange the kernel weights for an entire tile to a continuous block to ensure a high utilization of the bandwidth of external memory [3].  
有效的FPGA带宽随着突发长度的增加而上升，最终在一些突发长度阈值之上变平[7]。数据切片技术通常导致DRAM中行主数据布局的不连续DRAM访问。为了减少内存访问次数并增加有效内存带宽，我们将整个磁贴的内核权重排列为连续块，以确保高效利用外部存储器的带宽[3]。
## Parallel Convolution Engine  
The acceleration strategy of convolutional layer is similar to [5][6], which utilizes input and output parallelism to accelerate the computation. By designing multiple parallel multiplication units and add trees to achieve input parallelism (Tn parallelism) and output parallelism (Tm parallelism) in convolution calculation. The Tm*Tn multiplication units are calculated in parallel. The add trees of Log2 (Tn) depth are accumulated by pipeline, and generate the partial sums.  
并行卷积引擎
卷积层的加速策略类似于[5] [6]，它利用输入和输出并行来加速计算。 通过设计多个并行乘法单元并添加树来实现卷积计算中的输入并行性（Tn并行性）和输出并行性（Tm并行性）。 Tm * Tn乘法单元是并行计算的。 Log2（Tn）深度的添加树由管道累积，并生成部分和。
## Ping-Pong operation  
Similar to [8], the design implements ping-pong buffers to overlap the delay of reading input feature maps and weights, writing output feature maps and calculation, which greatly improves the dynamic utilization of the computing engines.  
与[8]类似，该设计实现了乒乓缓冲区，以重叠读取输入要素图和权重的延迟，编写输出要素图和计算，这极大地提高了计算引擎的动态利用率。
# Evaulate  
Experiments show that floating point addition in HLS requires three DSP resources, floating point multiplication requires two DSPs; fixed point 16-bit multiplication requires one DSP, and fixed-point 16-bit addition can be implemented only using LUT. After placing and routing, resource consumptions of fixed-16 (Tn=2, Tm=32, Tr=26, Tc=26) are shown as follows:     

  |  Resource     |  DSP      | BRAM      | LUT        |  FF        | Freq   |
  |  -----        |   -----   | -----     | -----      |  -----     | -----  |
  |Fixed-16(n2m32)| 106(48%)  | 100(72%)  | 27495(52%) | 30118(28%) |	130MHz |

According to the current design, DSP and BRAM are more expensive. The cost of DSP can be further reduced (there are many bit-width redundant multiplications), and the BRAM cost can be reduced. (As Shen [1] said, BRAM allocates an exponential size of 2 in HLS. Actually, many BRAMs are redundant. ).  
The performance comparison in the two cases is shown in the following table:  
  
| Performance              |        |
|  -----                   | -----  |
|CNN models	               |YOLO v2 |
|Board                     | PYNQ   |                
|Clock(MHz)		             |    130 |
|Precision		             |Fixed-16|
|Power (W)		             |   2.71 |
|Operations (GOP)		       |29.47   |
|Performance(GOP/s)		     |11.39   |
|Power Efficiency(GOP/s/W) |	4.20  |

# Result  
![image1](https://github.com/dhm2013724/yolov2_xilinx_fpga/blob/master/pynq/result.jpg)

# References:  
[1] Maximizing CNN Accelerator Efficiency Through Resource Partitioning  
[2] PLACID: A Platform for FPGA-Based Accelerator Creation for DCNNs  
[3] Going Deeper with Embedded FPGA Platform for Convolutional Neural Network  
[4] DianNao A Small-Footprint High-Throughput Accelerator for Ubiquitous Machine-Learning  
[5] An Automatic RTL Compiler for High-Throughput FPGA Implementation of Diverse Deep Convolutional Neural Networks  
[6] A Dynamic Multi-precision Fixed-Point Data Quantization Strategy for Convolutional Neural Network  
[7] Caffeine: Towards Uniformed Representation and Acceleration for Deep Convolutional Neural Networks  
[8] Optimizing FPGA-based Accelerator Design for Deep Convolutional Neural Networks  


  
  


