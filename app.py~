import pydantic
from flask import Flask, jsonify, request
from flask.views import MethodView
from sqlalchemy.exc import IntegrityError

from models import Session, Advert, User
from schema import CreateAdvert, CreateUser

app = Flask('adverts_app')


class HttpError(Exception):
    def __init__(self, status_code: int, description: str):
        self.status_code = status_code
        self.description = description


@app.errorhandler(HttpError)
def error_handler(error: HttpError):
    response = jsonify({'error': error.description})
    response.status_code = error.status_code
    return response


@app.before_request
def before_request():
    session = Session()
    request.session = session


@app.after_request
def after_request(response):
    request.session.close()
    return response


def get_advert_by_id(advert_id: int) -> Advert:
    advert = request.session.get(Advert, advert_id)
    if advert is None:
        raise HttpError(404, 'Advert not found')
    return advert


def validate(schema_class, json_data):
    try:
        return schema_class(**json_data).dict(exclude_unset=True)
    except pydantic.ValidationError as err:
        error = err.errors()[0]
        error.pop('ctx', None)
        raise HttpError(400, error)


def add_advert(advert: Advert):
    user = request.session.get(User, advert.owner_id)
    if user is None:
        raise HttpError(409, 'User (owner) doesn''t exists')
    try:
        request.session.add(advert)
        request.session.commit()
    except IntegrityError as err:
        raise HttpError(409, 'Advert already exists')
    return advert


def add_user(user: User):
    try:
        request.session.add(user)
        request.session.commit()
    except IntegrityError as err:
        raise HttpError(409, 'user already exists')
    return user


class AdvertView(MethodView):
    def get(self, advert_id: int):
        advert = get_advert_by_id(advert_id)
        return jsonify(advert.json)

    def post(self):
        json_data = validate(CreateAdvert, request.json)
        advert = Advert(**json_data)
        add_advert(advert)
        response = jsonify(advert.json)
        response.status_code = 201
        return response

    def delete(self, advert_id: int):
        advert = get_advert_by_id(advert_id)
        request.session.delete(advert)
        request.session.commit()
        return jsonify({'status': 'success'})


class UserView(MethodView):
    def get(self, user_id: int):
        user = request.session.get(User, user_id)
        if user is None:
            raise HttpError(404, "user not found")
        return jsonify(user.json)

    def post(self):
        json_data = validate(CreateUser, request.json)
        user = User(**json_data)
        add_user(user)
        response = jsonify(user.json)
        response.status_code = 201
        return response


advert_view = AdvertView.as_view('advert_view')
user_view = UserView.as_view('user_view')

app.add_url_rule(
    '/advert',
    view_func=advert_view,
    methods=[
        'POST',
    ]
)

app.add_url_rule(
    '/advert/<int:advert_id>',
    view_func=advert_view,
    methods=[
        'GET',
        'DELETE'
    ]
)

app.add_url_rule(
    '/user',
    view_func=user_view,
    methods=[
        'POST',
    ],
)
app.add_url_rule(
    '/user/<int:user_id>',
    view_func=user_view,
    methods=[
        'GET'
    ]
)

app.run()