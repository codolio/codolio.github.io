---
layout: post
title:  "Disruptor处理流程分析"
date:   2019-01-30 12:02:35 +0800
categories: disruptor
---

# 1. 概述
Disruptor是LMAX提供的一个高性能线程间消息库（A High Performance Inter-Thread Messaging Library），它起源于LMAX在并发、高性能及非阻塞算法等方面的研究成果，日渐成熟，而今构成了LMAX交易基础架构的核心组成部分。<br/>

# 2. 基本概念与基础组件
在开始分析Disruptor的使用方式及处理流程之前，我们先来了解一下Disruptor中的基本概念与基础组件。在了解了这些基本概念与基础组件的基础上，我们再通过一个具体的示例来详细分析这些基础组件是如何结合起来并构成Disruptor的完整处理流程的。
#### （1）Event
由生产者（Producer）向消费者（Consumer或称Event Processor）传递的数据基本单元。

#### （2）Ring Buffer
环形储存，每个Ring Buffer中包含指定数量（数量需为2的整数次幂）的可复用条目（Entry），这些条目会在创建Ring Buffer时预先分配。每个条目中包含的数据代表一个正在生产者和消费者之间传递的事件（Event）。

#### （3）Sequence及Sequencer
每个Sequence对象中包含一个递增的数值，用于跟踪Disruptor中一个特定组件（Ring Buffer或者Event Processor）的处理进度。在Disruptor中，每一个Event Processor对象均持有一个Sequence对象。每个Ring Buffer中持有一个Sequencer对象，其中包含一个游标（cursor，Sequence对象）及一组Gating Sequences。Sequencer中的游标用于追踪Ring Buffer的生产进度，Gating Sequences用于追踪每个Event Processor的消费进度。<br/>
常用的Sequencer有以下两种：SingleProducerSequencer及MultiProducerSequencer。构建Ring Buffer时会根君传入的生产者类型来创建相应类型的Sequencer。

#### （4）Sequence Barrier 
Sequence Barrier由Ring Buffer中所持有的Sequencer对象生成，其中包含一个来自该Sequencer的游标（cursor、Sequence对象）以及一组所依赖的Sequence。使用Sequence Barrier可以构建多组有前后依赖关系的消费者（使用Sequence Barrier中的Dependent Sequences）。<br/>
Sequence Barrier中持有一个Wait Strategy对象，用于实现在Ring Buffer无可用事件时的等待逻辑。

#### （5）Event Handler及Event Processor 
Event Handler作为Disruptor中的事件处理接口，需要由用户实现，用于完成消费者具体的业务处理逻辑。每个Event Processor持有一个Sequence对象，用于记录该消费者的消费进度，同时持有一个Sequence Barrier对象，用于控制消费者的消费进度。Disruptor启动时会启动每个Event Processor处理主循环，依次处理可用的事件。常用的Event Processor为BatchEventProcessor。

#### （6）WaitStrategy
在Ring Buffer中有就绪的事件前，消费者的等待策略。常用的等待策略有：BlockingWaitStrategy、BusySpinWaitStrategy等。

#### （7）Producer
生产者，调用Disruptor来将待处理的事件写入Ring Buffer。

# 3. 整合
以下为一个常见示例，我们使用该简单示例先从整体上说明Disruptor中的各个组件是如何结合起来。<br/>
<pre>
// Event Factory，用于初始化Ring Buffer时生成Entry（此处为SampleItem），并填充至Ring Buffer中。
EventFactory&lt;SampleItem&gt; eventFactory = new SampleItemEventFactory();

Disruptor&lt;SampleItem&gt; sampleDisruptor = new Disruptor<>(
  eventFactory, // Event Factory，见上述。
  1024, // Ring Buffer的大小，为2的整数次幂，此处设置为1024。
  new DefaultThreadFactory(), // Thread Factory，用于构造消费者线程。
  // 生产者类型，有Single和Multi两种，此处为Multi，表示生产者为多线程。
  // 此处需要根据实际情况设置，Disruptor会根据此处设置的类型来构建对应的Sequencer。
  // 如果设置错误，那么可能会造成线程安全问题。
  ProducerType.MULTI, 
  new BlockingWaitStrategy() // 在Ring Buffer中无可用事件时，消费者的等待策略。
); 

// 自定义Event Handler，Disruptor会将此处设置的Event Handler封装成对应的Event Processor，默认使用BatchEventProcessor。
// Event Handler可以为多个，此处设置为两个：SampleEventHandler1和SampleEventHandler2。
sampleDisruptor.handleEventsWith(new EventHandler[] { new SampleEventHandler1(), new SampleEventHandler2() });

// Disruptor准备就绪，开启Disruptor。开启后，用户可以向Disruptor中发布（Publish）事件，消费者消费事件。
sampleDisruptor.start();
</pre>

# 4. 处理流程分析
从上述流程看，构建Disruptor主要分为三大步：
* 创建Disruptor，并初始化Ring Buffer等组件。
* 构建并设置Event Handler；
* 启动Disrutpor。<br/>

处理流程时序图如下（基于Disruptor 3.4.2版本）：<br/>
![Disruptor处理流程](/assets/images/disruptor_sequence.png)

说明如下：<br/>
（1）在创建Disruptor阶段，主要工作为根据传入的参数（例如：生产者类型、Ring Buffer大小及WaitStrategy等）来创建Sequencer实例及Ring Buffer实例。上述示例中选取的生产者类型为Multi，所以在此过程中会创建MultiProducerSequencer。MultiProducerSequencer及用于单线程的SingleProducerSequencer均继承自AbstractSequencer，AbstractSequencer中包含一个游标（cursor，Sequence对象），用于追踪Ring Buffer的生产者进度，同时包含一组Sequence（Gating Sequences），用于追踪消费者进度。创建Ring Buffer对象时会传入该Sequencer对象。

（2）在创建并设置Event Handler过程中，主要完成如下工作：
* 使用Ring Buffer构建Sequence Barrier，并用于后续待创建的每个Event Processor。
* 对每一个Event Handler构建一个BatchEventProcessor对象，并收集每个Event Processor所持有的Sequence对象。
* 将上述步骤中收集到的每个Event Processor的Sequence对象添加至Ring Buffer的Gating Sequences。

（3）启动Disruptor，对于上述步骤中创建的每一个Event Processor调用其start方法，构建线程，启动Event Processor主循环。

# 5. 小结
以上内容简单描述了Disruptor的基本概念、组件及基本的使用方式，并阐述了主要处理流程。有关生产者及消费者更为详细的处理流程在后续文档中分析。

# 6. 参考文档
* https://github.com/LMAX-Exchange/disruptor/wiki/Introduction
