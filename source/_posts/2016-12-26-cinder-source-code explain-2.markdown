---
layout: post
title:  "openstack系列--cinder 源码解读(二)---常见cinder操作工作流程"
date:   2016-12-26 09:57:25
tags: openstack
---

## 目录结构：

[cinder 源码大致解析](#A)

[以新建 cinder volume 为例 深入 cinder 代码](#B)



- 关于cinder不再赘述，之前文章由介绍。
- cinder 的 API 目前有三个版本，本次讨论以v2为基准。
- 因为版本不同，代码也可能不同，该版本为Openstack Liberty


<a name="A"></a>

## cinder 源码大致解析

看下cinder项目的入口文件setup.cfg：

```
console_scripts =
    cinder-api = cinder.cmd.api:main
    cinder-backup = cinder.cmd.backup:main
    cinder-manage = cinder.cmd.manage:main
    cinder-rootwrap = oslo_rootwrap.cmd:main
    cinder-rtstool = cinder.cmd.rtstool:main
    cinder-scheduler = cinder.cmd.scheduler:main
    cinder-volume = cinder.cmd.volume:main
    cinder-volume-usage-audit = cinder.cmd.volume_usage_audit:main
```

- cinder-api: 启动cinder 的wsgi server,每一个API对应一种资源，在setup.cfg 的entry point 中进行了配置，使用stevedore进行加载。
- cinder-backup:将volume 备份到其他存储系统
- cinder-manage：提供对cinder其他服务的管理操作。
- cinder-rootwrap : 某些cinder操作需要root权限
- cinder-rtstool :如果iSCSI target管理工具不是使用的默认的tgtadm，而是使用LIO iSCSI的话，就可以使用该命令，能够让你创建，删除，和验证卷，以及决定target，且可添加iSCSI initiator到系统。
- cinder-scheduler :用于根据提供的策略调度cinder-volune节点
- cinder-volume：管理 volume的生命周期，与底层存储交互。
- cinder-volume-usage-audit：用于卷使用情况统计。



<a name="B"></a>

## 以新建 cinder volume 为例 深入 cinder 代码

首先看下cinder --debug create volume 都发送了那些请求：
![cinder create api](http://7xrnwq.com1.z0.glb.clouddn.com/2016-12-26-cinder-create-api.png)

可以看到先去keystone验证，然后向cinder-api发送一个v2/{tenant-id}/volumes 的post请求，volume创建，然后发送一个v2/{tenant-id}/volumes/{volume-id} 的get请求获取创建的volume数据。

接下来看下cinder-api中接受create volume 的post请求代码，路由过程略过：

/cinder/api/v2/volumes.py 主要代码如下
```python
@wsgi.response(202)
    def create(self, req, body):
        """Creates a new volume."""
        .....对传入参数的检测.......

        kwargs['metadata'] = volume.get('metadata', None)

        snapshot_id = volume.get('snapshot_id')
        if snapshot_id is not None:
            if not uuidutils.is_uuid_like(snapshot_id):
                msg = _("Snapshot ID must be in UUID form.")
                raise exc.HTTPBadRequest(explanation=msg)
            # Not found exception will be handled at the wsgi level
            kwargs['snapshot'] = self.volume_api.get_snapshot(context,
                                                              snapshot_id)
        else:
            kwargs['snapshot'] = None

        source_volid = volume.get('source_volid')
        if source_volid is not None:
            # Not found exception will be handled at the wsgi level
            kwargs['source_volume'] = \
                self.volume_api.get_volume(context,
                                           source_volid)
        else:
            kwargs['source_volume'] = None

        source_replica = volume.get('source_replica')
        if source_replica is not None:
            # Not found exception will be handled at the wsgi level
            src_vol = self.volume_api.get_volume(context,
                                                 source_replica)
            if src_vol['replication_status'] == 'disabled':
                explanation = _('source volume id:%s is not'
                                ' replicated') % source_replica
                raise exc.HTTPBadRequest(explanation=explanation)
            kwargs['source_replica'] = src_vol
        else:
            kwargs['source_replica'] = None

        consistencygroup_id = volume.get('consistencygroup_id')
        if consistencygroup_id is not None:
            # Not found exception will be handled at the wsgi level
            kwargs['consistencygroup'] = \
                self.consistencygroup_api.get(context,
                                              consistencygroup_id)
        else:
            kwargs['consistencygroup'] = None

        size = volume.get('size', None)
        if size is None and kwargs['snapshot'] is not None:
            size = kwargs['snapshot']['volume_size']
        elif size is None and kwargs['source_volume'] is not None:
            size = kwargs['source_volume']['size']
        elif size is None and kwargs['source_replica'] is not None:
            size = kwargs['source_replica']['size']

        LOG.info(_LI("Create volume of %s GB"), size)

        if self.ext_mgr.is_loaded('os-image-create'):
            image_ref = volume.get('imageRef')
            if image_ref is not None:
                image_uuid = self._image_uuid_from_ref(image_ref, context)
                kwargs['image_id'] = image_uuid

        kwargs['availability_zone'] = volume.get('availability_zone', None)
        kwargs['scheduler_hints'] = volume.get('scheduler_hints', None)
        kwargs['multiattach'] = utils.get_bool_param('multiattach', volume)

        new_volume = self.volume_api.create(context,
                                            size,
                                            volume.get('display_name'),
                                            volume.get('display_description'),
                                            **kwargs)

        retval = self._view_builder.detail(req, new_volume)

        return retval

```
















## 参考文章

[深入理解Openstack QoS控制实现与实践](http://int32bit.me/2016/07/16/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Openstack-QoS%E6%8E%A7%E5%88%B6%E5%AE%9E%E7%8E%B0%E4%B8%8E%E5%AE%9E%E8%B7%B5/)

[Volume Transfer(volume过户)](http://www.jianshu.com/p/fd432c29f277)

[openstack使用Ceph存储后端创建虚拟机快照原理剖析](http://int32bit.me/2016/10/25/Openstack%E4%BD%BF%E7%94%A8Ceph%E5%AD%98%E5%82%A8%E5%90%8E%E7%AB%AF%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%BF%AB%E7%85%A7%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***
