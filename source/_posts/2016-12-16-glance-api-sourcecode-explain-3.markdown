---
layout: post
title:  "openstack系列--glanceApi 源代码解读(三)"
date:   2016-12-16 10:08:25
tags: openstack
---

## 目录结构：


[glanceApi V2 代码解读](#A)


<a name="A"></a>

## glanceApi V2 代码解读

- 还是以image create为例，首先debug一下，看下都是发出了哪些restful请求：

本来是要截图的，网络出了点问题，云环境暂时用不了，直接写吧，发出了三个restful请求：
    1. GET /v2/schemas/image 
  
    2. POST /v2/images

    3. PUT /v2/images/xxxxxxxxxxxxxxxx/file
    
大致猜出第二，三个请求分别是image metedata的数据库操作请求，上传镜像的操作请求。接下来逐一分析：

 - 先看下GET /v2/schemas/image 请求，根据路由的规则router.py找到对应的执行函数：

```python
class API(wsgi.Router):

    """WSGI router for Glance v2 API requests."""

    def __init__(self, mapper):
        #默认scheme的property就几个，
        #如果自己增加的话需要写在一个schema-image.json文件中，通过此函数load
        custom_image_properties = images.load_custom_properties()

        reject_method_resource = wsgi.Resource(wsgi.RejectMethodController())
        #根据配置文件create 一个controller，看下文分析
        schemas_resource = schemas.create_resource(custom_image_properties)
        mapper.connect('/schemas/image',
                       controller=schemas_resource,
                       action='image',
                       conditions={'method': ['GET']},
                       body_reject=True)
        mapper.connect('/schemas/image',
                       controller=reject_method_resource,
                       action='reject',
                       allowed_methods='GET')
        mapper.connect('/schemas/images',
                       controller=schemas_resource,
                       action='images',
                       conditions={'method': ['GET']},
                       body_reject=True)
        ............................................
```

schemas.create_resource，看下该方法的实现：

```python
def create_resource(custom_image_properties=None):
    controller = Controller(custom_image_properties)
    return wsgi.Resource(controller)
```
可以看到该方法初始化了一个Controller,然后封装成一个WSGI APP，看下初始化：

```python
class Controller(object):
    def __init__(self, custom_image_properties=None):
        #创建一个image_schema
        self.image_schema = images.get_schema(custom_image_properties)
        self.image_collection_schema = images.get_collection_schema(
            custom_image_properties)
        self.member_schema = image_members.get_schema()
        self.member_collection_schema = image_members.get_collection_schema()
        self.task_schema = tasks.get_task_schema()
        self.task_collection_schema = tasks.get_collection_schema()

        # Metadef schemas
        self.metadef_namespace_schema = metadef_namespaces.get_schema()
        self.metadef_namespace_collection_schema = (
            metadef_namespaces.get_collection_schema())

        self.metadef_resource_type_schema = metadef_resource_types.get_schema()
        self.metadef_resource_type_collection_schema = (
            metadef_resource_types.get_collection_schema())

        self.metadef_property_schema = metadef_properties.get_schema()
        self.metadef_property_collection_schema = (
            metadef_properties.get_collection_schema())

        self.metadef_object_schema = metadef_objects.get_schema()
        self.metadef_object_collection_schema = (
            metadef_objects.get_collection_schema())

        self.metadef_tag_schema = metadef_tags.get_schema()
        self.metadef_tag_collection_schema = (
            metadef_tags.get_collection_schema())
    #调用此image方法，并最终返回image_schema.raw()的对象
    def image(self, req):
        return self.image_schema.raw()

```

看下创建image_schema的方法images.get_schema：

```python
def get_schema(custom_properties=None):
    #获取描述一个image的基本property，id,name ,status,owner等
    properties = get_base_properties()
    links = _get_base_links()
    #根据配置文件创建Schema,allow_additional_image_properties默认True,
    #区别是PermissiveSchema多设置了links参数 
    if CONF.allow_additional_image_properties:
        schema = glance.schema.PermissiveSchema('image', properties, links)
    else:
        schema = glance.schema.Schema('image', properties)
    #如果是自己定制的property,需要做合并操作
    if custom_properties:
        for property_value in custom_properties.values():
            property_value['is_base'] = False
        schema.merge_properties(custom_properties)
    return schema
```

最后看下Schema对象包含什么内容

```python
class Schema(object):

    def __init__(self, name, properties=None, links=None, required=None,
                 definitions=None):
        self.name = name
        if properties is None:
            properties = {}
        self.properties = properties
        self.links = links
        self.required = required
        self.definitions = definitions
        .........................
    #该方法返回一个dick
    def raw(self):
            raw = {
                'name': self.name,
                'properties': self.properties,
                'additionalProperties': False,
            }
            if self.definitions:
                raw['definitions'] = self.definitions
            if self.required:
                raw['required'] = self.required
            if self.links:
                raw['links'] = self.links
            return raw
 #Schema的子类 ，分析同上
class PermissiveSchema(Schema):
    @staticmethod
    def _filter_func(properties, key):
        return True

    def raw(self):
        raw = super(PermissiveSchema, self).raw()
        raw['additionalProperties'] = {'type': 'string'}
        return raw

    def minimal(self):
        minimal = super(PermissiveSchema, self).raw()
        return minimal
```

第一个请求的分析结束，所以该请求的目的就是获取镜像所支持的属性字典定义，接下来就该根据这些信息来验证用户输入参数。

- 看下第二条请求，路由到glance/api/v2/images.py.ImagesController.create方法：

```python
@utils.mutating
    def create(self, req, image, extra_properties, tags):
        #责任链的模式，
        #分别创建image_factory,image_repo对象
        image_factory = self.gateway.get_image_factory(req.context)
        image_repo = self.gateway.get_repo(req.context)
        try:
            image = image_factory.new_image(extra_properties=extra_properties,
                                            tags=tags, **image)
            image_repo.add(image)
        except (exception.DuplicateLocation,
                exception.Invalid) as e:
            raise webob.exc.HTTPBadRequest(explanation=e.msg)
        except (exception.ReservedProperty,
                exception.ReadonlyProperty) as e:
            raise webob.exc.HTTPForbidden(explanation=e.msg)
        except exception.Forbidden as e:
            LOG.debug("User not permitted to create image")
            raise webob.exc.HTTPForbidden(explanation=e.msg)
        except exception.LimitExceeded as e:
            LOG.warn(encodeutils.exception_to_unicode(e))
            raise webob.exc.HTTPRequestEntityTooLarge(
                explanation=e.msg, request=req, content_type='text/plain')
        except exception.Duplicate as e:
            raise webob.exc.HTTPConflict(explanation=e.msg)
        except exception.NotAuthenticated as e:
            raise webob.exc.HTTPUnauthorized(explanation=e.msg)
        except TypeError as e:
            LOG.debug(encodeutils.exception_to_unicode(e))
            raise webob.exc.HTTPBadRequest(explanation=e)
        #返回一个 image对象
        return image
```

关于责任链模式，[check this](http://www.cnblogs.com/ejiyuan/archive/2012/07/26/2610238.html)

先看下gateway.get_image_factory方法：

```python
class Gateway(object):
    def __init__(self, db_api=None, store_api=None, notifier=None,
                 policy_enforcer=None):
        self.db_api = db_api or glance.db.get_api()
        self.store_api = store_api or glance_store
        self.store_utils = store_utils
        self.notifier = notifier or glance.notifier.Notifier()
        self.policy = policy_enforcer or policy.Enforcer()
    #这里可以清晰地看出是责任链模式，
    def get_image_factory(self, context):
        image_factory = glance.domain.ImageFactory()
        #对location的检验
        store_image_factory = glance.location.ImageFactoryProxy(
            image_factory, context, self.store_api, self.store_utils)
        #对quota的检验
        quota_image_factory = glance.quota.ImageFactoryProxy(
            store_image_factory, context, self.db_api, self.store_utils)
        #policy检验
        policy_image_factory = policy.ImageFactoryProxy(
            quota_image_factory, context, self.policy)
        notifier_image_factory = glance.notifier.ImageFactoryProxy(
            policy_image_factory, context, self.notifier)
        if property_utils.is_property_protection_enabled():
            property_rules = property_utils.PropertyRules(self.policy)
            pif = property_protections.ProtectedImageFactoryProxy(
                notifier_image_factory, context, property_rules)
            authorized_image_factory = authorization.ImageFactoryProxy(
                pif, context)
        else:
            authorized_image_factory = authorization.ImageFactoryProxy(
                notifier_image_factory, context)
        return authorized_image_factory

```

可以看出*ImageFactoryProxy类都继承自glance/domain/proxy.py.ImageFactory，通过类名可以大致猜出其功能：镜像工厂，那就是用来创建封装镜像对象的；各个子类也分别实现:权限检查、消息通知、策略检查、配额检查等。
另外各个*ImageFactoryProxy类都依赖于*ImageProxy类。而各*ImageProxy类都继承自glance/domain/proxy.py.Image，该类描述的是镜像的属性信息，包括：name，image_id, status等。各*ImageProxy类是对Image的扩展。

关于这个责任链的代码一直没搞清楚，感兴趣的话，看这篇文章吧，[Openstack liberty Glance上传镜像源码分析](http://ceph.org.cn/2016/06/01/openstack-liberty-glance%E4%B8%8A%E4%BC%A0%E9%95%9C%E5%83%8F%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/),以后有机会再去解读。

按照上述给出的那篇参考文章的解释，new_image()方法和add()方法，类似一个封包，一个解包的过程，主要完成权限检查，配额检查，策略检查，发布通知以及记录数据库等操作。下面来看看镜像文件的上传过程。

 - 第三条请求，路由到glance/api/v2/image_data.py/ImageDataController.upload方法：

```python
@utils.mutating
    def upload(self, req, image_id, data, size):
        image_repo = self.gateway.get_repo(req.context)
        image = None
        refresher = None
        cxt = req.context
        try:
            #get方法与前述的add方法类似，首先从数据库取出image_id指向的条目，
            #封装成`domain/__init__.py/Image`对象，
            #然后经过层层封装返回 `authorization/ImageProxy`对象 
            image = image_repo.get(image_id)
            #更新镜像状态为saving - ‘保存中’
            image.status = 'saving'
            try:
                if CONF.data_api == 'glance.db.registry.api':
                    # create a trust if backend is registry
                    try:
                        # request user plugin for current token
                        user_plugin = req.environ.get('keystone.token_auth')
                        roles = []
                        # use roles from request environment because they
                        # are not transformed to lower-case unlike cxt.roles
                        for role_info in req.environ.get(
                                'keystone.token_info')['token']['roles']:
                            roles.append(role_info['name'])
                        refresher = trust_auth.TokenRefresher(user_plugin,
                                                              cxt.tenant,
                                                              roles)
                    except Exception as e:
                        LOG.info(_LI("Unable to create trust: %s "
                                     "Use the existing user token."),
                                 encodeutils.exception_to_unicode(e))
                #和上面的save方法相似的处理方式，逐层调用`ImageProxy`的set_data，
                #在该过程中会检查用户配额，发送通知，
                #最后根据glance-api.conf文件中配置存储后端上传镜像文件（通过add方法）到指定地方存储 
                image_repo.save(image, from_state='queued')
                #镜像上传成功后（在`location.py/set_data方法中上传文件成功后，
                #修改状态为active），更新数据库状态为active（这个时候可以在 
                #Dashboard上看到状态为'运行中'）
                image.set_data(data, size)

                try:
                    image_repo.save(image, from_state='saving')
                except exception.NotAuthenticated:
                    if refresher is not None:
                        # request a new token to update an image in database
                        cxt.auth_token = refresher.refresh_token()
                        image_repo = self.gateway.get_repo(req.context)
                        image_repo.save(image, from_state='saving')
                    else:
                        raise
         
            .........................................

```

主体代码也是责任链的模式，太蛋疼了，看不明白。


## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)

[Openstack liberty Glance上传镜像源码分析](http://blog.csdn.net/lzw06061139/article/details/51540456)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
