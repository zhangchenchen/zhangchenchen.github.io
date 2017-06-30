---
layout: post
title:  "openstack系列--novaApi 源代码解读"
date:   2016-12-1 15:06:25
tags: openstack
---

## 目录结构：

[novaApi 介绍](#A)


[以nova list 为例解读代码 ](#B)



<a name="A"></a>

## novaApi 介绍

 - 关于nova不再赘述，之前文章由介绍。
 - 之前说的novaclient的主要作用就是将命令行最终转化为restful请求去novaApi查找。
 - novaApi的执行过程主要分为两个阶段：接收到restful请求后，根据PasteDeploy 将请求的路由到具体的WSGI Application ,之后Routers将请求路由到具体函数并执行。
 - 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## 以nova list 为例解读代码

 - 首先看下novaapi的启动过程，进入nova项目的setup.cfg文件

```python
console_scripts =
    nova-all = nova.cmd.all:main # nova 所有service的启动脚本，已弃用
    nova-api = nova.cmd.api:main #此处为Nova-api的启动脚本
    nova-api-metadata = nova.cmd.api_metadata:main
    nova-api-os-compute = nova.cmd.api_os_compute:main
    nova-cells = nova.cmd.cells:main
    nova-cert = nova.cmd.cert:main
    nova-compute = nova.cmd.compute:main
    nova-conductor = nova.cmd.conductor:main
    nova-console = nova.cmd.console:main
    nova-consoleauth = nova.cmd.consoleauth:main
    nova-dhcpbridge = nova.cmd.dhcpbridge:main
    nova-idmapshift = nova.cmd.idmapshift:main
    nova-manage = nova.cmd.manage:main
    nova-network = nova.cmd.network:main
    nova-novncproxy = nova.cmd.novncproxy:main
    nova-policy = nova.cmd.policy_check:main
    nova-rootwrap = oslo_rootwrap.cmd:main
    nova-rootwrap-daemon = oslo_rootwrap.cmd:daemon
    nova-scheduler = nova.cmd.scheduler:main
    nova-serialproxy = nova.cmd.serialproxy:main
    nova-spicehtml5proxy = nova.cmd.spicehtml5proxy:main
    nova-xvpvncproxy = nova.cmd.xvpvncproxy:main
wsgi_scripts =
    nova-placement-api = nova.api.openstack.placement.wsgi:init_application

nova.api.v21.extensions =
    admin_actions = nova.api.openstack.compute.admin_actions:AdminActions
    admin_password = nova.api.openstack.compute.admin_password:AdminPassword
    agents = nova.api.openstack.compute.agents:Agents
    aggregates = nova.api.openstack.compute.aggregates:Aggregates
    assisted_volume_snapshots = nova.api.openstack.compute.assisted_volume_snapshots:AssistedVolumeSnapshots
    attach_interfaces = nova.api.openstack.compute.attach_interfaces:AttachInterfaces
    availability_zone = nova.api.openstack.compute.availability_zone:AvailabilityZone
    baremetal_nodes = nova.api.openstack.compute.baremetal_nodes:BareMetalNodes
    block_device_mapping = nova.api.openstack.compute.block_device_mapping:BlockDeviceMapping
```

简单解释下console_scripts中值得注意的几个服务：

1. nova-api:包括两种API服务：nova-api-metadata ,nova-api-os-compute。根据配置文件中的enable_apis确定启动哪种服务。
2. nova-api-metadata:接受虚拟机的metadata相关的请求。只有采用nova-network部署时才使用。该工作目前已由neutron项目完成。
3. nova-api-os-compute:主要的Openstack Compute API服务。
4. nova-cells:主要适用于openstack增强横向扩展能力，[check this](http://forum.huawei.com/enterprise/thread-295547-1-1.html)
5. nova-rootwrap:在openstack运行时以root身份运行某些shell命令

setup.cfg文件中nova.api.v21.extensions的字段中的每一项对应一个API，novaapi 启动时会用stevedore据此动态加载。

- 首先进入main函数

```python
def main():
    config.parse_args(sys.argv)
    logging.setup(CONF, "nova")
    utils.monkey_patch()
    objects.register_all()
    if 'osapi_compute' in CONF.enabled_apis:
        # NOTE(mriedem): This is needed for caching the nova-compute service
        # version which is looked up when a server create request is made with
        # network id of 'auto' or 'none'.
        objects.Service.enable_min_version_cache()
    log = logging.getLogger(__name__)

    gmr.TextGuruMeditation.setup_autorun(version) # 生成Guru report的相关代码
    launcher = service.process_launcher() 
    started = 0
    for api in CONF.enabled_apis:  #根据配置文件中的配置创建server
        should_use_ssl = api in CONF.enabled_ssl_apis
        try:
            server = service.WSGIService(api, use_ssl=should_use_ssl) #创建WSGI server ,创建过程中paste deploy参与进来，加载相应application
            launcher.launch_service(server, workers=server.workers or 1) #根据配置文件中worker个数启动server
            started += 1
        except exception.PasteAppNotFound as ex:
            log.warning(
                _LW("%s. ``enabled_apis`` includes bad values. "
                    "Fix to remove this warning."), ex)
                     ............................
```

看下service.WSGIService()创建时发生了什么：

```python
class WSGIService(service.Service):
    """Provides ability to launch API from a 'paste' configuration."""

    def __init__(self, name, loader=None, use_ssl=False, max_url_len=None):
        """Initialize, but do not start the WSGI server.

        :param name: The name of the WSGI server given to the loader.
        :param loader: Loads the WSGI application using the given name.
        :returns: None

        """
        self.name = name
        # NOTE(danms): Name can be metadata, os_compute, or ec2, per
        # nova.service's enabled_apis
        self.binary = 'nova-%s' % name
        self.topic = None
        self.manager = self._get_manager()
        self.loader = loader or wsgi.Loader() #从paste配置文件加载api对应的wsgi application
        self.app = self.loader.load_app(name)
        # inherit all compute_api worker counts from osapi_compute
        if name.startswith('openstack_compute_api'):
            wname = 'osapi_compute'
        else:
            wname = name
        self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")
        self.port = getattr(CONF, '%s_listen_port' % name, 0)
        self.workers = (getattr(CONF, '%s_workers' % wname, None) or
                        processutils.get_worker_count())
                        ............................
        # 指定IP，port监听socket，server的实现再用了eventlet进行封装，监听到http请求时，不会新建一个线程，而是用协程。
         self.server = wsgi.Server(name,
                                  self.app,
                                  host=self.host,
                                  port=self.port,
                                  use_ssl=self.use_ssl,
                                  max_url_len=max_url_len)
        # Pull back actual port used
        self.port = self.server.port
        self.backdoor_port = None
        ................................
```


 - 当路径为/v21/servers/detail请求过来时，WSGI server监听到请求，根据paste配置文件路由到特定的WSGI application。

```python
[composite:openstack_compute_api_v21] 
use = call:nova.api.auth:pipeline_factory_v21 #调用pipeline_factory_v21函数，根据nova配置文件中的auth_strategy决定使用noauth2或keystone
noauth2 = compute_req_id faultwrap sizelimit noauth2 osapi_compute_app_v21
keystone = compute_req_id faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v21
 ......................
 [app:osapi_compute_app_v21]
paste.app_factory = nova.api.openstack.compute:APIRouterV21.factory
 ....................
```

据此，我们便找到最终调用的函数APIRouterV21.factory，第一阶段完成，进入第二阶段

 - 第二阶段主要是找到最终调用的函数，nova封装了route模块实现该路由功能。进入上次找到的路由函数APIRouterV21.factory。

```python
class APIRouterV21(base_wsgi.Router): 
    """Routes requests on the OpenStack v2.1 API to the appropriate controller
    and method.
    """
    @classmethod
    def factory(cls, global_config, **local_config):
        """Simple paste factory, :class:`nova.wsgi.Router` doesn't have one."""
        return cls()

    @staticmethod
    def api_extension_namespace():
        return 'nova.api.v21.extensions'
    #该类主要实现利用stevedore加载位于setup.cfg中nova.api.v21.extensions下所有资源，并利用check_func()函数进行检查
    def __init__(self, init_only=None):
        def _check_load_extension(ext):
            return self._register_extension(ext)

        self.api_extension_manager = stevedore.enabled.EnabledExtensionManager(
            namespace=self.api_extension_namespace(),
            check_func=_check_load_extension,
            invoke_on_load=True,
            invoke_kwds={"extension_info": self.loaded_extension_info})

        mapper = ProjectMapper()

        self.resources = {}

        # NOTE(cyeoh) Core API support is rewritten as extensions
        # but conceptually still have core
        #这部分做的工作就是对所有封装的资源进行注册（对资源的属性进行一些扩展），并使用mapper对象建立路由规则（与***controllers类建立）。
        if list(self.api_extension_manager):
            # NOTE(cyeoh): Stevedore raises an exception if there are
            # no plugins detected. I wonder if this is a bug.
            self._register_resources_check_inherits(mapper)
            self.api_extension_manager.map(self._register_controllers)

        LOG.info(_LI("Loaded extensions: %s"),
                 sorted(self.loaded_extension_info.get_extensions().keys()))
        super(APIRouterV21, self).__init__(mapper)
        #打印出该mapper对象便可找到路由关系
        .....................

```
 nova中每种资源都被封装成一个nova.api.openstack.wsgi.Resource资源，并封装为一个WSGI application。

 - 最后APIRouterV21将mapper传给父类Router做最后的工作.

```python
class Router(object):
    """WSGI middleware that maps incoming requests to WSGI apps."""
    #使用routes模块将mapper与_dispatch()关联起来
    def __init__(self, mapper):
        """Create a router for the given routes.Mapper.

        Each route in `mapper` must specify a 'controller', which is a
        WSGI app to call.  You'll probably want to specify an 'action' as
        well and have your controller be an object that can route
        the request to the action-specific method.
        """
        self.map = mapper
        self._router = routes.middleware.RoutesMiddleware(self._dispatch,
                                                          self.map)
    #根据mapper将请求路由到适当的WSGI应用，即资源上。
    @webob.dec.wsgify(RequestClass=Request)
    def __call__(self, req):
        """Route the incoming request to a controller based on self.map.
        
        If no match, return a 404.
         
        """
        return self._router

    @staticmethod
    @webob.dec.wsgify(RequestClass=Request)
    def _dispatch(req):
        """Dispatch the request to the appropriate controller.

        Called by self._router after matching the incoming request to a route
        and putting the information into req.environ.  Either returns 404
        or the routed WSGI app's response.

        """
        match = req.environ['wsgiorg.routing_args'][1]
        if not match:
            return webob.exc.HTTPNotFound()
        app = match['controller']
        return app

```

至此，所有阶段便完成了，GET /v21/servers/detail最终会调用的函数是setup.cfg中的server对应的controller的detail函数。

```python
@extensions.expected_errors((400, 403))
    def detail(self, req):
        """Returns a list of server details for a given user."""
        context = req.environ['nova.context']
        context.can(server_policies.SERVERS % 'detail')
        try:
            servers = self._get_servers(req, is_detail=True) #调用下面的_get_servers函数
        except exception.Invalid as err:
            raise exc.HTTPBadRequest(explanation=err.format_message())
        return servers
        ..............

def _get_servers(self, req, is_detail):
        """Returns a list of servers, based on any search options specified."""

        search_opts = {}
        search_opts.update(req.GET)

        context = req.environ['nova.context']
        remove_invalid_options(context, search_opts,
                self._get_server_search_options(req))

        # Verify search by 'status' contains a valid status.
        # Convert it to filter by vm_state or task_state for compute_api.
        # For non-admin user, vm_state and task_state are filtered through
        # remove_invalid_options function, based on value of status field.
        # Set value to vm_state and task_state to make search simple.
        search_opts.pop('status', None)
        ............
        #最终调用函数在这里compute_api.get_all()
         try:
            instance_list = self.compute_api.get_all(elevated or context,
                    search_opts=search_opts, limit=limit, marker=marker,
                    expected_attrs=expected_attrs,
                    sort_keys=sort_keys, sort_dirs=sort_dirs)
        except exception.MarkerNotFound:
            msg = _('marker [%s] not found') % marker
            raise exc.HTTPBadRequest(explanation=msg)
        except exception.FlavorNotFound:
            LOG.debug("Flavor '%s' could not be found ",
                      search_opts['flavor'])
            instance_list = objects.InstanceList()

```



## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
