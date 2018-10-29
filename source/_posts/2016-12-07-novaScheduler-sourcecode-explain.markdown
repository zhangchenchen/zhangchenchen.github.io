---
layout: post
title:  "openstack系列--novaScheduler 源代码解读"
date:   2016-12-07 16:00:25
tags: openstack
---

## 目录结构：

[novaScheduler 介绍](#A)


[以nova boot 为例解读代码 ](#B)

[novascheduler 整体讲解](#C)

[Filtering 讲解 ](#D)

[Weighting 讲解 ](#E)

<a name="A"></a>

## novaScheduler 介绍

 - 关于nova不再赘述，之前文章由介绍。
 - nova-scheduler的主要作用就是就是根据各种规则为虚拟机选择一个合适的主机。
 - 同nova-volume被单独剥离为cinder项目一样，社区也致力于将nova-scheduler剥离为gantt,作为一个通用的调度服务。
 - 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## 以nova boot 为例解读代码

 - 由之前解读novaConductor时，我们把牵扯到调度novascheduler的部分拿过来：

```python
    def _schedule_instances(self, context, request_spec, filter_properties):
        scheduler_utils.setup_instance_group(context, request_spec,
                                             filter_properties)
        # TODO(sbauza): Hydrate here the object until we modify the
        # scheduler.utils methods to directly use the RequestSpec object
        spec_obj = objects.RequestSpec.from_primitives(
            context, request_spec, filter_properties)
        #调用scheduler_client,获取可用host列表
        hosts = self.scheduler_client.select_destinations(context, spec_obj)
        return hosts
```
找到scheduler_client.select_destinations函数

```python
    #有一个装饰器，当发送失败的时候retry
    @utils.retry_select_destinations
    def select_destinations(self, context, spec_obj):
        return self.queryclient.select_destinations(context, spec_obj)
```

最终调用rpcapi中的select_destinations方法：

```python
    def select_destinations(self, ctxt, spec_obj):
        version = '4.3'
        msg_args = {'spec_obj': spec_obj}
        if not self.client.can_send_version(version):
            del msg_args['spec_obj']
            msg_args['request_spec'] = spec_obj.to_legacy_request_spec_dict()
            msg_args['filter_properties'
                     ] = spec_obj.to_legacy_filter_properties_dict()
            version = '4.0'
        #生成 RPCClient 
        cctxt = self.client.prepare(version=version)
        #发送rpc call 同步请求
        return cctxt.call(ctxt, 'select_destinations', **msg_args)
```

 - 接下来看下novascheduler 接收到'select_destinations' 的rpc请求之后的动作，之前说Novaconductor的时候说到接受请求执行的函数都在manager文件中。

```python
    @messaging.expected_exceptions(exception.NoValidHost)
    def select_destinations(self, ctxt,
                            request_spec=None, filter_properties=None,
                            spec_obj=_sentinel):
        """Returns destinations(s) best suited for this RequestSpec.

        The result should be a list of dicts with 'host', 'nodename' and
        'limits' as keys.
        """
        if spec_obj is self._sentinel:
            spec_obj = objects.RequestSpec.from_primitives(ctxt,
                                                           request_spec,
                                                           filter_properties)
        #调用driver中的方法，该driver即典型的stevedore中的driver插件导入
        dests = self.driver.select_destinations(ctxt, spec_obj)
        return jsonutils.to_primitive(dests)
```

看下driver的相关代码：

```python
    def __init__(self, scheduler_driver=None, *args, **kwargs):
        if not scheduler_driver:
            scheduler_driver = CONF.scheduler.driver
        #根据nova配置文件选取scheduler_driver，利用stevedore的driver载入方式载入setup.cfg中对应scheduler_driver。
        self.driver = driver.DriverManager(
                "nova.scheduler.driver",
                scheduler_driver,
                invoke_on_load=True).driver
        super(SchedulerManager, self).__init__(service_name='scheduler',
                                               *args, **kwargs)
```

一般setup.cfg 中对应的scheduler.driver有：

```python
nova.scheduler.driver =
    filter_scheduler = nova.scheduler.filter_scheduler:FilterScheduler
    caching_scheduler = nova.scheduler.caching_scheduler:CachingScheduler
    chance_scheduler = nova.scheduler.chance:ChanceScheduler
    fake_scheduler = nova.tests.unit.scheduler.fakes:FakeScheduler
```

默认获取到的scheduler.driver是filter_scheduler，关于stevedore的用法，[check this](http://yansu.org/2013/06/09/learn-python-stevedore-module-in-detail.html)

找到filter_scheduler的select_destinations方法，该方法做一些参数的处理，然后调用_schedule方法：

```python
  def _schedule(self, context, spec_obj):
        """Returns a list of hosts that meet the required specs,
        ordered by their fitness.
        """
        elevated = context.elevated()

        config_options = self._get_configuration_options()
        #获取所有的host列表
        hosts = self._get_all_host_states(elevated)

        selected_hosts = []
        num_instances = spec_obj.num_instances
        # NOTE(sbauza): Adding one field for any out-of-tree need
        spec_obj.config_options = config_options
        #此处num_instances 猜测是新建虚拟机的个数，因为 boot 命令中有一个参数新建多个虚拟机
        for num in range(num_instances):
            # Filter local hosts based on requirements ...
            #根据过滤条件筛选符合filter 的host，具体filter代码位于filters/目录下
            hosts = self.host_manager.get_filtered_hosts(hosts,
                    spec_obj, index=num)
            if not hosts:
                # Can't get any more locally.
                break

            LOG.debug("Filtered %(hosts)s", {'hosts': hosts})
            #根据weight条件将host列表按权重排序，具体weight代码位于weights/目录下
            weighed_hosts = self.host_manager.get_weighed_hosts(hosts,
                    spec_obj)

            LOG.debug("Weighed %(hosts)s", {'hosts': weighed_hosts})

            host_subset_size = max(1, CONF.filter_scheduler.host_subset_size)
            if host_subset_size < len(weighed_hosts):
                weighed_hosts = weighed_hosts[0:host_subset_size]
            chosen_host = random.choice(weighed_hosts)

            LOG.debug("Selected host: %(host)s", {'host': chosen_host})
            selected_hosts.append(chosen_host)

            # Now consume the resources so the filter/weights
            # will change for the next instance.
            chosen_host.obj.consume_from_request(spec_obj)
            if spec_obj.instance_group is not None:
                spec_obj.instance_group.hosts.append(chosen_host.obj.host)
                # hosts has to be not part of the updates when saving
                spec_obj.instance_group.obj_reset_changes(['hosts'])
        return selected_hosts

```

 - 先看下获取所有的host列表的代码_get_all_host_states，直接调用host_manager.get_all_host_states，

```python
    def get_all_host_states(self, context):
        """Returns a list of HostStates that represents all the hosts
        the HostManager knows about. Also, each of the consumable resources
        in HostState are pre-populated and adjusted based on data in the db.
        """
        #从数据库获取nova-computer服务的service列表
        service_refs = {service.host: service
                        for service in objects.ServiceList.get_by_binary(
                            context, 'nova-compute', include_disabled=True)}
        # Get resource usage across the available compute nodes:
        #从数据库获取所有计算节点的资源使用情况
        compute_nodes = objects.ComputeNodeList.get_all(context)
        seen_nodes = set()
        #更新计算节点的资源使用情况
        for compute in compute_nodes:
            service = service_refs.get(compute.host)

            if not service:
                LOG.warning(_LW(
                    "No compute service record found for host %(host)s"),
                    {'host': compute.host})
                continue
            host = compute.host
            node = compute.hypervisor_hostname
            state_key = (host, node)
            host_state = self.host_state_map.get(state_key)
            if not host_state:
                #获取host_state,如果本身维护的数据更新时间比数据库的早晚，就不用从数据库获取了。
                host_state = self.host_state_cls(host, node, compute=compute)
                self.host_state_map[state_key] = host_state
            # We force to update the aggregates info each time a new request
            # comes in, because some changes on the aggregates could have been
            # happening after setting this field for the first time
            host_state.update(compute,
                              dict(service),
                              self._get_aggregates_info(host),
                              self._get_instance_info(context, compute))

            seen_nodes.add(state_key)

        # remove compute nodes from host_state_map if they are not active
        #去掉状态为inactive的node
        dead_nodes = set(self.host_state_map.keys()) - seen_nodes
        for state_key in dead_nodes:
            host, node = state_key
            LOG.info(_LI("Removing dead compute node %(host)s:%(node)s "
                         "from scheduler"), {'host': host, 'node': node})
            del self.host_state_map[state_key]
        #返回一个node的迭代器
        return six.itervalues(self.host_state_map)
```

 - 再看下根据filter过滤host的函数host_manager.get_filtered_hosts

```python
def get_filtered_hosts(self, hosts, spec_obj, index=0):
        """Filter hosts and return only ones passing all filters."""
        #若filter_properties中指定了ignore_hosts,则排除相应host
        def _strip_ignore_hosts(host_map, hosts_to_ignore):
            .................................
        ##若filter_properties中指定了forced_hosts,则排除不在forced_hosts中的host
        def _match_forced_hosts(host_map, hosts_to_force):
            .................................
        #若filter_properties中指定了forced_nodes,则排除不在forced_nodes中的host
        def _match_forced_nodes(host_map, nodes_to_force):
            .................................
        #若指定了requested_destination，则在指定requested_destination中filter host
        def _get_hosts_matching_request(hosts, requested_destination):
            ..................................

        ignore_hosts = spec_obj.ignore_hosts or []
        force_hosts = spec_obj.force_hosts or []
        force_nodes = spec_obj.force_nodes or []
        requested_node = spec_obj.requested_destination

        if requested_node is not None:
            # NOTE(sbauza): Reduce a potentially long set of hosts as much as
            # possible to any requested destination nodes before passing the
            # list to the filters
            hosts = _get_hosts_matching_request(hosts, requested_node)
        if ignore_hosts or force_hosts or force_nodes:
            # NOTE(deva): we can't assume "host" is unique because
            #             one host may have many nodes.
            name_to_cls_map = {(x.host, x.nodename): x for x in hosts}
            if ignore_hosts:
                _strip_ignore_hosts(name_to_cls_map, ignore_hosts)
                if not name_to_cls_map:
                    return []
            # NOTE(deva): allow force_hosts and force_nodes independently
            if force_hosts:
                _match_forced_hosts(name_to_cls_map, force_hosts)
            if force_nodes:
                _match_forced_nodes(name_to_cls_map, force_nodes)
            if force_hosts or force_nodes:
                # NOTE(deva): Skip filters when forcing host or node
                if name_to_cls_map:
                    return name_to_cls_map.values()
                else:
                    return []
            #获得经过上述条件后满足条件的host列表
            hosts = six.itervalues(name_to_cls_map)
        #调用配置文件中规定的filter对host列表进行过滤，筛选host
        return self.filter_handler.get_filtered_objects(self.enabled_filters,
                hosts, spec_obj, index)
```

 - 再看下根据weight条件将host列表按权重排序host_manager.get_weighed_hosts方法，直接调用host_manager.get_weighed_hosts：

```python
    def get_weighed_objects(self, weighers, obj_list, weighing_properties):
        """Return a sorted (descending), normalized list of WeighedObjects."""

        #获取WeighedObject对象 ，就是对host做一层抽象
        weighed_objs = [self.object_class(obj, 0.0) for obj in obj_list]
        #如果只有一个WeighedObject，直接返回
        if len(weighed_objs) <= 1:
            return weighed_objs
        #逐一调用各权重过滤器。
        for weigher in weighers:
            weights = weigher.weigh_objects(weighed_objs, weighing_properties)

            # Normalize the weights
            weights = normalize(weights,
                                minval=weigher.minval,
                                maxval=weigher.maxval)
            #累加权重值
            for i, weight in enumerate(weights):
                obj = weighed_objs[i]
                obj.weight += weigher.weight_multiplier() * weight
        #根据权重值降序排列并返回
        return sorted(weighed_objs, key=lambda x: x.weight, reverse=True)
```

到这里，Novascheduler的分析就基本结束了。

<a name="C"></a>

## novascheduler 整体讲解

 - novascheduler 源码目录架构：

 ![novascheduler code](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2016-12-08-openstack-nova-scheduler-code.png)

 - nova scheduler实现了三种调度器：ChanceScheduler(随机调度器)，FilterScheduler(过滤调度器)，CachingScheduler(缓存调度器，在FilterScheduler的基础上将主机资源缓存在本地，通过后台定时任务从数据库更新缓存信息)。如果我们想自己实现定制的调度器，需要实现的接口在nova.scheduler.driver.Scheduler,继承类SchedulerDriver.

 - 一般来说，scheduler的过程大致是这样的：从nova.scheduler.rpcapi.SchedulerAPI发送RPC请求到nova.scheduler.manager.SchedulerManager;从SchedulerManager到调度器（类SchedulerDriver）;从SchedulerDriver到Filters;从Filters到权重计算排序Weights。


<a name="D"></a>

## Filtering 讲解


 - 所有的filter都在/nova/scheduler/filters目录，继承自nova.scheduler.filters.BaseHostFilter。实现一个host_passes()函数。

```python
class DiskFilter(filters.BaseHostFilter):
    """Disk Filter with over subscription flag."""

    def _get_disk_allocation_ratio(self, host_state, spec_obj):
        return host_state.disk_allocation_ratio
    #重点在实现该函数
    def host_passes(self, host_state, spec_obj):
        """Filter based on disk usage."""
        requested_disk = (1024 * (spec_obj.root_gb +
                                  spec_obj.ephemeral_gb) +
                          spec_obj.swap)

        free_disk_mb = host_state.free_disk_mb
        total_usable_disk_mb = host_state.total_usable_disk_gb * 1024

        # Do not allow an instance to overcommit against itself, only against
        # other instances.  In other words, if there isn't room for even just
        # this one instance in total_usable_disk space, consider the host full.
        if total_usable_disk_mb < requested_disk:
            LOG.debug("%(host_state)s does not have %(requested_disk)s "
                      "MB usable disk space before overcommit, it only "
                      "has %(physical_disk_size)s MB.",
                      {'host_state': host_state,
                       'requested_disk': requested_disk,
                       'physical_disk_size':
                           total_usable_disk_mb})
            return False

        ...............................
        disk_gb_limit = disk_mb_limit / 1024
        host_state.limits['disk_gb'] = disk_gb_limit
        return True
```




<a name="E"></a>

## Weighting 讲解


 - 所有weighter位于nova/scheduler/weights目录，继承自weights.BaseWeighter;

```python
class RAMWeigher(weights.BaseHostWeigher):
    #可以设置maxval ,minval指明权重的最大最小值。
    minval = 0
   #权重的系数，最终排序时需要将各种weighter得到的权重乘上对应的系数，有多个weighter时才有意义，可以通过配置选项ram_weight_multiplier配置，默认为1.0
    def weight_multiplier(self):
        """Override the weight multiplier."""
        return CONF.filter_scheduler.ram_weight_multiplier
    #计算权重值，对于RAMWeighter直接返回可用内存大小即可
    def _weigh_object(self, host_state, weight_properties):
        """Higher weights win.  We want spreading to be the default."""
        return host_state.free_ram_mb
```











## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[Openstack liberty源码分析 之 云主机的启动过程2](http://blog.csdn.net/lzw06061139/article/details/51491891)

[学习Python动态扩展包stevedore](http://yansu.org/2013/06/09/learn-python-stevedore-module-in-detail.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
