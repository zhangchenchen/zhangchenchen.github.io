---
layout: post
title:  "openstack系列--glanceApi 源代码解读(一)"
date:   2016-12-15 16:08:25
tags: openstack
---

## 目录结构：

[glanceApi 介绍](#A)

[glanceApi 启动过程代码解读](#B)


<a name="A"></a>

## glanceApi 介绍

- glanceApi主要作用就是启动一个WSGI Server,接受restful请求，路由给对应V1/V2版本的不同执行函数。
- 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
## glanceApi 启动过程代码解读

- 首先看下glanceAPI 启动代码入口：

```python
def main():
    try:
        #找到配置文件并读取配置文件中的值到config.CONF中
        config.parse_args()
        #设置默认值
        config.set_config_defaults()
        #设置协程调度使用epoll,若不支持，使用selects
        wsgi.set_eventlet_hub()
        logging.setup(CONF, 'glance')
        notifier.set_defaults()
        #osprofiler的使用与Ceilometer性能分析有关，此处略过
        if cfg.CONF.profiler.enabled:
            _notifier = osprofiler.notifier.create("Messaging",
                                                   oslo_messaging, {},
                                                   notifier.get_transport(),
                                                   "glance", "api",
                                                   cfg.CONF.bind_host)
            osprofiler.notifier.set(_notifier)
            osprofiler.web.enable(cfg.CONF.profiler.hmac_keys)
        else:
            osprofiler.web.disable()
        #初始化WSGI Server
        server = wsgi.Server(initialize_glance_store=True)
        #先加载paste文件，生成app,然后启动WSGI Serve
        server.start(config.load_paste_app('glance-api'), default_port=9292)
        #循环等待请求
        server.wait()
    except KNOWN_EXCEPTIONS as e:
        fail(e)
```

- 看下WSGI Server初始化的过程：

```python
class Server(object):
    """Server class to manage multiple WSGI sockets and applications.

    This class requires initialize_glance_store set to True if
    glance store needs to be initialized.
    """
    def __init__(self, threads=1000, initialize_glance_store=False):
        os.umask(0o27)  # ensure files are created with the correct privileges
        self._logger = logging.getLogger("eventlet.wsgi.server")
        self.threads = threads
        self.children = set()
        self.stale_children = set()
        self.running = True
        # NOTE(abhishek): Allows us to only re-initialize glance_store when
        # the API's configuration reloads.
        #注意此处的参数会在start时用于初始化glance的后端存储
        self.initialize_glance_store = initialize_glance_store
        self.pgid = os.getpid()
        try:
            # NOTE(flaper87): Make sure this process
            # runs in its own process group.
            os.setpgid(self.pgid, self.pgid)
        except OSError:
            self.pgid = 0
```


- 启动Server之前会加载paste文件，看下代码：

```python
def load_paste_app(app_name, flavor=None, conf_file=None):
    """
    Builds and returns a WSGI app from a paste config file.
    """
    # append the deployment flavor to the application name,
    # in order to identify the appropriate paste pipeline
    app_name += _get_deployment_flavor(flavor)

    if not conf_file:
        conf_file = _get_deployment_config_file()

    try:
        logger = logging.getLogger(__name__)
        logger.debug("Loading %(app_name)s from %(conf_file)s",
                     {'conf_file': conf_file, 'app_name': app_name})

        app = deploy.loadapp("config:%s" % conf_file, name=app_name)

        # Log the options used when starting if we're in debug mode...
        if CONF.debug:
            CONF.log_opt_values(logger, logging.DEBUG)
        return app
    except (LookupError, ImportError) as e:
        msg = (_("Unable to load %(app_name)s from "
                 "configuration file %(conf_file)s."
                 "\nGot: %(e)r") % {'app_name': app_name,
                                    'conf_file': conf_file,
                                    'e': e})
        logger.error(msg)
        raise RuntimeError(msg)
```
不解释了，code explain everything.

- Server start 的过程：

```python
    def start(self, application, default_port):
        """
        Run a WSGI server with the given application.

        :param application: The application to be run in the WSGI server
        :param default_port: Port to bind to if none is specified in conf
        """
        self.application = application
        self.default_port = default_port
        self.configure()
        self.start_wsgi()
    def configure(self, old_conf=None, has_changed=None):
        """
        Apply configuration settings

        :param old_conf: Cached old configuration settings (if any)
        :param has changed: callable to determine if a parameter has changed
        """
        eventlet.wsgi.MAX_HEADER_LINE = CONF.max_header_line
        self.client_socket_timeout = CONF.client_socket_timeout or None
        self.configure_socket(old_conf, has_changed)
        #初始化后端存储
        if self.initialize_glance_store:
            initialize_glance_store()
    def start_wsgi(self):
        #根据配置文件指定worker数创建Server 进程
        workers = get_num_workers()
        if workers == 0:
            # Useful for profiling, test, debug etc.
            self.pool = self.create_pool()
            self.pool.spawn_n(self._single_run, self.application, self.sock)
            return
        else:
            LOG.info(_LI("Starting %d workers"), workers)
            signal.signal(signal.SIGTERM, self.kill_children)
            signal.signal(signal.SIGINT, self.kill_children)
            signal.signal(signal.SIGHUP, self.hup)
            while len(self.children) < workers:
                self.run_child()
```

- 再看下初始化后端存储部分，initialize_glance_store方法：
```python
def initialize_glance_store():
    """Initialize glance store."""
    glance_store.register_opts(CONF)
    glance_store.create_stores(CONF)
    glance_store.verify_default_store()
```
执行了三个glance_store导入函数，实现根据配置文件的指定，加载相应的后端存储。这里提一下glance-store项目，这个项目之前跟glance是在一块的，J版本之后就独立出来。看了一下github上的README,说该库提供的接口不是特别的稳定，有一些缺陷，可能最终会重写这部分，所以，就不详细展开了。



## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[glance-api 服务启动流程(官方Kilo版本)
](https://zhaoqqi.github.io/2016/08/02/glance-image-driver/)

[glance-store ](https://github.com/openstack/glance_store)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
