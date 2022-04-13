# proj128-kernel-livepatch-optimizations
Linux内核热补丁优化

### 项目描述与背景

Linux内核热补丁要达到的目的就是免重启对内核态代码进行打补丁，实现bug修复、性能优化、稳定性增强，甚至一些小特性的引入，通过分析整个整个热补丁的构建和运行原理，实际生产实践中发现存在一些优化点，包括但限于以下总结出来的3个点：

1. 内核热补丁间函数冲突检测增强  

	多个内核热补丁可能修改同一个函数，且这些补丁的加载顺序不同也会影响系统中正在运行的这些补丁的整体逻辑，造成潜在风险，目前内核中livepatch的实现是如果两个补丁之间存在冲突，那么对于冲突的函数而言，生效的是后加载的补丁，举个例子 


	* 补丁A的修改函数 a1、b1、c1，补丁B修改函数 a2, d2，且A先于B加载，那么系统中生效的补丁逻辑则为  a1、b1、c1、d2

	* 补丁A的修改函数 a1、b1、c1，补丁B修改函数 a2, d2，且A后于B加载，那么系统中生效的补丁逻辑则为  a2、b1、c1、d2

	备注：a1表示 a函数打补丁后的第个版本，同理 aN表示a函数打补丁后的第N个版本，可类推至其他函数，从上面可以不难分析出，a1搭配d2来跑可能会有问题，另外如果A和B加载的顺序不一致，那么对于机器集群来说，最终生效的补丁逻辑也存在不一致的情况。


2. 内核热补丁构建流程优化

	当前Linux内核热补丁构建是基于 [kpatch](https://github.com/dynup/kpatch) 来进行，用户提供一个patch文本文件，调用 kpatch-build 命令行工具即可完成内核热补丁的构建，对于开发者来说，快速构建、调试（试错）是一个强需求，社区的kpatch-build工具搭配主流Linux版本，在构建效率上有提高空间，可对整个构建流程进行优化，做到更高效

3. 内核热补丁一致性模型优化

	consistency model，这里翻译做一致性模型，其要达到的目的就是保证打补丁时整个系统的状态是一致的，具体来说，假设内核函数foo，打补丁前的原始函数为 foo\_old，打完补丁后的函数为 foo\_new，那么对于任务来说，它不能一会执行的是foo\_old，一会执行的是foo\_new，否则就会出现不可预期的行为。

	kernel livepatch的代码实现仅能达到 per-task consistency，也就是说对于单个任务来说，其在补丁生效前只能执行foo\_old的逻辑，补丁生效后只能执行 foo\_new 的逻辑，注意这里仅是per-task而不是整个系统的consistency，意味着同一个时刻，可能有的任务执行foo\_old的逻辑，有的任务执行替换后的foo\_new 的逻辑，这样在特定情况下就会有风险。

	[kpatch](https://github.com/dynup/kpatch)  社区的实现中，其进行打补丁时所用的一致性模型是利用 stop_machine把机器停住，然后检查每个任务栈， 看要补丁的函数有无在运行，若没有，则可安全打补丁； 若有，则退出，然后再继续恢复机器运行，其中保证整个系统的consistency，也就是从所有任务的视角来看，要补丁的函数要么没被替换，要么已被替换，不存在有的任务跑foo\_old逻辑，有的任务跑foo\_new逻辑的情况

	kpatch的stop_machine callback相比kernel livepatch更为安全，但是在多/众 核场景下的latency会更高，可能造成业务受损，需要一个既能保证稳定性又能兼顾性能的方案。



### 功能描述

#### 基础功能：
1. 内核热补丁间函数冲突检测增强

	* 补丁ko插入时判断其对内核函数的修改是否与当前running的其他补丁冲突，如果冲突，默认情况下报错退出
	* 支持确实要进行冲突覆盖的场景
	* 导出函数冲突信息到sys接口，方便用户态查询读取 
	
	涉及组件：kpatch的用户态部分、内核livepatch框架

2. 内核热补丁构建流程优化

	* 热补丁制作过程，大量时间耗费在第一次内核的编译上，可以考虑缓存起内核的二进制编译产物，这样就只需要做第二次增量的内核编译

环境要求：为避免内核版本不同造成差异，统一基于Linux 5.15进行代码开发

#### 扩展功能：

1. 内核热补丁一致性模型优化

	改进kernel的livepatch所用的consistency model，使其不再是per-task consistency，而是whole system consistency，同时保留其对于kpatch stop_machine方案的性能优势，要求最差性能情况下退化到stop machine的性能。

2. 内核热补丁构建流程优化

	二进制差异提取工具create-diff-object 性能优化

	学生可发挥自主创新，识别其他潜在的优化点并实现



### 所属赛道

2022全国大学生操作系统比赛的“OS功能挑战”赛道



### 参赛要求

- 以小组为单位参赛，最多三人一个小组，参赛对象为全国普通高等学校全日制在校本科生和在校研究生，参赛队员的在校本科生或在校研究生身份均以报名时为准
- 如学生参加了多个项目，参赛学生选择一个自己参加的项目参与评奖
- 请遵循“2022全国大学生操作系统比赛”的章程和技术方案要求



### 项目导师
启瑞 qirui.001@bytedance.com




### 难度

高


### 参考资料

项目相关参考资料

[kernel livepatch](https://www.kernel.org/doc/html/latest/livepatch/livepatch.html)  
[livepatch:consistency model](https://lwn.net/Articles/632582/)  
[kpatch](https://github.com/dynup/kpatch)   
[kpatch-build](https://github.com/dynup/kpatch/blob/master/kpatch-build/kpatch-build)  

### License

* [GPL-2.0](https://opensource.org/licenses/GPL-2.0)



## 预期目标

完成基础和扩展功能的开发，扩展功能部分若无法实现需提出想法和建议，并输出说明文档一篇。

**注意：下面的内容是建议内容，不要求必须全部完成。选择本项目的同学也可与导师联系，提出自己的新想法，如导师认可，可加入预期目标**

参考任务描述部分。
