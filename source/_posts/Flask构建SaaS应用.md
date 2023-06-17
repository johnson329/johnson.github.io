---
title: Flask构建SaaS应用
date: 2020-12-18 14:49:01
tags: [flask开发]
categories: [flask]
---

### 租户隔离

一个客户就是一个租户，每个租户的数据在数据表中都有个一个tenantid字段用来与其他租户隔离

- 租户识别
<!--more-->

```python
from flask import g, request
    
def get_tenant_from_request():
    auth = validate_auth(request.headers.get('Authorization'))
    return Tenant.query.get(auth.tenant_id)
        
def get_current_tenant():
    rv = getattr(g, 'current_tenant', None)
    if rv is = None:
        rv = get_tenant_from_request()
        g.current_tenant = rv
    return rv
```

- 自动租户隔离

例如，每个租户有自己的Project，像下面这样批量直接修改project又忘记带上tenantid查询字段，就会把其他租户的project一并修改了

```python
def batch_update_projects(ids, changes):
    projects = Project.query.filter(
        Project.id.in_(ids) &
        Project.status != ProjectStatus.INVISIBLE
    )
    for project in projects: 
        update_projects(project, changes)
```

我们可以override Project的query属性以及使用sqlalchemy的event listener来自动的进行租户隔离，也就是说我们在使用orm查询的时候，会自动带上`tenentid`查询字段，这样无论是修改、删除、查询都只会是操作自己租户下面的数据，毕竟在orm中，删除和修改都是要先查询的嘛



```python
# model
class Project(TenantBoundMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    status = db.Column(db.Integer)
    def __repr__(self):
        return '<Project name=%r>' % self.name
```



```python
from sqlalchemy.ext.declarative import declared_attr
​
# 覆写Project类的query_class属性
class TenantQuery(db.Query):
    current_tenant_constrained = True
    def tenant_unconstrained_unsafe(self):
        rv = self._clone()
        rv.current_tenant_constrained = False
        return rv
# 通过mixin的方式，让Project类多继承这个类
# 从而覆写了Project类的query_class属性
class TenantBoundMixin(object):
    query_class = TenantQuery
    # 为project表定义了一个tenant_id字段
    @declared_attr
    def tenant_id(cls):
        return db.Column(db.Integer, db.ForeignKey('tenant.id'))
​
    @declared_attr
    def tenant(cls):
        return db.relationship(Tenant, uselist=False)
​
# event_listener的意思是，每次执行例如Project.query的时候都会执行
# 下面的钩子函数，
# 这个钩子函数会给query对象带上ilter_by(tenant=get_current_tenant())
@db.event.listens_for(TenantQuery, 'before_compile', retval=True)
def ensure_tenant_constrained(query):
    for desc in query.column_descriptions:
        if hasattr(desc['type'], 'tenant') and \
            query.current_tenant_constrained:
              query = query.filter_by(tenant=get_current_tenant())
    return query
```

接下来我们来看一下使用

```python
# 只查询我这个租户名下的project42
>>> Project.query.all()
[<Project name='project42'>]
# 去掉租户约束，查询所有project信息
>>> Project.query.tenant_unconstrained_unsafe().all()
[<Project name='project1'>, Project.name='project2', ...]
```

### 审计日志

```python
def log(action, message=None):
  data = {
        'action': action
        'timestamp': datetime.utcnow()
  }
  if message is not None:
         data['message'] = message
  if request:
         data['ip'] = request.remote_addr
  user = get_current_user()
  if user is not None:
         data['user'] = User
  db.session.add(LogMessage(**data))
```

更多详细信息，请参考

[flask多租户实践](http://mitsuhiko.pocoo.org/practicalsaas.pdf)