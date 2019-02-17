.. _quickstart:

빨리 해 보기
============

.. currentmodule:: flask_sqlalchemy

Flask-SQLAlchemy는 재밌게 사용할 수 있으며 기본적인 응용에서는
굉장히 쉬우면서 큰 응용으로도 손쉽게 확장된다. 상세한 안내는
:class:`SQLAlchemy` 클래스 API 문서를 살펴보면 된다.

가장 작은 응용
--------------

플라스크 응용이 하나 있는 흔한 경우에서 해 줘야 할 건 플라스크
응용을 하나 만들고 원하는 설정을 올린 다음 응용을 인자로 줘서
:class:`SQLAlchemy` 객체를 만드는 것이다.

그렇게 만든 객체에는 :mod:`sqlalchemy` 와 :mod:`sqlalchemy.orm`
모두의 함수와 헬퍼들이 모두 들어 있다. 또 ``Model`` 이라는
클래스를 제공하는데, 이 선언적 기반 클래스를 이용해 모델을
선언할 수 있다. ::

    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy

    app = Flask(__name__)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test.db'
    db = SQLAlchemy(app)


    class User(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        username = db.Column(db.String(80), unique=True, nullable=False)
        email = db.Column(db.String(120), unique=True, nullable=False)

        def __repr__(self):
            return '<User %r>' % self.username

초기 데이터베이스 생성을 하려면 대화형 파이썬 셸에서 ``db``
객체를 임포트 해서 :meth:`SQLAlchemy.create_all` 메소드를
실행하기만 하면 테이블과 데이터베이스가 만들어진다. ::

    >>> from yourapplication import db
    >>> db.create_all()

짜잔! 데이터베이스가 생겼다. 그럼 사용자를 좀 만들어 보자. ::

    >>> from yourapplication import User
    >>> admin = User(username='admin', email='admin@example.com')
    >>> guest = User(username='guest', email='guest@example.com')

아직은 데이터베이스 안에 들어가 있지 않다. 들어가게 해 보자. ::

    >>> db.session.add(admin)
    >>> db.session.add(guest)
    >>> db.session.commit()

데이터베이스의 데이터에 접근하는 것도 누워서 떡 먹기다. ::

    >>> User.query.all()
    [<User u'admin'>, <User u'guest'>]
    >>> User.query.filter_by(username='admin').first()
    <User u'admin'>

근데 ``User`` 클래스에 ``__init__`` 메소드를 정의해 준 적이
없지 않은가? 왜냐면 SQLAlchemy에서 컬럼과 릴레이션 전체에
대한 키워드 인자를 받는 생성자를 모든 모델 클래스에 묵시적으로
추가해 주기 때문이다. 만일 어떤 이유로 그 생성자를 오버라이드
하기로 했다면 계속 ``**kwargs`` 를 받도록 하고 그
``**kwargs`` 로 상위 생성자를 호출해야 위 동작이 유지된다. ::

    class Foo(db.Model):
        # ...
        def __init__(**kwargs):
            super(Foo, self).__init__(**kwargs)
            # 기타 작업

간단한 관계
-----------

SQLAlchemy는 관계형 데이터베이스에 연결하는 것이고 관계형
데이터베이스가 잘 하는 건 관계를 다루는 일이다. 그러니 서로 관계
있는 두 테이블을 사용하는 응용을 예로 들어 보자. ::

    from datetime import datetime


    class Post(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        title = db.Column(db.String(80), nullable=False)
        body = db.Column(db.Text, nullable=False)
        pub_date = db.Column(db.DateTime, nullable=False,
            default=datetime.utcnow)

        category_id = db.Column(db.Integer, db.ForeignKey('category.id'),
            nullable=False)
        category = db.relationship('Category',
            backref=db.backref('posts', lazy=True))

        def __repr__(self):
            return '<Post %r>' % self.title


    class Category(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        name = db.Column(db.String(50), nullable=False)

        def __repr__(self):
            return '<Category %r>' % self.name

일단 객체들을 좀 만들자. ::

    >>> py = Category(name='Python')
    >>> Post(title='Hello Python!', body='Python is pretty cool', category=py)
    >>> p = Post(title='Snakes', body='Ssssssss')
    >>> py.posts.append(p)
    >>> db.session.add(py)

보다시피 ``Post`` 객체를 세션에 추가해 줄 필요가 없다.
``Category`` 가 세션에 포함돼 있으므로 릴레이션을 통해 연결된
모든 객체들이 함께 추가된다.
:meth:`db.session_add() <sqlalchemy.orm.session.Session.add>`
를 그 객체들을 생성하기 전에 호출하는지 후에 호출하는지는
상관없다. 또 릴레이션의 어느 쪽에서도 연결이 가능하다. 그래서
카테고리를 써서 게시 글을 만들 수도 있고 카테고리별 목록에
게시 글을 추가할 수도 있다.

글들을 보자. 관계를 게으르게 적재하기 때문에 접근을 해야
데이터베이스에서 읽어 온다. 하지만 아마 차이를 느끼지 못할
것이다. 목록을 가져오는 게 꽤 빠르다. ::

    >>> py.posts
    [<Post 'Hello Python!'>, <Post 'Snakes'>]

관계를 게으르게 적재하는 게 빠르기는 하지만 많은 객체들에
루프를 돌면서 추가 질의를 유발시키게 되면 금방 주요 병목이
될 수 있다. 이런 경우에 SQLAlchemy에서는 질의 수준에서 적재
방식을 바꿀 수 있다. 질의 한 번으로 카테고리와 그 게시 글
전부를 가져오고 싶다면 다음처럼 하면 된다. ::

    >>> from sqlalchemy.orm import joinedload
    >>> query = Category.query.options(joinedload('posts'))
    >>> for category in query:
    ...     print category, category.posts
    <Category u'Python'> [<Post u'Hello Python!'>, <Post u'Snakes'>]


그 관계에 대한 질의 객체를 얻고 싶으면
:meth:`~sqlalchemy.orm.query.Query.with_parent` 를 이용하면
된다. 예를 들어 뱀에 대한 글을 제외해 보자. ::

    >>> Post.query.with_parent(py).filter(Post.title != 'Snakes').all()
    [<Post 'Hello Python!'>]


깨달음으로 향한 길
------------------

SQLAlchemy만 쓰는 것과 비교해 알아 둬야 하는 것은 얼마 되지 않는다.

1.  :class:`SQLAlchemy` 를 통해 다음에 접근할 수 있다.

    -   :mod:`sqlalchemy` 및 :mod:`sqlalchemy.orm` 의 모든
        함수와 클래스
    -   ``session`` 이라는 미리 구성된 스코프 있는 세션
    -   :attr:`~SQLAlchemy.metadata`
    -   :attr:`~SQLAlchemy.engine`
    -   모델에 따라서 테이블을 만들고 없앨 수 있는
        :meth:`SQLAlchemy.create_all` 및 :meth:`SQLAlchemy.drop_all`
        메소드
    -   미리 구성된 선언적 기반 클래스인 :class:`Model` 클래스

2.  선언적 기반 클래스 :class:`Model` 은 일반 파이썬 클래스처럼
    동작하되 딸려 있는 ``query`` 속성을 사용해 모델을 질의할 수
    있다. (:class:`Model` 및 :class:`BaseQuery` 참고.)

3.  세션 커밋은 직접 해야 하지만 요청 마지막에서 제거해 줄 필요는
    없다. Flask-SQLAlchemy가 대신 해 준다.
