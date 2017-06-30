---
layout: post
title:  "openstack系列--glanceApi 源代码解读(二)"
date:   2016-12-15 17:08:25
tags: openstack
---

## 目录结构：


[glanceApi V1 代码解读](#A)


<a name="A"></a>

## glanceApi V1 代码解读

- 首先简单看下V1 代码与V2 代码的目录结构：

![v1-vs-v2](http://7xrnwq.com1.z0.glb.clouddn.com/201612-15-openstack-glance-v1-v2.png)

也可以明显看出V2相对V1加了许多API。

- 下面以glance image-create 为例解读下glanceAPI V1 的代码，glanceClient发出rest请求后，glance-api Server监听到对应端口的请求，根据api-paste文件经过middleware的过滤后，根据配置文件中选定的版本，最终路由到v1/images.py文件中的对应函数，先看下WSGI 路由文件router.py

```python
class API(wsgi.Router):

    """WSGI router for Glance v1 API requests."""

    def __init__(self, mapper):
        reject_method_resource = wsgi.Resource(wsgi.RejectMethodController())

        images_resource = images.create_resource()

        mapper.connect("/",
                       controller=images_resource,
                       action="index")
        mapper.connect("/images",
                       controller=images_resource,
                       action='index',
                       conditions={'method': ['GET']})
        mapper.connect("/images",
                       controller=images_resource,
                       action='create',
                       conditions={'method': ['POST']})
        mapper.connect("/images",
                       controller=reject_method_resource,
                       action='reject',
                       allowed_methods='GET, POST')
        mapper.connect("/images/detail",
                       controller=images_resource,
                       action='detail',
                       conditions={'method': ['GET', 'HEAD']})
        mapper.connect("/images/detail",
                       controller=reject_method_resource,
                       action='reject',
                       allowed_methods='GET, HEAD')
        mapper.connect("/images/{id}",
                       controller=images_resource,
                       action="meta",
                       conditions=dict(method=["HEAD"]))
        mapper.connect("/images/{id}",
        ...............................

        members_resource = members.create_resource()

        mapper.connect("/images/{image_id}/members",
                       controller=members_resource,
                       action="index",
                       conditions={'method': ['GET']})
        mapper.connect("/images/{image_id}/members",
                       controller=members_resource,
                       action="update_all",
                       conditions=dict(method=["PUT"]))
        mapper.connect("/images/{image_id}/members",
                       controller=reject_method_resource,
                       action='reject',
                       allowed_methods='GET, PUT')
        mapper.connect("/images/{image_id}/members/{id}",
                       controller=members_resource,
                       action="show",
                       conditions={'method': ['GET']})
        ..................................
```

 主要有两部分，一部分是镜像image的操作，一部分是members的操作。POST  /images 请求最终路由到create函数。

```python
 @utils.mutating
    def create(self, req, image_meta, image_data):
        """
        Adds a new image to Glance. Four scenarios exist when creating an
        image:

        1. If the image data is available directly for upload, create can be
           passed the image data as the request body and the metadata as the
           request headers. The image will initially be 'queued', during
           upload it will be in the 'saving' status, and then 'killed' or
           'active' depending on whether the upload completed successfully.

        2. If the image data exists somewhere else, you can upload indirectly
           from the external source using the x-glance-api-copy-from header.
           Once the image is uploaded, the external store is not subsequently
           consulted, i.e. the image content is served out from the configured
           glance image store.  State transitions are as for option #1.

        3. If the image data exists somewhere else, you can reference the
           source using the x-image-meta-location header. The image content
           will be served out from the external store, i.e. is never uploaded
           to the configured glance image store.

        4. If the image data is not available yet, but you'd like reserve a
           spot for it, you can omit the data and a record will be created in
           the 'queued' state. This exists primarily to maintain backwards
           compatibility with OpenStack/Rackspace API semantics.

        The request body *must* be encoded as application/octet-stream,
        otherwise an HTTPBadRequest is returned.

        Upon a successful save of the image data and metadata, a response
        containing metadata about the image is returned, including its
        opaque identifier.

        :param req: The WSGI/Webob Request object
        :param image_meta: Mapping of metadata about image
        :param image_data: Actual image data that is to be stored

        :raises: HTTPBadRequest if x-image-meta-location is missing
                and the request body is not application/octet-stream
                image data.
        """
        #上文注释里面提到create image时出现的四种情况.不赘述
        #查看policy是否允许‘add_image’操作,下同
        self._enforce(req, 'add_image')
        is_public = image_meta.get('is_public')
        if is_public:
            self._enforce(req, 'publicize_image')
        if Controller._copy_from(req):
            self._enforce(req, 'copy_from')
        if image_data or Controller._copy_from(req):
            self._enforce(req, 'upload_image')
        #查看property是否合法
        self._enforce_create_protected_props(image_meta['properties'].keys(),
                                             req)
        #验证property的个数是否超过配置文件规定项目的配额quota
        self._enforce_image_property_quota(image_meta, req=req)
        #将image_meta传给glance-registry,调用glance-registrycli
        image_meta = self._reserve(req, image_meta)
        id = image_meta['id']
        #上传镜像数据操作。
        image_meta = self._handle_source(req, id, image_meta, image_data)

        location_uri = image_meta.get('location')
        if location_uri:
            self.update_store_acls(req, id, location_uri, public=is_public)

        # Prevent client from learning the location, as it
        # could contain security credentials
        image_meta = redact_loc(image_meta)

        return {'image_meta': image_meta}
```

- 先看下metadata 的处理过程_reserve方法：

```python
def _reserve(self, req, image_meta):
        """
        Adds the image metadata to the registry and assigns
        an image identifier if one is not supplied in the request
        headers. Sets the image's status to `queued`.

        :param req: The WSGI/Webob Request object
        :param id: The opaque image identifier
        :param image_meta: The image metadata

        :raises: HTTPConflict if image already exists
        :raises: HTTPBadRequest if image metadata is not valid
        """
        #注释里说得很清楚了，给rigistry增加一个image metedata,如果没有id的话
        #给他绑定一个id,并将该image status 设置为‘queued’
        location = self._external_source(image_meta, req)
        scheme = image_meta.get('store')
        if scheme and scheme not in store.get_known_schemes():
            msg = _("Required store %s is invalid") % scheme
            LOG.warn(msg)
            raise HTTPBadRequest(explanation=msg,
                                 content_type='text/plain')

        image_meta['status'] = ('active' if image_meta.get('size') == 0
                                else 'queued')
        #如果不是直接上床，路径为外部路径，那么先对外部路径validate以及获取image size
        if location:
            try:
                backend = store.get_store_from_location(location)
            except (store.UnknownScheme, store.BadStoreUri):
                LOG.debug("Invalid location %s", location)
                msg = _("Invalid location %s") % location
                raise HTTPBadRequest(explanation=msg,
                                     request=req,
                                     content_type="text/plain")
            # check the store exists before we hit the registry, but we
            # don't actually care what it is at this point
            self.get_store_or_400(req, backend)

            # retrieve the image size from remote store (if not provided)
            image_meta['size'] = self._get_size(req.context, image_meta,
                                                location)
        #如果是本地直接上传，那么将size设置为0，上传的时候会再设置
        else:
            # Ensure that the size attribute is set to zero for directly
            # uploadable images (if not provided). The size will be set
            # to a non-zero value during upload
            image_meta['size'] = image_meta.get('size', 0)

        try:
            #调用registry.client，向glance-registry发送rest请求
            image_meta = registry.add_image_metadata(req.context, image_meta)
            self.notifier.info("image.create", redact_loc(image_meta))
            return image_meta
        except exception.Duplicate:
            msg = (_("An image with identifier %s already exists") %
                   image_meta['id'])
            LOG.warn(msg)
            raise HTTPConflict(explanation=msg,
                               request=req,
                               content_type="text/plain")
        except exception.Invalid as e:
            msg = (_("Failed to reserve image. Got error: %s") %
                   encodeutils.exception_to_unicode(e))
            LOG.exception(msg)
            raise HTTPBadRequest(explanation=msg,
                                 request=req,
                                 content_type="text/plain")
        except exception.Forbidden:
            msg = _("Forbidden to reserve image.")
            LOG.warn(msg)
            raise HTTPForbidden(explanation=msg,
                                request=req,
                                content_type="text/plain")
```

进入registry.client，看下相关代码：

```python
def add_image_metadata(context, image_meta):
    LOG.debug("Adding image metadata...")
    #获取registry_client ,code explain veerything
    c = get_registry_client(context)
    return c.add_image(image_meta)
```

看下最终的add_image()方法：

```python
    def add_image(self, image_metadata):
        """
        Tells registry about an image's metadata
        """
        headers = {
            'Content-Type': 'application/json',
        }

        if 'image' not in image_metadata:
            image_metadata = dict(image=image_metadata)

        encrypted_metadata = self.encrypt_metadata(image_metadata['image'])
        image_metadata['image'] = encrypted_metadata
        body = jsonutils.dump_as_bytes(image_metadata)
        #向registry发送POST请求。返回一个类似JSON形式 的dict
        res = self.do_request("POST", "/images", body=body, headers=headers)
        # Registry returns a JSONified dict(image=image_info)
        data = jsonutils.loads(res.read())
        image = data['image']
        return self.decrypt_metadata(image)
```

再往下就是glance-registry 接收到rest请求后对数据的操作。

不像nova-conductor通过Object Model进行db访问，glance-registry相对比较简单，直接通过glance db模块进行访问，就不再另写一篇文章介绍glance-registry,直接看下经过路由后的执行函数：

```python
 @utils.mutating
    def create(self, req, body):
        """Registers a new image with the registry.

        :param req: wsgi Request object
        :param body: Dictionary of information about the image

        :returns: The newly-created image information as a mapping,
            which will include the newly-created image's internal id
            in the 'id' field
        """
        image_data = body['image']

        # Ensure the image has a status set
        image_data.setdefault('status', 'active')

        # Set up the image owner
        if not req.context.is_admin or 'owner' not in image_data:
            image_data['owner'] = req.context.owner

        image_id = image_data.get('id')
        if image_id and not uuidutils.is_uuid_like(image_id):
            LOG.info(_LI("Rejecting image creation request for invalid image "
                         "id '%(bad_id)s'"), {'bad_id': image_id})
            msg = _("Invalid image id format")
            return exc.HTTPBadRequest(explanation=msg)

        if 'location' in image_data:
            image_data['locations'] = [image_data.pop('location')]

        try:
            image_data = _normalize_image_location_for_db(image_data)
            #调用glance db 模块进行数据库操作，
            #glance db 模块是对sqlalchemy 的封装
            image_data = self.db_api.image_create(req.context, image_data)
            image_data = dict(image=make_image_dict(image_data))
            LOG.info(_LI("Successfully created image %(id)s"),
                     {'id': image_data['image']['id']})
            return image_data
        except exception.Duplicate:
            msg = _("Image with identifier %s already exists!") % image_id
            LOG.warn(msg)
            return exc.HTTPConflict(msg)
        except exception.Invalid as e:
            msg = (_("Failed to add image metadata. "
                     "Got error: %s") % encodeutils.exception_to_unicode(e))
            LOG.error(msg)
            return exc.HTTPBadRequest(msg)
        except Exception:
            LOG.exception(_LE("Unable to create image %s"), image_id)
            raise
```

OK,对metedata的分析到这，再回去看下对image chunk data的处理,先是方法_handle_source：

```python
 def _handle_source(self, req, image_id, image_meta, image_data):
        copy_from = self._copy_from(req)
        location = image_meta.get('location')
        sources = [obj for obj in (copy_from, location, image_data) if obj]
        if len(sources) >= 2:
            msg = _("It's invalid to provide multiple image sources.")
            LOG.warn(msg)
            raise HTTPBadRequest(explanation=msg,
                                 request=req,
                                 content_type="text/plain")
        if len(sources) == 0:
            return image_meta
        if image_data:
            #第一种情况
            image_meta = self._validate_image_for_activation(req,
                                                             image_id,
                                                             image_meta)
            image_meta = self._upload_and_activate(req, image_meta)
        elif copy_from:
            #第二种情况，异步上传
            msg = _LI('Triggering asynchronous copy from external source')
            LOG.info(msg)
            pool = common.get_thread_pool("copy_from_eventlet_pool")
            pool.spawn_n(self._upload_and_activate, req, image_meta)
        else:
            #第三种情况，不上传，修改metadata location,以及status
            if location:
                self._validate_image_for_activation(req, image_id, image_meta)
                image_size_meta = image_meta.get('size')
                if image_size_meta:
                    try:
                        image_size_store = store.get_size_from_backend(
                            location, req.context)
                    except (store.BadStoreUri, store.UnknownScheme) as e:
                        LOG.debug(encodeutils.exception_to_unicode(e))
                        raise HTTPBadRequest(explanation=e.msg,
                                             request=req,
                                             content_type="text/plain")
                    # NOTE(zhiyan): A returned size of zero usually means
                    # the driver encountered an error. In this case the
                    # size provided by the client will be used as-is.
                    if (image_size_store and
                            image_size_store != image_size_meta):
                        msg = (_("Provided image size must match the stored"
                                 " image size. (provided size: %(ps)d, "
                                 "stored size: %(ss)d)") %
                               {"ps": image_size_meta,
                                "ss": image_size_store})
                        LOG.warn(msg)
                        raise HTTPConflict(explanation=msg,
                                           request=req,
                                           content_type="text/plain")
                location_data = {'url': location, 'metadata': {},
                                 'status': 'active'}
                image_meta = self._activate(req, image_id, location_data)
        return image_meta
```
看上面这段代码其实就是之前描述的四种情况：

        1. 如果是本地文件，直接上传，image status 为queued,上传成功，queued 更改为active。

        2. 如果是外部文件（如在swift或ceph等），如果使用这个header:x-glance-api-copy-from,那么会异步上传，status同1.

        3. 如果是外部文件，使用的header是x-image-meta-location，那么就不用上传了，但是image metedata 中的location 要更改为对应的外部地址。
        
        4. image文件是not available的话，不上传，但image metedata 会有，image status 为queued。


## 参考文章

[openstack 设计与实现](https://book.douban.com/subject/26374647/)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
