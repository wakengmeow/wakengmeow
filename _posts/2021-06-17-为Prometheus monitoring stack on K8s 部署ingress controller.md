

最近有个项目需要在k8s上部署prometheus，alertmanager和grafana。其中有个需求是用统一的入口访问三个服务的web接口从而尽可能少的暴露k8s node ip和port。
要求比较简单，但是可能因为有一段时间没有正儿八经地做项目，实现过程中还是踩坑而不自知：

# 需求分析 #
  K8s支持以loadbalancer或者nodeport的形式对外暴露服务，不过目的是安全和之后的运维简便所以这类native的方案不被考虑了
  
  ingress controller的方案作为原生的成熟的方案是第一选择，接下来就是选哪个了，其实想试试istio，考虑到企业级需求和运维team的skillset还是选了nginx
  
  那接下来就是选k8s的nginx ingress controller还是nginx的ingress controller，以一个monitoring stack的web接口来讲需要的功能就是代理，2个产品都是满足的
  这个url有2者的功能点的区别：https://codechina.csdn.net/mirrors/nginxinc/kubernetes-ingress/-/blob/master/docs/nginx-ingress-controllers.md
  
  一开始是打算用nginx版本，主要因为它家提供plus版本的企业支持，对企业级比较适合，而k8s版本社区报的issue有点多。然而。。。。。我竟然没能第一时间把nginx版本部署成功，说明这个有点难用，所以就改成k8s版本了，毕竟issue多说明用的也多


# 系统架构 #

![](2021061701.JPG)



# 实施 #

## 步骤 ##

  1. 为方便测试，先部署一个prometheus的monitoring stack(https://github.com/wakengmeow/monitoring），记得把prometheus，alertmanager和grafana部署成clusterip的service
  2. 部署nginx的ingress controller，（https://github.com/wakengmeow/monitoring），这个ingress controller其实就是在k8s上的一个pod里运行的nginx server，所以这个nginx也必须以service的形式发布出去，因为这里nginx ingress controller service是提供对外服务的，所以service type必须是LoadBalancer或者nodeport。nodeport不考虑，因为需求就是不希望使用k8s的nodeip来访问
  3. 为每一个服务部署一个ingress（https://github.com/wakengmeow/monitoring），这个ingress的用处就是把每个服务的路由规则注入到ingress controller这个nginx server上来避免每次手动修改nginx.conf

## yml上主要注意点 ##
  
  * ingress controller是可以监听整个k8s cluster的ingress的，也就是跨namespace的。所以如果希望只路由特定namespace的服务，可以用 k8s的annotations “kubernetes.io/ingress.class“来实现

  在ingress controller 部署deployment的时候要显式地定义如下：
  ```  
    containers:
     - name: nginx-ingress-controller
       image: quay.io/kubernetes-ingress-controller/ nginx-ingress-controller:0.30.0
        args:
          - /nginx-ingress-controller
          - --ingress-class=《your ingress controller class name》
   ```
     

  在ingress 部署时候要加入相应的配置：
  ```
         apiVersion: networking.k8s.io/v1  
          kind: Ingress 
          metadata: 
              name: ingress-monitoring
              namespace: testing
          annotations:
              kubernetes.io/ingress.class:《your ingress controller class name》 
    
  ```

## 接下来说坑 ##
    
我们想做的：
  * 通过 http://loadbalancerip/prometheus 访问prometheus的web       
  * 通过 http://loadbalancerip/alertmanager 访问alertmanager的web  
  * 通过 http://loadbalancerip/grafana 访问grafana的web
    
看上去也确实简单，狗狗小冰小度回答都是用 k8s的annotations 来实现，类似如下这种： 
    
`  nginx.ingress.kubernetes.io/use-regex: "true" `
`  nginx.ingress.kubernetes.io/rewrite-target: /$1 `
      

同时,用多path，正则来redirect也是nginx的概念，实现上应该straightforward。事实是一整个工作日都没有调通，作为一个配置类的task的用时超出太多了。


## 解决方法 ##
    
  1. 在k8s github发现issue https://github.com/kubernetes/ingress-nginx/issues/6437，确认了新旧版本的变化 
        
      在issue conversation里找到 ingress conformance testing 的test case https://aledbf.github.io/ingress-conformance-sample/features/path-rules.html
      据此，去掉之前所有的rewarite啦正则啦，改用最简单的path 路由

  2. 之后测试显示路由没有问题了，但是backend service的loading有问题，这题大家都会， 用--web.external-url，社区这方面的issue也有不少  
      https://github.com/prometheus/prometheus/issues/4925。 但是这个目前只有prometheus和alertmanager支持。Grafana则需要修改grafana.ini.
      从时间和以后维护来讲，都不值得用configmap去改ini，而且考虑到grafana web 是最频繁被使用的，所以最后改成以下的实现来躲开ini文件的修改：  

        * 通过 http://loadbalancerip/prometheus 访问prometheus的web       
        * 通过 http://loadbalancerip/alertmanager 访问alertmanager的web  
        * 通过 http://loadbalancerip/ 访问grafana的web


# 总结 #
  过去用开源技术栈有问题都是第一时间去社区查issue，这次失误了
