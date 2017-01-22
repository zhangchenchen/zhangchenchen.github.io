---
layout: post
title:  "openstack系列--novaCompute 源代码解读"
date:   2016-12-09 08:56:25
tags: openstack
---

## 目录结构：

[novacompute 介绍](#A)

[以nova boot 为例解读代码 ](#B)

[novacompute 整体讲解](#C)



<a name="A"></a>

## novacompute 介绍

 - novacomputer 完成的主要任务是虚拟机的生命周期管理。
 - 目前支持的虚拟化包括libvirt (KVM, Xen, LXC and more), Hyper-V, VMware, XenServer 
 - 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## 以nova boot 为例解读代码

 - 首先我们先把之前解读novaconductor时发送rpc消息给novacompute的代码拿过来nova/conductor/manager.py.ComputeTaskManager.build_instances：

```python
            #发送异步消息给nova-compute，完成instance的boot
            self.compute_rpcapi.build_and_run_instance(context,
                    instance=instance, host=host['host'], image=image,
                    request_spec=request_spec,
                    filter_properties=local_filter_props,
                    admin_password=admin_password,
                    injected_files=injected_files,
                    requested_networks=requested_networks,
                    security_groups=security_groups,
                    block_device_mapping=bdms, node=host['nodename'],
                    limits=host['limits'])
```

可以看到novaconductor 引入novacomputer中的rpcapi.py文件，调用ComputeAPI.build_and_run_instance方法，看下该方法：

```python
    def build_and_run_instance(self, ctxt, instance, host, image, request_spec,
            filter_properties, admin_password=None, injected_files=None,
            requested_networks=None, security_groups=None,
            block_device_mapping=None, node=None, limits=None):

        version = '4.0'
        #创建一个指定host的RPCClient
        cctxt = self.router.by_host(ctxt, host).prepare(
                server=host, version=version)
        #发送异步rpc请求
        cctxt.cast(ctxt, 'build_and_run_instance', instance=instance,
                image=image, request_spec=request_spec,
                filter_properties=filter_properties,
                admin_password=admin_password,
                injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping, node=node,
                limits=limits)
```

 - 由之前分析可知，rpc的请求被服务端接收后，真正执行函数的地方在manager.py这个文件里，找到执行函数：

```python
    @wrap_exception() #捕获所有exception的装饰器
    @reverts_task_state #一旦build instance failure 会还原 task-state
    @wrap_instance_fault #捕获所有instance相关的异常
    def build_and_run_instance(self, context, instance, image, request_spec,
                     filter_properties, admin_password=None,
                     injected_files=None, requested_networks=None,
                     security_groups=None, block_device_mapping=None,
                     node=None, limits=None):
        #保证某一instance的任务是同步执行的
        @utils.synchronized(instance.uuid)
        def _locked_do_build_and_run_instance(*args, **kwargs):
            # NOTE(danms): We grab the semaphore with the instance uuid
            # locked because we could wait in line to build this instance
            # for a while and we want to make sure that nothing else tries
            # to do anything with this instance while we wait.
            with self._build_semaphore:
                #最终调用该函数
                self._do_build_and_run_instance(*args, **kwargs)

        # NOTE(danms): We spawn here to return the RPC worker thread back to
        # the pool. Since what follows could take a really long time, we don't
        # want to tie up RPC workers.
        #利用eventlet新建一个线程执行_locked_do_build_and_run_instance函数
        utils.spawn_n(_locked_do_build_and_run_instance,
                      context, instance, image, request_spec,
                      filter_properties, admin_password, injected_files,
                      requested_networks, security_groups,
                      block_device_mapping, node, limits)
```

看下最后执行的_do_build_and_run_instance函数：

```python
    @hooks.add_hook('build_instance')
    @wrap_exception()
    @reverts_task_state
    @wrap_instance_event(prefix='compute')
    @wrap_instance_fault
    def _do_build_and_run_instance(self, context, instance, image,
            request_spec, filter_properties, admin_password, injected_files,
            requested_networks, security_groups, block_device_mapping,
            node=None, limits=None):

        try:
            LOG.debug('Starting instance...', instance=instance)
            instance.vm_state = vm_states.BUILDING
            instance.task_state = None
            #更新task_stat
            instance.save(expected_task_state=
                    (task_states.SCHEDULING, None))
        except exception.InstanceNotFound:
            msg = 'Instance disappeared before build.'
            LOG.debug(msg, instance=instance)
            return build_results.FAILED
        except exception.UnexpectedTaskStateError as e:
            LOG.debug(e.format_message(), instance=instance)
            return build_results.FAILED

        # b64 decode the files to inject:
        decoded_files = self._decode_files(injected_files)

        if limits is None:
            limits = {}

        if node is None:
            #由具体实现的driver（如libvirtDriver）获取nodename,因为hypervisor不一样，有的hypervisor其实就一个node(如libvirt),而有的则是多个node.
            node = self.driver.get_available_nodes(refresh=True)[0]
            LOG.debug('No node specified, defaulting to %s', node,
                      instance=instance)

        try:
            #设定一个时间检测，调用_build_and_run_instance
            with timeutils.StopWatch() as timer:
                self._build_and_run_instance(context, instance, image,
                        decoded_files, admin_password, requested_networks,
                        security_groups, block_device_mapping, node, limits,
                        filter_properties)
            LOG.info(_LI('Took %0.2f seconds to build instance.'),
                     timer.elapsed(), instance=instance)
            return build_results.ACTIVE
        except exception.RescheduledException as e:
            retry = filter_properties.get('retry')
            if not retry:
                # no retry information, do not reschedule.
                LOG.debug("Retry info not present, will not reschedule",
                    instance=instance)
                self._cleanup_allocated_networks(context, instance,
                    requested_networks)
                compute_utils.add_instance_fault_from_exc(context,
                        instance, e, sys.exc_info(),
                        fault_message=e.kwargs['reason'])
                self._nil_ou  t_instance_obj_host_and_node(instance)
                self._set_instance_obj_error_state(context, instance,
                                                   clean_task_state=True)
                return build_results.FAILED
               ....................................
```

接着往下看self._build_and_run_instance函数：

```python
def _build_and_run_instance(self, context, instance, image, injected_files,
            admin_password, requested_networks, security_groups,
            block_device_mapping, node, limits, filter_properties):
        #image是一个包含镜像信息的字典，‘name’是镜像的名字
        image_name = image.get('name')
        self._notify_about_instance_usage(context, instance, 'create.start',
                extra_usage_info={'image_name': image_name})

        self._check_device_tagging(requested_networks, block_device_mapping)

        try:
            # #获取/创建ResourceTracker实例，为后续的资源申请做准备（此处涉及的resource Tracker Claim机制后面后详细讲）
            rt = self._get_resource_tracker(node)
            with rt.instance_claim(context, instance, limits):
                # NOTE(russellb) It's important that this validation be done
                # *after* the resource tracker instance claim, as that is where
                # the host is set on the instance.
                #group_policy包含两种affinity 和 anti-affinity，在创建server_group的时候指定。
                #创建虚拟机时指定server_group,会根据group_policy倾向于创建在同一台host或不同host
                self._validate_instance_group_policy(context, instance,
                        filter_properties)
                image_meta = objects.ImageMeta.from_dict(image)
                #为云主机申请网络资源，完成块设备验证及映射
                with self._build_resources(context, instance,
                        requested_networks, security_groups, image_meta,
                        block_device_mapping) as resources:
                    instance.vm_state = vm_states.BUILDING
                    instance.task_state = task_states.SPAWNING
                    # NOTE(JoshNang) This also saves the changes to the
                    # instance from _allocate_network_async, as they aren't
                    # saved in that function to prevent races.
                    #更新实例状态
                    instance.save(expected_task_state=
                            task_states.BLOCK_DEVICE_MAPPING)
                    block_device_info = resources['block_device_info']
                    network_info = resources['network_info']
                    LOG.debug('Start spawning the instance on the hypervisor.',
                              instance=instance)
                    with timeutils.StopWatch() as timer:
                        #调用hypervisor的spawn方法启动instance，如果使用libvirt的话，会调用nova/virt/libvirt/driver.py/libvirtDriver.spawn
                        self.driver.spawn(context, instance, image_meta,
                                          injected_files, admin_password,
                                          network_info=network_info,
                                          block_device_info=block_device_info)
                    LOG.info(_LI('Took %0.2f seconds to spawn the instance on '
                                 'the hypervisor.'), timer.elapsed(),
                             instance=instance)
        except (exception.InstanceNotFound,
                exception.UnexpectedDeletingTaskStateError) as e:
```

 - 看下nova/virt/libvirt/driver.py/libvirtDriver.spawn 方法：

```python
def spawn(self, context, instance, image_meta, injected_files,
              admin_password, network_info=None, block_device_info=None):
        #根据模拟器类型，获取块设备及光驱的总线类型
        #默认使用kvm，所以：块设备默认使用virtio；光驱默认使用ide；并且根据block_device_info设置设备映射
        #最后返回包含{disk_bus,cdrom_bus,mapping}的字典
        disk_info = blockinfo.get_disk_info(CONF.libvirt.virt_type,
                                            instance,
                                            image_meta,
                                            block_device_info)
        gen_confdrive = functools.partial(self._create_configdrive,
                                          context, instance,
                                          admin_pass=admin_password,
                                          files=injected_files,
                                          network_info=network_info)
        #从glance下载镜像（如果本地_base目录没有的话），然后上传到后端存储
        self._create_image(context, instance,
                           disk_info['mapping'],
                           network_info=network_info,
                           block_device_info=block_device_info,
                           files=injected_files,
                           admin_pass=admin_password)

        # Required by Quobyte CI
        self._ensure_console_log_for_instance(instance)
        #生成libvirt xml文件（libvirt创建虚拟机使用）
        xml = self._get_guest_xml(context, instance, network_info,
                                  disk_info, image_meta,
                                  block_device_info=block_device_info)
        #调用libvirt启动实例
        self._create_domain_and_network(
            context, xml, instance, network_info, disk_info,
            block_device_info=block_device_info,
            post_xml_callback=gen_confdrive)
        LOG.debug("Instance is running", instance=instance)

        def _wait_for_boot():
            """Called at an interval until the VM is running."""
            state = self.get_info(instance).state

            if state == power_state.RUNNING:
                LOG.info(_LI("Instance spawned successfully."),
                         instance=instance)
                raise loopingcall.LoopingCallDone()
        #等待实例创建结果（通过libvirt获取云主机状态判断）
        timer = loopingcall.FixedIntervalLoopingCall(_wait_for_boot)
        timer.start(interval=0.5).wait()
```

 - 关于具体创建系统磁盘的代码_create_image，生成libvirt xml，调用libvirt启动实例的代码不再分析，可以参考这篇文章[check this](http://blog.csdn.net/lzw06061139/article/details/51505514)。

<a name="C"></a>

## novacompute 整体讲解

 - novacomputer 代码目录如下：

![nova-computer-code](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-09-nova-computer-code.png)


 - Resource Tracker:
   nova-computer 会维护数据库中一个computer_nodes的表用来存储主机的资源使用情况，以便nova-scheduler获取作为选择主机的依据，这就要求每次创建，迁移，删除虚拟机时都要对数据库中的的computer_node进行更新。
   nova-computer会为每一个主机创建一个ResourceTracker对象，更新computernode对象。
   目前是有两种更新机制：一种是ResourceTracker的claim机制，一种是使用周期性任务（Periodic Task）
1. Claim机制：当一台主机被多个nova-scheduler同时选中并发出创建虚拟机的请求时，该主机并不一定有足够的资源满足创建要求。所以Claim机制就是在创建虚拟机之前先测试下是否满足新建需求，满足，则更新数据库，并将虚拟机申请的资源从主机可用的资源减掉，如果后来创建失败，或者虚拟机删除时，会通过claim加上之前减掉的部分。

``` python
@utils.synchronized(COMPUTE_RESOURCE_SEMAPHORE)
def instance_claim(self, context, instance, limits=None):

    if self.disabled:
        # compute_driver doesn't support resource tracking, just
        # set the 'host' and node fields and continue the build:
        self._set_instance_host_and_node(instance)
        return claims.NopClaim()

    # sanity checks:
    if instance.host:
        LOG.warning(_LW("Host field should not be set on the instance "
                        "until resources have been claimed."),
                    instance=instance)

    if instance.node:
        LOG.warning(_LW("Node field should not be set on the instance "
                        "until resources have been claimed."),
                    instance=instance)
    #允许超配创建虚拟机
    # get the overhead required to build this instance:
    overhead = self.driver.estimate_instance_overhead(instance)
    LOG.debug("Memory overhead for %(flavor)d MB instance; %(overhead)d "
              "MB", {'flavor': instance.flavor.memory_mb,
                      'overhead': overhead['memory_mb']})
    LOG.debug("Disk overhead for %(flavor)d GB instance; %(overhead)d "
              "GB", {'flavor': instance.flavor.root_gb,
                     'overhead': overhead.get('disk_gb', 0)})

    pci_requests = objects.InstancePCIRequests.get_by_instance_uuid(
        context, instance.uuid)
    #如果Claim返回none表示主机的资源满足不了新建虚拟机的需求，
    #会调用__exit__()方法将占用的资源返还到主机可用资源中。
    claim = claims.Claim(context, instance, self, self.compute_node,
                         pci_requests, overhead=overhead, limits=limits)

    # self._set_instance_host_and_node() will save instance to the DB
    # so set instance.numa_topology first.  We need to make sure
    # that numa_topology is saved while under COMPUTE_RESOURCE_SEMAPHORE
    # so that the resource audit knows about any cpus we've pinned.
    instance_numa_topology = claim.claimed_numa_topology
    instance.numa_topology = instance_numa_topology
    #通过Object model，若不在同一节点则调用ConductorAPI 更新instance的host,node等属性。
    self._set_instance_host_and_node(instance)

    if self.pci_tracker:
        # NOTE(jaypipes): ComputeNode.pci_device_pools is set below
        # in _update_usage_from_instance().
        self.pci_tracker.claim_instance(context, pci_requests,
                                        instance_numa_topology)

    # Mark resources in-use and update stats
    #根据新建虚拟机的需求计算主机可用资源。
    self._update_usage_from_instance(context, instance)

    elevated = context.elevated()
    # persist changes to the compute node:
    #根据上面的计算结果更新数据库。
    self._update(elevated)

    return claim
``` 



2. 使用Periodic Task:在类nova.compute.manager.ComputeManager中有个周期性的任务update_available_resource()用于更新主机可用资源。顺便提一点，nova-compute在启动的时候会启动两个周期任务，一个用于更新主机可用资源update_available_resource，一个用于汇报本机nova-compute的服务状态给数据库（report state）。

```python
    #根据配置文件周期性的调用该函数。
    #该周期性的任务用于同步数据库内可用资源与hypervisor保持一致
    @periodic_task.periodic_task(spacing=CONF.update_resources_interval)
    def update_available_resource(self, context):
        """See driver.get_available_resource()

        Periodic process that keeps that the compute host's understanding of
        resource availability and usage in sync with the underlying hypervisor.

        :param context: security context
        """
        compute_nodes_in_db = self._get_compute_nodes_in_db(context,
                                                            use_slave=True)
        nodenames = set(self.driver.get_available_nodes())
        #更新所有主机数据库中的资源数据。
        for nodename in nodenames:
            self.update_available_resource_for_node(context, nodename)

        self._resource_tracker_dict = {
            k: v for k, v in self._resource_tracker_dict.items()
            if k in nodenames}

        # Delete orphan compute node not reported by driver but still in db
        #清除inactive但仍在数据库中存有数据的计算节点信息
        for cn in compute_nodes_in_db:
            if cn.hypervisor_hostname not in nodenames:
                LOG.info(_LI("Deleting orphan compute node %s"), cn.id)
                cn.destroy()
``` 

3. 这两种机制并不冲突，一种是在新建或迁移虚拟机等涉及计算节点资源操作时，在当前数据库中数据的基础上更新，，保证数据库里的资源及时更新，以便为nova-scheduler 提供最新数据。周期性任务是保证hypervisor获取的资源数据与数据库中的数据保持一致。

4. 关于resource tracker 如何更新数据库的代码分析
    [nova-compute Periodic tasks 机制](http://blog.csdn.net/gj19890923/article/details/50583435)



## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[Openstack liberty源码分析 之 云主机的启动过程3](http://blog.csdn.net/lzw06061139/article/details/51505514)

[nova-compute Periodic tasks 机制](http://blog.csdn.net/gj19890923/article/details/50583435)


***END***
