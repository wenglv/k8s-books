#### 小结

1. 伴随着业务场景复杂度的提高，单体架构应用弊端显现，微服务的思想逐步盛行，微服务架构带来诸多便捷的同时，也带来了很多问题，最主要的是多个微服务的服务治理（服务发现、调用、负载均衡、跟踪）
2. 为了解决服务治理问题，出现了微服务框架（Dubbo、Spring Cloud等）
3. Spring Cloud是一个大的生态，基于Java语言封装了一系列的工具，方便业务直接使用来解决上述服务治理相关的问题
4. Spring Cloud Netflix 体系下提供了eureka、ribbon、feign、hystrix、zuul等工具结合spring cloud sleuth合zipkin实现服务跟踪
5. SpringBoot是微服务的开发框架，通过maven与Spring Cloud生态中的组件集成，极大方便了java应用程序的交付

![](images\arch.png)



问题：

1. 无论是Dubbo还是SpringCloud，均属于Java语言体系下的产物，跨语言没法共用，同时，通过走了一遍内部集成的过程，可以清楚的发现，服务治理过程中，各模块的集成，均需要对原始业务逻辑形成侵入。
2. 在kubernetes的生态下，已经与生俱来带了很多好用的功能（自动服务发现与负载均衡）
3. 服务治理的根本其实是网络节点通信的治理，因此，以istio为代表的第二代服务治理平台开始逐步兴起

