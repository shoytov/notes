# DRF captcha своими руками

## Вступление.
Пришла мне тут по работе задача сделать возможность добавления товара в корзину пользователем без авторизации. Корзина, причем должна храниться на бэке для того, чтобы можно было проводить аналитику по неоформленным заказам, а также, если пользователь авторизуется, то добавлять эту корзину к его профилю для доступности ее с любых других устройств, на которых он (пользователь авторизован). 
Ясное дело, что для корзины одного пользователя в сеансе оформления заказа нужен уникальный идентификатор, по которому можно определить в какую “корзину” поместить товар. 
Так как на проекте мы использует REST подход к проектированию API, я подумал, что можно генерировать `uuid` ключ на клиенте и передавать его при каждом запросе добавления товара в корзину.
Да, все бы хорошо, но возникает потенциальная проблема паразитных запросов от недоброжелателей. Да, безусловно, есть куча всяких способов защититься от троттлинга, но все это кажется мне в этой ситуации не совсем уместным. Поэтому, я решил, что лучше уникальный идентификатор генерировать на бэкенде, а выдавать его при прохождении **CAPTCHA**.

### Поиск готовых решений.
На проекте используется Django 5.x и Django Rest Framework. Я начал поиск готовых библиотек, подходящих под мою задачу, а именно, что должно быть в готовом решении:
- хэндлер get-запроса, который возвращает `captcha_id` и закодированное в base64 изображение этой самой капчи;
- хэндлер post-запроса, который вальсирует введенные пользователем данные с картинки. В идеале, чтобы в хендлере был вызов некого метода, который возвращает True/False в зависимости от результатов валидации, введенных пользователем данных. Это было бы очень удобно для расширения логики.

Казалось бы, что там особенного, наверняка есть решения. Начал искать. Попались на глаза:
- Django REST framework reCAPTCHA (не подходит из-за использования стороннего сервиса)
- Django YaCaptcha (не подходит по аналогичной причине)
- Django rest captcha (не удалось попробовать ввиду того, что библиотека не обновилась больше 2-х лет и не поддерживает Django 5.x)
- Django Simple Captcha (не подошло по текущей бизнес-логики + посмотрел код на GitHub и мне не понравилось, ну, что поделать - вкусы у всех разные)

Ввиду того, что ничего из того, что я нашел поиском в интернете, мне не подошло, было принято решение сделать все самому. Задача не выглядит как Rocket Science, да и кто из нас не любит строить свой велосипед, так что погнали...

## Реализация
### Описание логики работы
Упрощенно разобьем логику работы на 2 этапа:
- get-запрос на получение картинки капчи. На этом шаге мы генерируем 2 значения: `captcha_id`, `captcha_value`. Записываем эти значения в какое-нибудь хранилище (Postgres, Redis и т.д.) для дальнейшей проверки ввода пользователя. Так как у нас REST, то ответом будет json, который будет иметь 2 поля: `captcha_id` и `image` - картинка капчи, закодированная в base64.
- post-запрос для валидации данных капчи, введенных пользователем. Передаваемые от пользователя данные: `captcha_id`, `captcha_value`. Получаем из хранилища данных, куда до этого мы поместили капчу, данные по `captcha_id` . Сравниваем переданные пользователем `captcha_value` со значением, которое мы получили из хранилища данных. Если значения совпадают - пользователь ввел капчу правильно, то выдаем юзеру сгенерированный uuid-токен (тут я сразу буду описывать такую логику, так как в моей задаче нужно именно такое поведение). Если капча введена правильно, удаляем ее данные из хранилища данных (также по-хорошему надо ограничить время жизни капчи в хранилище по TTL). 

### Программируем
Для начала создадим новое Django-приложение, или, по-нашему, пакет: ```python manage.py startapp captcha```
Сразу добавим его в `INSTALLED_APPS ` в `settings.py`:

```
INSTALLED_APPS = [
...
"captcha",
]
```

Также в настройки добавим: 
- переменную, отвечающую за длину капчи; 
- переменную для префикса капчи в кэше;
- переменную, определяющую время жизни капчи в кэше;
- переменную, задающую размер изображения капчи.

```
CAPTCHA_LENGTH = env.int("CAPTCHA_LENGTH", default=4)
CAPTCHA_CACHE_PREFIX = env.str("CAPTCHA_CACHE_PREFIX", default="captcha_")
CAPTCHA_CACHE_TTL = env.int("CAPTCHA_CACHE_TTL", default=120)
CAPTCHA_IMAGE_SIZE = env.tuple("CAPTCHA_IMAGE_SIZE", default=(90, 40))
```

Сделаем пакет с утилитами, в нем создадим модуль для генерации текста для капчи:

```
import random

from django.conf import settings


def generate_captcha_value(length: int = settings.CAPTCHA_LENGTH) -> str:
    symbols = "abcdefghijklmnopqrstuvwxyz1234567890"
    res = []

    for _ in range(length):
        res.append(random.choice(symbols))
    result = "".join(res)

    return result.upper()
```

Создадим модель Captcha в файле `models.py`

```
import uuid

from django.db import models
from django.utils import timezone

from captcha.utils.generate import generate_captcha_value


class CaptchaModel(models.Model):
    captcha_id = models.UUIDField(default=uuid.uuid4)
    code = models.CharField(max_length=6, default=generate_captcha_value)
    created_at = models.DateTimeField(default=timezone.now)

    class Meta:
        managed = False
```

Здесь я использую `managed = False` для того, чтобы таблица в БД не создавалась, так как данные капчи буду хранить в Redis. Далее я переопределю метод `save` модели для сохранения данных именно в Redis.

Добавим импорт:

```
from django.core.cache import cache
```

Сам метод `save`:

```
def save(self, *args, force_insert=False, force_update=False, using=None, update_fields=None):
    cache.set(
        f"{settings.CAPTCHA_CACHE_PREFIX}{self.captcha_id}",
        self.code,
        settings.CAPTCHA_CACHE_TTL,
    )
```

Теперь займемся генерацией изображения с кодом капчи.
Эту логику вынесем в сервисный слой, для этого создадим пакет `services` в корне пакета `captcha`. Далее, в пакете `services` создадим модуль `captcha_service.py` и начнем писать сам сервис:

```
from django.conf import settings
from PIL import Image, ImageDraw, ImageFont

from captcha.models import CaptchaModel


class CaptchaService:
    def __init__(self):
        self.captcha = CaptchaModel()
        self.captcha.save()

        self.font = ImageFont.truetype(settings.CAPTCHA_FONT_PATH, settings.CAPTCHA_FONT_SIZE)
``` 

Для отображения текста на картинке будем использовать шрифт `Vera.ttf` [https://github.com/mps/fonts/blob/master/Vera.ttf](https://github.com/mps/fonts/blob/master/Vera.ttf). Скачаем этот шрифт и поместим в папку `fonts` внутри пакета `captcha`.
Добавим в `settings.py` нашего проекта путь к этому шрифту и размер шрифта:

```
CAPTCHA_FONT_PATH = os.path.join(BASE_DIR, "captcha/fonts/Vera.ttf")
CAPTCHA_FONT_SIZE = env.int("CAPTCHA_FONT_SIZE", default=22)
```

Теперь давайте добавим метод для генерации изображения в `CaptchaService`:

```
def _generate_image(self) -> Image.Image:
    image = Image.new("RGB", settings.CAPTCHA_IMAGE_SIZE, (128, 213, 215))
    draw = ImageDraw.Draw(image)
    text_width, text_height = self.font.getmask(self.captcha.code).size  # type: ignore
    draw_points = (
        (settings.CAPTCHA_IMAGE_SIZE[0] - text_width) // 2,
        (settings.CAPTCHA_IMAGE_SIZE[1] - text_height) // 2,
    )
    draw.text(
        draw_points,
        self.captcha.code,  # type: ignore
        font=self.font,
        fill=(255, 255, 255),
    )
    image = image.filter(ImageFilter.GaussianBlur(1.5))

    return image
```

Этот код генерирует объект Image.Image с заданными размерами, добавляет текст из капчи и добавляет размытие по Гауссу для осложнения автоматического распознавания текста, во всяком случае MacOS из коробки (стандартный просмотрщик умеет копировать текст с изображения) не смогла скопировать текст с капчи с такими настройками цвета фона и степени размытия.

Теперь надо сделать кодирование изображения в `base64` для дальнейшей отправки клиенту (фронтэнду).

Добавим такой метод в `CaptchaService`:

```
def base_64_captcha_image(self) -> str:
    image = self._generate_image()
    buffered = BytesIO()
    image.save(buffered, format="PNG")
    return base64.b64encode(buffered.getvalue()).decode("utf-8")
```

Генерация капчи готова! Теперь наша задача сделать валидацию введенных пользователем данных. Для этих целей создадим еще один модуль в сервисном слое, назовем этот модуль `validate_captcha.py`. В этом модуле создадим сервис:

```
from django.conf import settings
from django.core.cache import cache


class ValidateCaptchaService:
    def __init__(self, code: str, captcha_id: str):
        self.code = code
        self.captcha_id = captcha_id
        
    def validate(self) -> bool:
        return (
            cache.get(f"{settings.CAPTCHA_CACHE_PREFIX}{self.captcha_id}", "") == self.code.upper()
        )
```

На этом основная работа закончена. Теперь создадим APIView для этих действий. 
Но прежде сделаем сериализатор, который будет моделью ответа на запрос капчи. Создадим пакет `serializers` в корне пакета `captcha`. Там создадим модуль `captcha_serializers.py`.
Добавим код сериализатора:

```
from rest_framework import serializers


class CaptchaSerializer(serializers.Serializer):
    captcha_id = serializers.CharField()
    image = serializers.CharField()
    created_at = serializers.DateTimeField()
```

В модуле `views.py` пакета `captcha` напишем следующий код:

```
from drf_spectacular.utils import extend_schema
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView

from captcha.serializers.captcha_serializers import CaptchaSerializer
from captcha.services.captcha_service import CaptchaService
from main.serializers.common_serializers import BadRequestSerializer


class CaptchAPIView(APIView):
    @extend_schema(
        description=("Запрос на получение Captcha"),
        responses={
            status.HTTP_200_OK: CaptchaSerializer,
            status.HTTP_400_BAD_REQUEST: BadRequestSerializer,
        },
        tags=["captcha"],
    )
    def get(self, request) -> Response:
        service = CaptchaService()
        serializer = CaptchaSerializer(
            data={
                "captcha_id": service.captcha.captcha_id,
                "created_at": service.captcha.created_at,
                "image": service.base_64_captcha_image(),
            }
        )

        if serializer.is_valid():
            return Response(serializer.data, status=status.HTTP_200_OK)
        return Response(
            BadRequestSerializer({"detail": serializer.errors}).data,
            status=status.HTTP_400_BAD_REQUEST,
        )
```

Здесь `extend_schema` - декоратор, который выводит описание метода в `Swagger`. Есть несколько известных решений `swagger` для `DRF`, но я в последнее время использую `drf_spectacular `. Кому-то может показаться слишком избыточно описывать методы при помощи декораторов, но я не вижу в этом ничего криминального. Конечно, в `FastAPI` все из коробки работает очень элегантно, но  я уже смирился, что в `DRF` пока такого и близко нет и приходится использовать специальные для этого библиотеки. Кстати про `Swagger` и `DRF`, я вот такие еще использовал:
- drf-yasg [https://drf-yasg.readthedocs.io/en/stable/readme.html](https://drf-yasg.readthedocs.io/en/stable/readme.html)
- django-rest-swagger [https://django-rest-swagger.readthedocs.io/en/latest/](https://django-rest-swagger.readthedocs.io/en/latest/) - сейчас уже неактуален, так как проект не обновлялся с 2021 года

Вот код `BadRequestSerializer`, который возвращается в случае ошибок:

```
from rest_framework import serializers


class BadRequestSerializer(serializers.Serializer):
    detail = serializers.CharField()
```

Вообще, считается хорошей практикой создавать свои кастомные исключения `Exceptions` и свои стандартные наборы классов для работы с ошибками.

Ладно, продолжим. Создадим модуль `urls.py` в корне пакета `captcha` вот с таким содержимым:

```
from django.urls import path

from captcha.views import CaptchAPIView

urlpatterns = [
    path("captcha/", CaptchAPIView.as_view(), name="captcha"),
]
```

В модуле `urls.py` в основном пакете проекта добавим этот код:

```
from django.urls import include

urlpatterns = [
...
path("captcha/", include("captcha.urls")),
]
```

Итак, осталось сделать только валидацию данных капчи, введенных пользователем. Для этого нам потребуется сериализатор входных данных и сериализатор успешного ответа для хэндлера post-запроса.

Добавим в модуль `captcha_serializers.py` следующее:

```
class CaptchUserInputSerializer(serializers.Serializer):
    code = serializers.CharField()
    captcha_id = serializers.CharField()


class CaptchResponseSerializer(serializers.Serializer):
    session_token = serializers.CharField()
```

Осталось совсем немного - добавить хэндлер post-запроса в класс `CaptchAPIView` модуля `views.py` пакета `captcha`:

```
@extend_schema(
    description=("Валидация капчи. Если капча валидна, то возвращается session_token"),
    request=CaptchUserInputSerializer,
    responses={
        status.HTTP_200_OK: CaptchResponseSerializer,
        status.HTTP_400_BAD_REQUEST: BadRequestSerializer,
    },
    tags=["captcha"],
)
def post(self, request) -> Response:
    service = ValidateCaptchaService(request.data["code"], request.data["captcha_id"])

    if service.validate() is True:
        serializer = CaptchResponseSerializer(data={"session_token": str(uuid.uuid4())})
        if serializer.is_valid():
            return Response(serializer.data, status=status.HTTP_200_OK)
        return Response(
            BadRequestSerializer({"detail": serializer.errors}).data,
            status=status.HTTP_400_BAD_REQUEST,
        )
    return Response(
        BadRequestSerializer({"detail": "Invalid captcha"}).data,
        status=status.HTTP_400_BAD_REQUEST,
    )
```

В последнем листинге не стал добавлять импорты, так как, мне кажется, тут все очевидно.

На этом все, вот такой простенький велосипед мы сегодня нагородили, будем продвигать велосипедостроение ;)