## 介绍
spring-kafka是一个脚手架，方便我们使用kafka，可以通过注解的方式创建kafka的监听实例，方便编程。其初始化过程主要包括两大核心即MessageListenerContainer与KafkaListenerContainerFactory内。在初始化过程中主要是完成创建接口GenericMessageListener的实例、创建consumer，向kafka集群注册消费者等步骤。

## MessageListenerContainer

继承与实现关系：

![](2024-11-19-17-34-02.png)

ConcurrentMessageListenerContainer与KafkaMessageListenerContainer是两个MessageListenerContainer的实现，一个是多线程的（ConcurrentMessageListenerContainer），一个是单线程的（KafkaMessageListenerContainer）。

## KafkaListenerContainerFactory

继承与实现关系：
![](2024-11-19-17-34-51.png)

## 初始化过程
利用spring的BeanPostProcessor机制触发KafkaListenerAnnotationBeanPostProcessor对bean的注解的扫描，当扫描到KafkaListener或KafkaListeners注解时，利用KafkaListenerEndpointRegistrar和KafkaListenerEndpointRegistry发起Container的创建。Container的创建使用了工厂模式，由KafkaListenerContainerFactory工厂创建Container。



```plantuml

@startuml

participant BeanPostProcessor
participant KafkaListenerAnnotationBeanPostProcessor
BeanPostProcessor -> KafkaListenerAnnotationBeanPostProcessor : postProcessAfterInitialization

activate KafkaListenerAnnotationBeanPostProcessor
KafkaListenerAnnotationBeanPostProcessor -> KafkaListenerAnnotationBeanPostProcessor : 获取bean中方法级别的KafkaListener注解

    alt 方法级别的注解为空 
        KafkaListenerAnnotationBeanPostProcessor-> KafkaListenerAnnotationBeanPostProcessor : 打印日志
    else 
        loop 所有的带KafkaListener的方法
            loop 方法上的KafkaListener注解
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 处理方法代理，获取被代理的方法
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 创建MethodKafkaListenerEndpoint
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 注册listenerScope
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 获取KafkaListener注解上的topic列表
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 获取KafkaListener注解上\n的TopicPartitionOffset 
                note right : TopicPartitionOffset为\ntopic，分区，分区的\noffset配置信息
                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 获取
                alt 处理RetryTopic 成功 （此处省略）
                    
                else 处理普通的topic
                    KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 将bean、KafkaListener上的信息\n经过提取、转换后填充到\nMethodKafkaListenerEndpoint中\n(processKafkaListenerAnnotation)
                    KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 获取KafkaListenerContainerFactory \n（通过KafkaListener上的配置中Spring中获取bean）
                    KafkaListenerAnnotationBeanPostProcessor->KafkaListenerEndpointRegistrar : 注册MethodKafkaListenerEndpoint
                end

                KafkaListenerAnnotationBeanPostProcessor->KafkaListenerAnnotationBeanPostProcessor : 注销listenerScope
            end
        end
    end
deactivate KafkaListenerAnnotationBeanPostProcessor

@enduml
```

### annotation注册过程

```plantuml
@startuml

participant KafkaListenerAnnotationBeanPostProcessor
participant KafkaListenerEndpointRegistrar
participant KafkaListenerEndpointRegistry
participant KafkaListenerContainerFactory
KafkaListenerAnnotationBeanPostProcessor -> KafkaListenerEndpointRegistrar : 注册EndPoint registerEndpoint

KafkaListenerEndpointRegistrar -> KafkaListenerEndpointRegistry : 注册ListenerContainer registerListenerContainer

KafkaListenerEndpointRegistry -> KafkaListenerContainerFactory : 创建ListenerContainer
activate KafkaListenerContainerFactory #FFBBBB
    alt topicPartition 配置不为空
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 用topicPartition创建 ContainerProperties
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 获取ConsumerFactory
        note right : 通常我们在kafka配置的时候，\n自己创建并塞入ContainerFactory的属性
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 创建ConcurrentMessageListenerContainer实例
    else topics 不为空
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 用topics创建 ContainerProperties
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 获取ConsumerFactory
        note right : 通常我们在kafka配置的时候，\n自己创建并塞入ContainerFactory的属性
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 创建ConcurrentMessageListenerContainer实例
    else 处理topicPattern模式
    KafkaListenerContainerFactory->KafkaListenerContainerFactory : 用topicPattern创建 ContainerProperties
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 获取ConsumerFactory
        note right : 通常我们在kafka配置的时候，\n自己创建并塞入ContainerFactory的属性
        KafkaListenerContainerFactory->KafkaListenerContainerFactory : 创建ConcurrentMessageListenerContainer实例
    end
    KafkaListenerContainerFactory -> KafkaListenerContainerFactory : 将endpoint的id设置为当前Container的beanName
    alt endpoint是继承自AbstractKafkaListenerEndpoint
    KafkaListenerContainerFactory -> KafkaListenerContainerFactory : 将container的部分属性设置到endpoint中，如：batchListener，ackDiscarded等
    end 

    activate KafkaListenerContainerFactory #AAAAAA
    KafkaListenerContainerFactory -> KafkaListenerContainerFactory : 初始化container
    KafkaListenerContainerFactory -> ConcurrentMessageListenerContainer : 填充container的属性值。
    KafkaListenerContainerFactory -> KafkaListenerContainerFactory : 设置个性化参数
    KafkaListenerContainerFactory -> KafkaListenerEndpointRegistry : 返回container实例
    deactivate
deactivate KafkaListenerContainerFactory
KafkaListenerEndpointRegistry -> KafkaListenerEndpointRegistry : 处理Container实例的启动时机
KafkaListenerEndpointRegistry -> KafkaListenerEndpointRegistry : 将Container实例放入listenerContainers缓存中
KafkaListenerEndpointRegistry -> KafkaListenerEndpointRegistry : 处理endpoint的group，获取ContainerGroup的bean，并将container塞入ContainerGroup中。
alt 处理自动启动的Container
    KafkaListenerEndpointRegistry-> ConcurrentMessageListenerContainer : 启动
end
@enduml

```


```plantuml
@startuml
participant ConcurrentMessageListenerContainer
participant KafkaMessageListenerContainer
participant ListenerConsumer

ConcurrentMessageListenerContainer -> ConcurrentMessageListenerContainer : doStart

ConcurrentMessageListenerContainer -> ConcurrentMessageListenerContainer : 校验topics（里面会调用AdminClient去校验）
ConcurrentMessageListenerContainer -> ConcurrentMessageListenerContainer : 获取topicpartition、containerProperties

ConcurrentMessageListenerContainer -> ConcurrentMessageListenerContainer : 设置running标志位

loop 循环concurrency次数
    ConcurrentMessageListenerContainer->KafkaMessageListenerContainer : 创建KafkaMessageListenerContainer实例
    ConcurrentMessageListenerContainer->ConcurrentMessageListenerContainer : 配置 ConcurrentMessageListenerContainer 实例属性
    ConcurrentMessageListenerContainer-> KafkaMessageListenerContainer : 启动
    KafkaMessageListenerContainer -> KafkaMessageListenerContainer : 校验ack模式
    KafkaMessageListenerContainer -> KafkaMessageListenerContainer : 创建consumer任务执行器
    KafkaMessageListenerContainer -> KafkaMessageListenerContainer : 计算listener的类型（ListenerType）
    KafkaMessageListenerContainer -> ListenerConsumer : 创建ListenerConsumer
    activate ListenerConsumer
    ListenerConsumer-> ListenerConsumer: 校验groupInstance（group.instance.id配置）
    ListenerConsumer -> ListenerConsumer: 依据properties计算autocommit值
    ListenerConsumer -> DefaultKafkaConsumerFactory:  创建 Consumer
    activate DefaultKafkaConsumerFactory
        alt 是否需要调整参数
            DefaultKafkaConsumerFactory -> DefaultKafkaConsumerFactory : 调整groupId、clientId参数    
        end
        DefaultKafkaConsumerFactory -> DefaultKafkaConsumerFactory : 创建原始KafkaConsumer(kafka-client)
        alt listeners中元素的个数大于1
            DefaultKafkaConsumerFactory -> DefaultKafkaConsumerFactory : 为原始KafkaConsumer创建代理
            DefaultKafkaConsumerFactory -> DefaultKafkaConsumerFactory :将KafkaConsumer设置到listener中
        end
        DefaultKafkaConsumerFactory -> DefaultKafkaConsumerFactory : 处理postProcessors步骤
        note right : 此处是为了扩展出来给使用方处理\n原始KafkaConsumer用的，可以注\n入自己的后处理步骤逻辑
        DefaultKafkaConsumerFactory -> ListenerConsumer : 返回创建好的Consumer实例
    deactivate DefaultKafkaConsumerFactory
    
    ListenerConsumer -> ListenerConsumer : 从Consumer中获取并计算clientId
    ListenerConsumer -> ListenerConsumer : 处理事务模板
    ListenerConsumer -> ListenerConsumer : 设置genericListener属性，消费者的消息处理逻辑实体
    ListenerConsumer -> ListenerConsumer : 计算commitCurrentOnAssignment属性
    alt 批量消费Listener
        ListenerConsumer -> ListenerConsumer : 设置批量消费属性
    else
        ListenerConsumer -> ListenerConsumer : 设置单个消费属性
    end

    ListenerConsumer -> ListenerConsumer : 设置listenerType
    ListenerConsumer -> ListenerConsumer : 计算isConsumerAwareListener
    ListenerConsumer -> ListenerConsumer : 获取ErrorHandler
    ListenerConsumer -> ListenerConsumer : 初始化taskScheduler，用于执行定时任务
    ListenerConsumer -> ListenerConsumer : 初始化monitorTask，监控定时任务
    ListenerConsumer -> ListenerConsumer : 计算checkNullKeyForExceptions、checkNullValueForExceptions
    ListenerConsumer -> ListenerConsumer : 计算syncCommitTimeout
    ListenerConsumer -> ListenerConsumer : 计算maxPollInterval（max.poll.interval.ms）
    ListenerConsumer -> ListenerConsumer : 处理监控micrometerHolder
    ListenerConsumer -> ListenerConsumer : 处理deliveryAttemptAware
    ListenerConsumer -> ListenerConsumer : 处理subBatchPerPartition
    ListenerConsumer -> KafkaMessageListenerContainer :返回ListenerConsumer实例
    deactivate ListenerConsumer
    KafkaMessageListenerContainer -> KafkaMessageListenerContainer : 设置running标志位
    KafkaMessageListenerContainer -> SimpleAsyncTaskExecutor : 向执行器提交ListenerConsumer实例，并执行
    activate SimpleAsyncTaskExecutor
    SimpleAsyncTaskExecutor -> ListenerConsumer : 执行run方法
    deactivate SimpleAsyncTaskExecutor
    KafkaMessageListenerContainer -> KafkaMessageListenerContainer : 当前线程等待一定的时间并结束初始化过程
KafkaMessageListenerContainer -> ConcurrentMessageListenerContainer : 启动结束
ConcurrentMessageListenerContainer ->ConcurrentMessageListenerContainer : 将当前container实例加入到自己的list中
end
@enduml


```