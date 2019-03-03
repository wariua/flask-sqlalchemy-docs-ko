.. _contexts:

.. currentmodule:: flask_sqlalchemy

문맥 소개
=========

응용을 한 개만 사용할 계획이라면 이 장은 대강 건너뛰어도 된다.
:class:`SQLAlchemy` 생성자에 응용을 건네 주기만 하면 일반적으로
끝이다. 하지만 여러 응용을 사용하거나 응용을 함수에서 동적으로
생성하고 싶다면 계속 읽어 나가면 된다.

응용은 함수 안에서 정의하지만 :class:`SQLAlchemy` 객체는
전역이라면 어떻게 응용에 대해 알려 줄 수 있을까? 답은
:meth:`~SQLAlchemy.init_app` 함수를 쓰는 것이다. ::

    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy

    db = SQLAlchemy()

    def create_app():
        app = Flask(__name__)
        db.init_app(app)
        return app


이렇게 하면 응용에서 :class:`SQLAlchemy` 를 쓸 수 있도록 준비
작업을 해 준다. 하지만 여기선 :class:`SQLAlchemy` 객체를
응용에 결속하지 않는다. 왜일까? 응용을 여러 개 생성할 수도
있기 때문이다.

그럼 :class:`SQLAlchemy` 는 어떻게 응용에 대해 알게 될까? 응용
문맥을 구성해 줘야 한다. 플라스크 뷰 함수나 CLI 명령 안에서
작업하는 경우에는 그게 자동으로 이뤄진다. 하지만 대화형 셸
안에서 작업하고 있다면 직접 해 줘야 한다. (`응용 문맥 만들기
<http://flask.pocoo.org/docs/appcontext/#creating-an-application-context>`_
참고.)

응용 문맥 밖에서 데이터베이스 동작을 수행하려고 하면 다음
오류를 보게 된다.

    No application found. Either work inside a view function or push an
    application context.

간단히 말해 다음처럼 해 주면 된다.

>>> from yourapp import create_app
>>> app = create_app()
>>> app.app_context().push()

또는 with 문을 써서 구성에서 해제까지 신경 써 줄 수도 있다. ::

    def my_function():
        with app.app_context():
            user = db.User(...)
            db.session.add(user)
            db.session.commit()

Flask-SQLAlchemy 내의 일부 함수들은 동작 대상이 되는 응용을
선택적으로 받기도 한다.

>>> from yourapp import db, create_app
>>> db.create_all(app=create_app())
