title: 简洁Flask RESTful API设计
date: 2016-03-26 17:50:44
categories: Flask
tags:
	- Python web frame
	- 笔记
---
#### 搞个大新闻
关于什么是RESTful APIs已经有足够多的资料可查阅了，但是怎么设计一个冗余代码少的API，还是需要踩很多坑才能体悟出来的。
好比要设计一个用户注册的API，先假设需要经过下列步骤:
1. 解析json并获取email以及password
2. 验证email及password
3. 用crypt（或其他的库）对password进行哈希。
4. 将新的user插入数据库
5. 发送验证邮件/欢迎邮件给用户
6. 发送新用户提醒给系统管理员
7. 返回json response给客户端，告知其请求失败还是成功
8. 限制单个用户IP的请求频率，防止攻击。


看起来也不是特别的复杂是不是？
<!--more-->
将全部流程在一个视图入口里实现，看起来是这样子的。
```python
@api.route('/user', methods=['POST'])
def create_user():
    data = request.get_json(force=True)
    email    = json.get('email','')
    password = json.get('password','')
    attrs = {}

    if not re.match("^[^@ ]+@[^@ ]+\.[^@ ]+$", email):
        raise ValidationError()
    attrs['email'] = email

    if len(password) < 7:
        raise ValidationError()
    attrs['password'] = bcrypt.hashpw(password, bcrypt.gensalt())
    attrs['id'] = str(db.users.insert(attrs))
    del attrs['password']
    session['id'] = new_user['id']
    notify('admins', 'New User', repr(new_user))
    envelope = Envelope(
        to_addr   = new_user['email'],
        subject   = api.config['WELCOME_EMAIL_SUBJECT'],
        text_body = render_template('emails/new_user.txt'),
        html_body = render_template('emails/new_user.html'))
    def sender():
        envelopes.connstack.push_connection(conn)
        smtp = envelopes.connstack.get_current_connection()
        smtp.send(envelope)
        envelopes.connstack.pop_connection()
    gevent.spawn(sender)
    
    return jsonify({'status': 'OK'})
```

#### 一点人生经验
可是，这样子的API不好维护。设计API有几点DOs:
1. 利用blueprints来实现API版本更迭
2. 构建可测试的API模块
3. 利用装饰器减少冗余的代码
4. 充分利用Flask可定制的错误处理

一个简单的API目录结构如下：
```python
realmon/
    server.py
    api/
        __init__.py
        v1/                  <--- API major version
            __init__.py      <--- Blueprint location
            decorators.py    <--- Decorators
            errors.py        <--- Custom Error Handlers
            endpoints.py     <--- API endpoints
            models/
                __init__.py
                building.py
                user.py      <--- All validators and code
                meter.py          related to the model layer
                env.py
            templates/
                ...
            tests/
                ...
```
据此，重构views中的函数：
```python
@api.route('/users', methods=['POST'])
@limit(requests=10, window=60, by="ip")
@email
def create_user():
    data = request.get_json(force=True)
    new_user = user.create(json=data)
    session['uid'] = new_user['id']
    notify('admins', 'New User', repr(new_user))
    return Envelope(
        to_addr   = new_user['email'],
        subject   = api.config['WELCOME_EMAIL_SUBJECT'],
        text_body = render_template('emails/new_user.txt'),
        html_body = render_template('emails/new_user.html'))

```
实际上API设计中有一些重复的操作，这些操作可以讲其抽象出来写成一个装饰器。
#### 闷声发大财——装饰器
其中email装饰器的实现，这个装饰器用于生成user发送邮件：
```python
def email(f):
    @functools.wraps(f)
    def wrapped(*args, **kwargs):
        envelope = f(*args, **kwargs)
        envelope.from_addr = api.config['SYSTEM_EMAIL']
        def task():
            smtp().send(envelope)
        gevent.spawn(task)
        return jsonify({"status": "OK"})
    return wrapped
```
接下来是limit装饰器，在limit装饰器有几个定义：
* requests 在一个时间窗口内最多允许多少次请求
* window 一个时间窗口是多少秒
* by 一个用来去用户区分信息的key ID
* group(可选) 定义一组有相公速度限制的装饰器

limit装饰器实现起来如下：
```python
def limit(requests=100, window=60, by="ip", group=None):
    if not callable(by):
        by = { 'ip': lambda: request.headers.remote_addr }[by]

    def decorator(f):
        @functools.wraps(f)
        def wrapped(*args, **kwargs):
            group = group or request.endpoint
            key = ":".join(["rl", group, by()])

            try:
                remaining = requests - int(redis.get(key))
            except (ValueError, TypeError):
                remaining = requests
                redis.set(key, 0)

            ttl = redis.ttl(key)
            if not ttl:
                redis.expire(key, window)
                ttl = window

            g.view_limits = (requests,remaining-1,time()+ttl)

            if remaining > 0:
                redis.incr(key, 1)
                return f(*args, **kwargs)
            else:
                return Response("Too Many Requests", 429)
        return wrapped
    return decorator
```
既然有请求次数限制，那么也就必须在返回的headers信息告知client，因此，要在每次请求最后添加这些信息，如下:
```python
@api.after_request
def inject_rate_limit_headers(response):
    try:
        requests, remaining, reset = map(int, g.view_limits)
    except (AttributeError, ValueError):
        return response
    else:
        h = response.headers
        h.add('X-RateLimit-Remaining', remaining)
        h.add('X-RateLimit-Limit', requests)
        h.add('X-RateLimit-Reset', reset)
        return response
```
对于一些请求需要花比较诚时间的，可以异步来执行，比如：
```python
@api.route('/reports/power/monthly/<int:year>/<int:month>')
@limit(requests=1, by='ip')
@background
def report_monthly_power(year, month):
    rdata = generate_power_report(year, month)
    return rdata
```
但generate_power_report执行的时间很长，显然有改进空间。对此我们可以将耗时操作抽象诚一个backgroud装饰器:
```python
def background(f):
    @functools.wraps(f)
    def wrapper(*args, **kwargs):
        jobid = uuid4().hex
        key = 'job-{0}'.format(jobid)
        skey = 'job-{0}-status'.format(jobid)
        expire_time = 3600
        redis.set(skey, 202)
        redis.expire(skey, expire_time)

        @copy_current_request_context
        def task():
            try:
                data = f(*args, **kwargs)
            except:
                redis.set(skey, 500)
            else:
                redis.set(skey, 200)
                redis.set(key, data)
                redis.expire(key, expire_time)
            redis.expire(skey, expire_time)

        gevent.spawn(task)
        return jsonify(**{"job": jobid})
    return wrapper
```
#### 已经决定了，这样来处理Error
假如我们在models/users.py中定义了一个异常：
```python
class ValidationError(Exception):
    def __init__(self, field, message):
        self.field = field
        self.message = message
```
那么，在错误处理中
```python
# errors.py
@api.errorhandler(user.ValidationError)
def handle_user_validation_error(error):
    response = jsonify(**{
        'msg': error.message,
        'type': 'validation',
        'field': error.field })
    response.status_code = 400
    return response
```
而不是在每一个视图中重复的去做错误处理。
#### 一些微小的总结
1. 将API中相同的处理模式抽象成一个装饰器
2. 不要在视图入口写臭脚布逻辑处理
3. 将一些任务异步执行，可以利用gevent及celery等等
4. 冒泡处理错误
5. 不要忘记发掘框架内置的功能

[原slide地址](http://pycoder.net/bospy/presentation.html)
#### API用户认证
##### 基于用户密令认证
RESTful服务是无状态的，当一些API需要授权后才能访问的，而API又不能把登录或验证信息放在cookie中，只能通过HTTP认证发送
密令，将密令包含在请求Authorization中。Flask的Flask-HTTPAuth扩展已经提供了一个方便的实现。也就是相当于每次调用API，都必须用密令登录认证。
##### 基于令牌的认证
每次请求时,客户端都要发送认证密令。为了避免总是发送敏感信息,我们可以提供一种
基于令牌的认证方案。
使用基于令牌的认证方案时,客户端要先把登录密令发送给一个特殊的 URL,从而生成认证令牌。一旦客户端获得令牌,就可用令牌代替登录密令认证请求。出于安全考虑,令牌有过期时间。令牌过期后,客户端必须重新发送登录密令以生成新令牌。令牌落入他人之手所带来的安全隐患受限于令牌的短暂使用期限。为了生成和验证认证令牌,我们要在User模型中定义两个新方法。这两个新方法用itsdangerous 包
```python
class User(db.Model):
# ...
def generate_auth_token(self, expiration):
	s = Serializer(current_app.config['SECRET_KEY'],
	expires_in=expiration)
	return s.dumps({'id': self.id})
    
@staticmethod
def verify_auth_token(token):
	s = Serializer(current_app.config['SECRET_KEY'])
	try:
		data = s.loads(token)
	except:
		return None
	return User.query.get(data['id'])
```
