---
title: 关于falsk-sqlalchemy中多数据库配置及使用
date: 2024-08-02 20:11:36
tags: flask python TiDB Postgre SQLite
---

近期发现了几个免费的数据库，想着尝试用Flask进行简单连接  
以前一直使用SQLite，近期打算尝试连接MySQL和Postgre  
一些简单的安装等命令就此略过
### 多数据源配置
```python
    # 配置SQLite数据源
    sqlite_path = os.path.join(BASE_DIR, "sys.db")
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + sqlite_path
    # 配置MySQL（TiDB）数据源
    MYSQL_USERNAME = 'USERNAME'
    MYSQL_PASSWORD = 'PASSWORD'
    MYSQL_URL = 'SQL_URL'
    MYSQL_PORT = 'PORT'
    MYSQL_DATABASE = 'main'
    MYSQL_SSL = {
        'ssl_verify_cert': 'True',
        'ssl_verify_identity': 'True',
        'ssl_cert': BASE_DIR + '/ca/cert.crt',
        'ssl_key': BASE_DIR + '/ca/private.key'
    }
    MYSQL_URL_ALL = 'mysql+pymysql://%s:%s@%s:%s/%s?%s' % (
        MYSQL_USERNAME, MYSQL_PASSWORD, MYSQL_URL, MYSQL_PORT, MYSQL_DATABASE, urlencode(MYSQL_SSL))
    # 配置Postgre数据源
    PG_USERNAME = 'USERNAME'
    PG_PASSWORD = 'PASSWORD'
    PG_URL = 'SQL_URL'
    PG_PORT = 'PORT'
    PG_DATABASE = 'main'
    PG_URL_ALL = 'postgresql://%s:%s@%s:%s/%s' % (PG_USERNAME, PG_PASSWORD, PG_URL, PG_PORT, PG_DATABASE)
    app.config['SQLALCHEMY_BINDS'] = {
        'tidb': MYSQL_URL_ALL,
        'pg': PG_URL_ALL,
    }

```

#### 注意点
1、因为TiDB需要SSL认证，所以加入了SSL=True的条件，可根据需要自行设置，关于证书生成如下，共计生成3个文件
```Shell
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
openssl req -new -key private.pem -out cert.csr
openssl x509 -req -days 365 -in cert.csr -signkey private.pem -out cert.crt
```
### Model中的配置区别
1、SQLite中的使用，和日常没区别
```Python
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True)
    email = db.Column(db.String(120), unique=True)

    def __init__(self, username, email):
        self.username = username
        self.email = email

    def __repr__(self):
        return '<User %r>' % self.username
```
2、MySQL中的使用，需要添加BINDS相关信息，用于区分数据源，tidb为前期配置文件内的值
```Python
class Good(db.Model):
    __bind_key__ = 'tidb'
    id = db.Column(db.Integer, primary_key=True)
    good = db.Column(db.String(80), unique=True)
    email = db.Column(db.String(120), unique=True)

    def __init__(self, good, email):
        self.good = good
        self.email = email

    def __repr__(self):
        return '<Good %r>' % self.good
```
3、Postgre中的使用，同样需要添加BINDS信息   
区分MYSQL，PG库中有一个schema概念，若未设置对应模式，则默认为public下的表，如下展示为设置test模式下的表，也增加了tablename的示例
```Python
class Name(db.Model):
    __bind_key__ = 'pg'
    __table_args__ = ({"schema": "test"})
    __tablename__ = 'name'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(80), unique=True)
    email = db.Column(db.String(120), unique=True)

    def __init__(self, name, email):
        self.name = name
        self.email = email

    def __repr__(self):
        return '<Name %r>' % self.name
```

### 最终程序测试
经过此配置，则可通过此种方式进行测试
```Python
    # SQLite
    ur = User('admin', 'admin@example.com')
    db.session.add(ur)
    #TiDB
    gd = Good('good', 'good@example.com')
    db.session.add(gd)
    # PG
    ne = Name('name', 'name@example.com')
    db.session.add(ne)
    db.session.commit()
```
