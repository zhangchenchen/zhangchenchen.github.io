---
layout: post
title:  "openstack系列--novaClient 源代码解读"
date:   2016-11-30 13:06:25
tags: openstack
---

## 目录结构：

[novaclient 介绍](#A)

[以nova list 为例解读代码 ](#B)





<a name="A"></a>

## novaclient 介绍

 - novaclient是Python写的nova API服务的客户端.
 - 提供了两种使用形式：命令行形式和Python API形式（命令行形式其实最终也是调用的Python API）。
 - novaclient 本身不是类似于nova-api这种的service，没有一个守护进程在运行，而是在我们输入命令的之后根据入口main函数去相应的执行。
 - 因为版本不同，代码也可能不同，该版本为Openstack Liberty

 
<a name="B"></a>

## 以nova list 为例解读代码

 - 进入novaclient的根目录，找到setup.cfg文件，找到入口函数

  ![novaclientsetup](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20161130-setup.cfg.png)
  其实我们也可以在安装了novaclient的机器中执行 which nova 找到对应入口。

 - 进入novaclient.shell 的main函数，截取部分代码如下：

```python
def main():
    try:
        argv = [encodeutils.safe_decode(a) for a in sys.argv[1:]]
        OpenStackComputeShell().main(argv) #重点在这个函数
    except Exception as exc:
        logger.debug(exc, exc_info=1)
        if six.PY2:
            message = encodeutils.safe_encode(six.text_type(exc))
        else:
            message = encodeutils.exception_to_unicode(exc)
        print("ERROR (%(type)s): %(msg)s" % {
              'type': exc.__class__.__name__,
              'msg': message},
              file=sys.stderr)
        sys.exit(1)
    except KeyboardInterrupt:
        print(_("... terminating nova client"), file=sys.stderr)
        sys.exit(130)

if __name__ == "__main__":
    main()
```

 - 找到OpenStackComputeShell().main函数,

```python
 def main(self, argv):
        # Parse args once to find version and debug settings
        parser = self.get_base_parser(argv)
        (args, args_list) = parser.parse_known_args(argv)

        self.setup_debugging(args.debug)
        self.extensions = []
        do_help = ('help' in argv) or (
            '--help' in argv) or ('-h' in argv) or not argv

        # bash-completion should not require authentication
        skip_auth = do_help or (
            'bash-completion' in argv)

        if not args.os_compute_api_version:
            api_version = api_versions.get_api_version(
                DEFAULT_MAJOR_OS_COMPUTE_API_VERSION)
        else:
            api_version = api_versions.get_api_version(
                args.os_compute_api_version)
            .........................
```

这个函数很长，大体做的工作如下：

1,解析命令行，判定是否需要做认证操作（如help命令不需要）

2,需要做认证操作的话，判定认证方法（一般为keystone）

``` python
  if use_session:
                # Not using Nova auth plugin, so use keystone
                with utils.record_time(self.times, args.timings,
                                       'auth_url', args.os_auth_url):
                    keystone_session = (
                        loading.load_session_from_argparse_arguments(args))
                    keystone_auth = (
                        loading.load_auth_from_argparse_arguments(args))
            else:
                # set password for auth plugins
                os_password = args.os_password
```

3,创建一个Client用于确定API版本

``` python
        # This client is just used to discover api version. Version API needn't
        # microversion, so we just pass version 2 at here.
        self.cs = client.Client(
            api_versions.APIVersion("2.0"),
            os_username, os_password, os_project_name,
            tenant_id=os_project_id, user_id=os_user_id,
            auth_url=os_auth_url, insecure=insecure,
            region_name=os_region_name, endpoint_type=endpoint_type,
            extensions=self.extensions, service_type=service_type,
            service_name=service_name, auth_token=auth_token,
            volume_service_name=volume_service_name,
            timings=args.timings, bypass_url=bypass_url,
            os_cache=os_cache, http_log_debug=args.debug,
            cacert=cacert, timeout=timeout,
            session=keystone_session, auth=keystone_auth,
            logger=self.client_logger)
        if not skip_auth:
            if not api_version.is_latest():
                if api_version > api_versions.APIVersion("2.0"):
                    if not api_version.matches(novaclient.API_MIN_VERSION,
                                               novaclient.API_MAX_VERSION):
                        raise exc.CommandError(
                            _("The specified version isn't supported by "
                              "client. The valid version range is '%(min)s' "
                              "to '%(max)s'") % {
                                "min": novaclient.API_MIN_VERSION.get_string(),
                                "max": novaclient.API_MAX_VERSION.get_string()}
                        )
            api_version = api_versions.discover_version(self.cs, api_version)
```

4,根据发现的API版本创建一个client 

``` python
        # Recreate client object with discovered version.
        self.cs = client.Client(
            api_version,
            os_username, os_password, os_project_name,
            tenant_id=os_project_id, user_id=os_user_id,
            auth_url=os_auth_url, insecure=insecure,
            region_name=os_region_name, endpoint_type=endpoint_type,
            extensions=self.extensions, service_type=service_type,
            service_name=service_name, auth_token=auth_token,
            volume_service_name=volume_service_name,
            timings=args.timings, bypass_url=bypass_url,
            os_cache=os_cache, http_log_debug=args.debug,
            cacert=cacert, timeout=timeout,
            session=keystone_session, auth=keystone_auth)
```

5,根据args调用指定版本API

``` python
 args.func(self.cs, args)
```

6,看下最终调用的V2版本下的shell.py(此处为do_list()函数)

``` python
 @utils.arg(
    '--not-tags-any',
    dest='not-tags-any',
    metavar='<not-tags-any>',
    default=None,
    help=_("Only the servers that do not have at least one of the given tags"
           "will be included in the list result. Boolean expression in this "
           "case is 'NOT(t1 OR t2)'. Tags must be separated by commas: "
           "--not-tags-any <tag1,tag2>"),
    start_version="2.26")
def do_list(cs, args):
    """List active servers."""
    imageid = None
    flavorid = None
    if args.image:
        imageid = _find_image(cs, args.image).id
    if args.flavor:
        flavorid = _find_flavor(cs, args.flavor).id
    # search by tenant or user only works with all_tenants
    if args.tenant or args.user:
        args.all_tenants = 1
        .......................
```
比较重要的语句如下：

``` python
servers = cs.servers.list(detailed=detailed,
                              search_opts=search_opts,
                              sort_keys=sort_keys,
                              sort_dirs=sort_dirs,
                              marker=args.marker,
                              limit=args.limit)
        .......................
```
7,调用v2 下client中的servers 中的list方法，

``` python
        password = kwargs.pop('password', api_key)
        self.projectid = project_id
        self.tenant_id = tenant_id
        self.user_id = user_id
        self.flavors = flavors.FlavorManager(self)
        self.flavor_access = flavor_access.FlavorAccessManager(self)
        self.images = images.ImageManager(self)
        self.glance = images.GlanceManager(self)
        self.limits = limits.LimitsManager(self)
        self.servers = servers.ServerManager(self) # 调用此处的servers
        self.versions = versions.VersionManager(self)
        .......................
```

8,终于到最后一步


``` python
 def list(self, detailed=True, search_opts=None, marker=None, limit=None,
             sort_keys=None, sort_dirs=None):
      
        if search_opts is None:
            search_opts = {}

        qparams = {}

        for opt, val in six.iteritems(search_opts):
            if val:
                if isinstance(val, six.text_type):
                    val = val.encode('utf-8')
                qparams[opt] = val

        detail = ""
        if detailed:
            detail = "/detail"

        result = base.ListWithMeta([], None)
        while True:
            if marker:
                qparams['marker'] = marker

            if limit and limit != -1:
                qparams['limit'] = limit

            # Transform the dict to a sequence of two-element tuples in fixed
            # order, then the encoded string will be consistent in Python 2&3.
            if qparams or sort_keys or sort_dirs:
                # sort keys and directions are unique since the same parameter
                # key is repeated for each associated value
                # (ie, &sort_key=key1&sort_key=key2&sort_key=key3)
                items = list(qparams.items())
                if sort_keys:
                    items.extend(('sort_key', sort_key)
                                 for sort_key in sort_keys)
                if sort_dirs:
                    items.extend(('sort_dir', sort_dir)
                                 for sort_dir in sort_dirs)
                new_qparams = sorted(items, key=lambda x: x[0])
                query_string = "?%s" % parse.urlencode(new_qparams)
            else:
                query_string = ""

            servers = self._list("/servers%s%s" % (detail, query_string),
                                 "servers") #重点在这
            result.extend(servers)
            result.append_request_ids(servers.request_ids)

            if not servers or limit != -1:
                break
            marker = result[-1].id
        return result
```
找到最终调用的_list方法如下：

``` python
def _list(self, url, response_key, obj_class=None, body=None):
        if body:
            resp, body = self.api.client.post(url, body=body)
        else:
            resp, body = self.api.client.get(url)

        .......................
```
即最终通过restful api 去查找对应的数据,此处路径为（/servers/detail ）
self.api.client是novaclient.client.SessionClient类的对象，但是在SessionClient类中没有找到get函数，在其父类adapter.LegacyJsonAdapter中发现该函数，而adapter文件是在keystoneclient中，所以self.api.client.get(url)调用的是keystoneclient/adapter.py中的LegacyJsonAdapter类的get()函数,最终找到Adapter类中的get()函数。

```python
 def get(self, url, **kwargs):
        return self.request(url, 'GET', **kwargs)
```

之后会调用sessionClient的request()函数。

```python
    def request(self, url, method, **kwargs):
        kwargs.setdefault('headers', kwargs.get('headers', {}))
        api_versions.update_headers(kwargs["headers"], self.api_version)
        # NOTE(jamielennox): The standard call raises errors from
        # keystoneauth1, where we need to raise the novaclient errors.
        raise_exc = kwargs.pop('raise_exc', True)
        with utils.record_time(self.times, self.timings, method, url):
            resp, body = super(SessionClient, self).request(url,
                                                            method,
                                                            raise_exc=False,
                                                            **kwargs) # 该函数

        # if service name is None then use service_type for logging
        service = self.service_name or self.service_type
        _log_request_id(self.logger, resp, service)

        # TODO(andreykurilin): uncomment this line, when we will be able to
        #   check only nova-related calls
        # api_versions.check_headers(resp, self.api_version)
        if raise_exc and resp.status_code >= 400:
            raise exceptions.from_response(resp, body, url, method)

        return resp, body
```
调用adapter.LegacyJsonAdapter类中的request方法

```python
class LegacyJsonAdapter(Adapter):
    """Make something that looks like an old HTTPClient.

    A common case when using an adapter is that we want an interface similar to
    the HTTPClients of old which returned the body as JSON as well.

    You probably don't want this if you are starting from scratch.
    """

    def request(self, *args, **kwargs):
        headers = kwargs.setdefault('headers', {})
        headers.setdefault('Accept', 'application/json')

        try:
            kwargs['json'] = kwargs.pop('body')
        except KeyError:  # nosec(cjschaef): kwargs doesn't contain a 'body'
            # key, while 'json' is an optional argument for Session.request
            pass

        resp = super(LegacyJsonAdapter, self).request(*args, **kwargs)

        body = None
        if resp.text:
            try:
                body = jsonutils.loads(resp.text)
            except ValueError:  # nosec(cjschaef): return None for body as
                # expected
                pass

        return resp, body
```

不往下看了，再往下代码比较复杂。


## 参考文章

[openstack-L版源码解析之novaclient](http://www.lolo99.com/2016/09/23/openstack-L%E7%89%88%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B9%8Bnovaclient-2/)
[pythonnovaclient](https://github.com/openstack/python-novaclient)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
