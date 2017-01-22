---
layout: post
title:  "openstack系列--novaConductor 源代码解读"
date:   2016-12-06 16:06:25
tags: openstack
---

## 目录结构：

[novaConductor 介绍](#A)


[以nova boot 为例解读代码 ](#B)



<a name="A"></a>

## novaConductor 介绍

 - 关于nova不再赘述，之前文章由介绍。
 - 根据[wiki](http://docs.openstack.org/developer/nova/conductor.html)介绍，NovaConductor更像是一个编排工具，它会接受novaapi的请求，将创建虚拟机等任务交给novacompute处理，将寻找host的任务交给novascheduler，同时，会负责跟数据库的交互。
 - 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## 以nova boot 为例解读代码 

 - 先简略叙述下nova boot 在novaapi的过程，novaapi接收restful请求（post /servers）,根据apipaste配置文件,APIrouter,该请求最终路由到nova/api/openstack/servers.py.ServersController.create ,该方法完成参数的转换，解析，以及policy认证等工作后，调用nova/compute/api.py.API.create，再转给_create_instance 方法。

```python
def _create_instance(self, context, instance_type,
               image_href, kernel_id, ramdisk_id,
               min_count, max_count,
               display_name, display_description,
               key_name, key_data, security_groups,
               availability_zone, user_data, metadata, injected_files,
               admin_password, access_ip_v4, access_ip_v6,
               requested_networks, config_drive,
               block_device_mapping, auto_disk_config, filter_properties,
               reservation_id=None, legacy_bdm=True, shutdown_terminate=False,
               check_server_group_quota=False):
        """Verify all the input parameters regardless of the provisioning
        strategy being performed and schedule the instance(s) for
        creation.
        """
        # Normalize and setup some parameters
        if reservation_id is None:
            reservation_id = utils.generate_uid('r')
        security_groups = security_groups or ['default']
        min_count = min_count or 1
        max_count = max_count or min_count
        block_device_mapping = block_device_mapping or []
        #获取image相关信息（metadata）
        if image_href:
            image_id, boot_meta = self._get_image(context, image_href)
        else:
            image_id = None
            boot_meta = self._get_bdm_image_metadata(
                context, block_device_mapping, legacy_bdm)

        self._check_auto_disk_config(image=boot_meta,
                                     auto_disk_config=auto_disk_config)
        #生成instance配置
        base_options, max_net_count, key_pair = \
                self._validate_and_build_base_options(
                    context, instance_type, boot_meta, image_href, image_id,
                    kernel_id, ramdisk_id, display_name, display_description,
                    key_name, key_data, security_groups, availability_zone,
                    user_data, metadata, access_ip_v4, access_ip_v6,
                    requested_networks, config_drive, auto_disk_config,
                    reservation_id, max_count)
        
        # max_net_count is the maximum number of instances requested by the
        # user adjusted for any network quota constraints, including
        # consideration of connections to each requested network
        if max_net_count < min_count:
            raise exception.PortLimitExceeded()
        elif max_net_count < max_count:
            LOG.info(_LI("max count reduced from %(max_count)d to "
                         "%(max_net_count)d due to network port quota"),
                        {'max_count': max_count,
                         'max_net_count': max_net_count})
            max_count = max_net_count
        #确定块设备映射
        block_device_mapping = self._check_and_transform_bdm(context,
            base_options, instance_type, boot_meta, min_count, max_count,
            block_device_mapping, legacy_bdm)

        # We can't do this check earlier because we need bdms from all sources
        # to have been merged in order to get the root bdm.
        self._checks_for_create_and_rebuild(context, image_id, boot_meta,
                instance_type, metadata, injected_files,
                block_device_mapping.root_bdm())

        instance_group = self._get_requested_instance_group(context,
                                   filter_properties)
        #创建instance对象，并写入数据库
        instances = self._provision_instances(context, instance_type,
                min_count, max_count, base_options, boot_meta, security_groups,
                block_device_mapping, shutdown_terminate,
                instance_group, check_server_group_quota, filter_properties,
                key_pair)
        #更新instance状态为create
        for instance in instances:
            self._record_action_start(context, instance,
                                      instance_actions.CREATE)
        #调用conductor api，之后会通过conductor rpc将请求转发给conductor manager
        self.compute_task_api.build_instances(context,
                instances=instances, image=boot_meta,
                filter_properties=filter_properties,
                admin_password=admin_password,
                injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=False)
        return (instances, reservation_id)

```

 -nova/conductor/api.py.ComputeTaskAPI.build_instance 直接将请求转发给nova/conductor/rpcapi.py.ComputeTaskAPI.build_instance。

```python
 def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True):
        image_p = jsonutils.to_primitive(image)
        version = '1.10'
        if not self.client.can_send_version(version):
            version = '1.9'
            if 'instance_type' in filter_properties:
                flavor = filter_properties['instance_type']
                flavor_p = objects_base.obj_to_primitive(flavor)
                filter_properties = dict(filter_properties,
                                         instance_type=flavor_p)
        kw = {'instances': instances, 'image': image_p,
               'filter_properties': filter_properties,
               'admin_password': admin_password,
               'injected_files': injected_files,
               'requested_networks': requested_networks,
               'security_groups': security_groups}
        if not self.client.can_send_version(version):
            version = '1.8'
            kw['requested_networks'] = kw['requested_networks'].as_tuples()
        if not self.client.can_send_version('1.7'):
            version = '1.5'
            bdm_p = objects_base.obj_to_primitive(block_device_mapping)
            kw.update({'block_device_mapping': bdm_p,
                       'legacy_bdm': legacy_bdm})
        #生成RPCClient对象，准备发送
        cctxt = self.client.prepare(version=version)
        #发送异步rpc消息到消息队列，conductor-manager会收到该消息
        cctxt.cast(context, 'build_instances', **kw)
```

顺便说下这个生成的cctxt其实是一个RPCClient对象，在oslo.messaging项目中，调用的call,cast方法就是RPCClient对象对应的方法。

 - 在讲conductor-manage之前，先说一下rabbitmq服务端的创建过程，该过程在服务启动的时候创建。其实server创建的主要目的就是监听端口，而openstack的服务主要通过两种方式调用：AMQP和restful。所以server的创建也主要有两种：一种是监听AMQP请求（Service），一种是监听restful请求（WSGIService）。大致看下nova-conductor的启动过程。

```python
def main():
    config.parse_args(sys.argv) #解析配置文件
    logging.setup(CONF, "nova") # 日志模块
    utils.monkey_patch() #eventlet模块的补丁
    objects.register_all() # object模块注册（与数据库访问有关）
    objects.Service.enable_min_version_cache()

    gmr.TextGuruMeditation.setup_autorun(version)
    #生成一个server对象
    server = service.Service.create(binary='nova-conductor',
                                    topic=CONF.conductor.topic,
                                    manager=CONF.conductor.manager)
    workers = CONF.conductor.workers or processutils.get_worker_count()
    #启动service
    service.serve(server, workers=workers)
    service.wait()
```

看下server对象的创建过程service.Service.create：

```python
    def create(cls, host=None, binary=None, topic=None, manager=None,
               report_interval=None, periodic_enable=None,
               periodic_fuzzy_delay=None, periodic_interval_max=None,
               db_allowed=True):
        """Instantiates class and passes back application object.

        :param host: defaults to CONF.host
        :param binary: defaults to basename of executable
        :param topic: defaults to bin_name - 'nova-' part
        :param manager: defaults to CONF.<topic>_manager
        :param report_interval: defaults to CONF.report_interval
        :param periodic_enable: defaults to CONF.periodic_enable
        :param periodic_fuzzy_delay: defaults to CONF.periodic_fuzzy_delay
        :param periodic_interval_max: if set, the max time to wait between runs

        """
        # 下面的参数解释见上面的英文注释
        if not host:
            host = CONF.host
        if not binary:
            binary = os.path.basename(sys.argv[0])
        #topic就是rabbitmq中连接queue 的依据
        if not topic:
            topic = binary.rpartition('nova-')[2]
        #真正执行代码的manager类
        if not manager:
            manager_cls = ('%s_manager' %
                           binary.rpartition('nova-')[2])
            manager = CONF.get(manager_cls, None)
        if report_interval is None:
            report_interval = CONF.report_interval
        if periodic_enable is None:
            periodic_enable = CONF.periodic_enable
        if periodic_fuzzy_delay is None:
            periodic_fuzzy_delay = CONF.periodic_fuzzy_delay

        debugger.init()
        #传参，完成初始化
        service_obj = cls(host, binary, topic, manager,
                          report_interval=report_interval,
                          periodic_enable=periodic_enable,
                          periodic_fuzzy_delay=periodic_fuzzy_delay,
                          periodic_interval_max=periodic_interval_max,
                          db_allowed=db_allowed)

        return service_obj
```
 
 再看下service.serve()完成的工作,该函数直接调用oslo-service的launch()方法:

``` python
def launch(conf, service, workers=1, restart_method='reload'):

    if workers is not None and workers <= 0:
        raise ValueError(_("Number of workers should be positive!"))
    #只起一个进程来运行服务
    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf, restart_method=restart_method)
    else:
    #起woker个进程运行服务
        launcher = ProcessLauncher(conf, restart_method=restart_method)
    launcher.launch_service(service, workers=workers)

    return launcher
```

进入launcher.launch_service,调用一个add方法，看下代码（此处为起一个进程的launcher为例）：

``` python
    def add(self, service):
        """Add a service to a list and create a thread to run it.

        :param service: service to run
        """
        #将一个service加入service list,并起一个thread去执行
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)
```
add_thread会起一个green thread，执行callback这个回调，在这里就是server.start():

``` python
def start(self):
 
          .......................................
        target = messaging.Target(topic=self.topic, server=self.host)

        endpoints = [
            self.manager,
            baserpc.BaseRPCAPI(self.manager.service_name, self.backdoor_port)
        ]
        endpoints.extend(self.manager.additional_endpoints)

        serializer = objects_base.NovaObjectSerializer()
        #获取rpcserver并监听对应端口
        self.rpcserver = rpc.get_server(target, endpoints, serializer)
        self.rpcserver.start()

        self.manager.post_start_hook()
        ...................................
```

 - 接下来看conductor-manage的处理过程。nova-conductor收到rpc请求，根据路由映射，该请求会交给对应manager的函数处理，nova/conductor/manager.py.ComputeTaskManager.build_instances

``` python
def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping=None, legacy_bdm=True):
        if (requested_networks and
                not isinstance(requested_networks,
                               objects.NetworkRequestList)):
            requested_networks = objects.NetworkRequestList.from_tuples(
                requested_networks)
        # 
        flavor = filter_properties.get('instance_type')
        if flavor and not isinstance(flavor, objects.Flavor):
            # Code downstream may expect extra_specs to be populated since it
            # is receiving an object, so lookup the flavor to ensure this.
            flavor = objects.Flavor.get_by_id(context, flavor['id'])
            filter_properties = dict(filter_properties, instance_type=flavor)
        request_spec = {}
        try:
            # check retry policy. Rather ugly use of instances[0]...
            # but if we've exceeded max retries... then we really only
            # have a single instance.
            #生成发送给scheduler的信息
            request_spec = scheduler_utils.build_request_spec(
                context, image, instances)
            scheduler_utils.populate_retry(
                filter_properties, instances[0].uuid)
            #发送同步消息给nova-scheduler，选取用于创建云主机的主机(留给下篇文章)
            hosts = self._schedule_instances(
                    context, request_spec, filter_properties)
        except Exception as exc:
            updates = {'vm_state': vm_states.ERROR, 'task_state': None}
            #设置虚拟机state,taskstate
            for instance in instances:
                self._set_vm_state_and_notify(
                    context, instance.uuid, 'build_instances', updates,
                    exc, request_spec)
                try:
                    # If the BuildRequest stays around then instance show/lists
                    # will pull from it rather than the errored instance.
                    self._destroy_build_request(context, instance)
                except exception.BuildRequestNotFound:
                    pass
                self._cleanup_allocated_networks(
                    context, instance, requested_networks)
            return
            .............................
            #发送异步消息给nova-compute，完成instance的boot(下篇文章)
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

从上面的源码可以看出，build_instances方法主要实现过滤参数的组装，然后通过客户端scheduler_client发送rpc请求到scheduler完成host的选取，最后发送rpc请求到选取的host上，由nova-compute完成云主机的启动。

 - 对nova boot在 Novaconductor的分析就完成了。简单从整体源码层面分析一下novaconductor,源码目录结构如下：

![nova conductor](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-07-nova-conductor-code.png)

一般来说，rpcapi.py都是与rpc相关的，别的服务只需导入该模块便可以使用它提供的远程调用Novaconductor的服务，Novaconductor注册的RPCServer接收到rpc请求，然后由manager.py中的ConductorManager真正完成数据库的访问。由于数据库访问的特殊性，api.py又对rpc.api做了一层封装，所以其他模块需要导入的是api.py，api.py的封装主要是区分访问数据库是否需要通过RPC,如果数据库就在本地host,那么无需通过RPC调用。所以api.py中包含了四个类：LocalAPI,API,LocalComputerTaskAPI,ComputerTaskAPI。前两个类是Novaconductor访问数据库的接口，后两个是TaskaAPI接口，TaskaAPI接口主要包含耗时较长的任务，比如新建虚拟机，迁移虚拟机等。如果Novacompute与Novaconductor模块在同一台机器上，那么不用通过RPC,直接通过LocalAPI访问数据库，同理TaskAPI也不需要RPC,使用LocalTaskAPI。看下nova/conductor/__init__.py

``` python
def API(*args, **kwargs):
    #根据配置选项use_local判断Novacompute与novaconductor是否在同一节点，决定使用哪一类API
    use_local = kwargs.pop('use_local', False)
    if CONF.conductor.use_local or use_local:
        api = conductor_api.LocalAPI
    else:
        api = conductor_api.API
    return api(*args, **kwargs)

def ComputeTaskAPI(*args, **kwargs):
    #分析同上
    use_local = kwargs.pop('use_local', False)
    if CONF.conductor.use_local or use_local:
        api = conductor_api.LocalComputeTaskAPI
    else:
        api = conductor_api.ComputeTaskAPI
    return api(*args, **kwargs)
```

 - 接下来看下novaconductor与数据库交互的部分，在manager.py文件中有两个类，一个ConductorManager,一个ComputerTaskManager,对应两种API。与数据库交互的主要是ConductorManager。但当我打开这个类时瞬间懵圈了：
 
  ![conductor-manager](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-07-conductor-manager.png)

类里面不是我们期待的数据库操作方法，而是关于object的一些方法。这里要讲下object Model。

 - Object Model算是nova访问数据库的分水岭。之前，对某一个表的操作都是放在一个同名文件中,例如flavor.py,使用时直接调用文件中的函数操作数据库。Object Model引入后，新建flavor对象与flavor表对应，将对flavor 的操作封装在flavor对象中。这么做的原因大致是：1，nova-computer 与数据库升级时的版本问题。2，减少写入数据库的数据量。3，数据库的传值类型问题。
  看下图Object Model 工作流程：

   ![object model process](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-07-object-model.jpg)

   novaComputer 与novaConductor不在同一个节点时，虚线为引入ObjectModel之前NovaComputer访问数据库的流程。实现表示引入objectModel后的流程。可以看到，Novacomputer需要做数据库的操作时，将通过ObjectModel调用nova.conductor.rpcapi.ConductorAPI提供的RPC接口，novaConductor 接受到RPC请求后，通过本地ObjectModel完成数据库的更新。
   ObjectModel的代码位于nova/objects目录，里面的每一个类对应数据库中的一个表，如下instance.py

 ``` python
 @obj_base.NovaObjectRegistry.register
class Network(obj_base.NovaPersistentObject, obj_base.NovaObject,
              obj_base.NovaObjectDictCompat):
    # Version 1.0: Initial version
    # Version 1.1: Added in_use_on_host()
    # Version 1.2: Added mtu, dhcp_server, enable_dhcp, share_address
    VERSION = '1.2'
    #基类base.NovaObject 会记录变化的字段，更新数据库时只更新这些变化的字段。
    #字典field是ComputerNode对象维护的信息，该字典的值不一定包含ComputerNode表内所有信息，每一个值的类型都是nova.object.fields模块定义的类型，若数据类型不匹配，就会抛出异常。
    fields = {
        'id': fields.IntegerField(),
        'label': fields.StringField(),
        'injected': fields.BooleanField(),
        'cidr': fields.IPV4NetworkField(nullable=True),
        'cidr_v6': fields.IPV6NetworkField(nullable=True),
        'multi_host': fields.BooleanField(),
        'netmask': fields.IPV4AddressField(nullable=True),
        'gateway': fields.IPV4AddressField(nullable=True),
        'broadcast': fields.IPV4AddressField(nullable=True),
        'netmask_v6': fields.IPV6AddressField(nullable=True),
        'gateway_v6': fields.IPV6AddressField(nullable=True),
        'bridge': fields.StringField(nullable=True),
        'bridge_interface': fields.StringField(nullable=True),
        'dns1': fields.IPAddressField(nullable=True),
        'dns2': fields.IPAddressField(nullable=True),
        'vlan': fields.IntegerField(nullable=True),
        'vpn_public_address': fields.IPAddressField(nullable=True),
        'vpn_public_port': fields.IntegerField(nullable=True),
        'vpn_private_address': fields.IPAddressField(nullable=True),
        'dhcp_start': fields.IPV4AddressField(nullable=True),
        'rxtx_base': fields.IntegerField(nullable=True),
        'project_id': fields.UUIDField(nullable=True),
        'priority': fields.IntegerField(nullable=True),
        'host': fields.StringField(nullable=True),
        'uuid': fields.UUIDField(),
        'mtu': fields.IntegerField(nullable=True),
        'dhcp_server': fields.IPAddressField(nullable=True),
        'enable_dhcp': fields.BooleanField(),
        'share_address': fields.BooleanField(),
        }
     ......................................

    @obj_base.remotable_classmethod
    def get_by_id(cls, context, network_id, project_only='allow_none'):
        db_network = db.network_get(context, network_id,
                                    project_only=project_only)
        return cls._from_db_object(context, cls(), db_network)

    @obj_base.remotable_classmethod
    def get_by_uuid(cls, context, network_uuid):
        db_network = db.network_get_by_uuid(context, network_uuid)
        return cls._from_db_object(context, cls(), db_network)

 ```

nova.object.base 中有定义两个重要的修饰函数：remotable_classmethod 和remotable,前者用于修饰类的方法，后者用于修饰实例的方法。
如果novacomputer 与novaconductor不在同一节点，novacomputer 初始化时会将novaObject.indirection_api初始化为nova.conductor.rpcapi.ConductorAPI,此时调用remotable_classmethod修饰的函数时，会通过rpc交给Novaconductor处理，rpc请求中包含了该函数名，novaconductor接收到该rpc请求会调用相应的object model 函数完成数据库操作。







## 参考文章

[nova wiki](http://docs.openstack.org/developer/nova/conductor.html)

[Openstack liberty源码分析 之 云主机的启动过程1](http://blog.csdn.net/lzw06061139/article/details/51488899)

[openstack 设计与实现](https://book.douban.com/subject/26374647/)


***END***
