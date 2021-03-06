---
title: Сборка образа
permalink: django/100_basic/10_build.html
---

{% filesused title="Файлы, упомянутые в главе" %}
- Dockerfile
- werf.yaml
{% endfilesused %}

Классическим способом сборки является использование [Dockerfile](https://docs.docker.com/engine/reference/builder/). Возможно, в ваших приложениях уже реализована сборка с помощью этого механизма, поэтому мы начнём с него, а затем подключим его в werf. В следующих главах мы ускорим сборку, воспользовавшись альтернативным синтаксисом описания сборки, а сейчас — сфокусируемся на быстром получении результата.

## Подготовка рабочего места

Мы предполагаем, что вы уже [установили werf]({{ site.docsurl }}/installation.html) и Docker.

Создайте директорию на своём компьютере и выполните там следующие шаги:

```shell
git clone https://github.com/werf/werf-guides.git
cp -r werf-guides/examples/django/000_app ./
cd 000_app 
git init
git add .
git commit -m "initial commit"
```

_Так вы скопируете себе код [приложения на Django](https://github.com/werf/werf-guides/tree/master/examples/django/000_app) и инициируете Git в каталоге с ним._

werf следует принципам [гитерминизма]({{ site.docsurl }}/documentation/advanced/configuration/giterminism.html): опирается на состояние, описанное в Git-репозитории. Это означает, что файлы, не коммитнутые в Git-репозиторий, по умолчанию будут игнорироваться. Благодаря этому, имея исходные коды приложения, вы всегда можете реализовать его конкретное работоспособное состояние.

## Реализация сборки в Dockerfile

Конфигурация сборки нашего приложения будет состоять из следующих шагов:

- взять базовый образ с Python3 (`python:3.9.1-alpine3.12` подойдёт);
- добавить в него код приложения;
- установить зависимости npm.

Реализуем это в `Dockerfile`:

{% snippetcut name="Dockerfile" url="https://github.com/werf/werf-guides/blob/master/examples/django/010_build/Dockerfile" %}
{% raw %}
```Dockerfile
FROM python:3.9.1-alpine3.12
WORKDIR /app
COPY project /app

RUN pip3 install -r requirements.txt

ENTRYPOINT ["python3", "manage.py"]
CMD ["runserver"]
```
{% endraw %}
{% endsnippetcut %}

## Интеграция Dockerfile с werf

Подключим готовый Dockerfile к werf. Для этого, создадим в корне репозитория файл `werf.yaml`, описывающий сборку всего проекта:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/django/011_build_werf/werf.yaml" %}
{% raw %}
```yaml
project: werf-guided-django
configVersion: 1
---
image: basicapp
dockerfile: Dockerfile
```
{% endraw %}
{% endsnippetcut %}

{% offtopic title="Что тут написано?" %}

`werf.yaml` начинается с обязательной **секции мета-информации**:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/django/011_build_werf/werf.yaml" %}
{% raw %}
```yaml
project: werf-guided-django
configVersion: 1
```
{% endraw %}
{% endsnippetcut %}

Здесь:
- **_project_** — поле, задающее уникальное имя проекта приложения. Имя проекта по умолчанию используется при генерации имени Helm-релиза и пространства имен, `namespace`, в Kubernetes. Изменение имени у активного проекта затруднительно и требует ряда действий, которые необходимо выполнить вручную (подробнее о возможных последствиях можно прочитать [здесь]({{ site.docsurl }}/documentation/reference/werf_yaml.html#последствия-смены-имени-проекта));
- **_configVersion_** определяет версию синтаксиса, используемую в `werf.yaml` (на данный момент поддерживается только версия `1`).

Следующая секция конфигурации, которая и будет основной для сборки: [**image config section**]({{ site.docsurl }}/documentation/reference/werf_yaml.html#секция-image).

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/django/011_build_werf/werf.yaml" %}
{% raw %}
```yaml
---
image: basicapp
dockerfile: Dockerfile
```
{% endraw %}
{% endsnippetcut %}

В строке `image: basicapp` определяется идентификатор образа, который впоследствии может использоваться при конфигурации выката, а также для вызова команд werf для определённого образа из `werf.yaml` (к примеру, `werf build basicapp`, `werf run basicapp` и т.д.). 

Строка `dockerfile: Dockerfile` указывает, что сборочная конфигурация описана в существующем файле, расположенном по пути `Dockerfile`. 

Также доступны и другие директивы, с которыми можно ознакомиться [по ссылке]({{ site.docsurl }}/documentation/reference/werf_yaml.html#dockerfile-image-section-image).

В одном `werf.yaml` может быть описано произвольное количество образов.

{% endofftopic %}

## Сборка

После того, как мы добавили описанные выше файлы `Dockerfile` и `werf.yaml`, надо обязательно закоммитить изменения в Git:

{% raw %}
```shell
git add .
git commit -m "work in progress"
```
{% endraw %}

Для того, чтобы запустить сборку, воспользуемся [командой `build`]({{ site.docsurl }}/documentation/reference/cli/werf_build.html):

{% raw %}
```shell
werf build
```
{% endraw %}

_В подглаве «Ускорение сборки» мы переведём сборку с Dockerfile на альтернативный синтаксис werf под названием Stapel и получим расширенные возможности: инкрементальную пересборку с учетом истории Git, возможность применения Ansible, использование кэша между сборками, удобные инструменты диагностики и многое другое._

Но уже сейчас вы можете заметить, что werf делает расширенный вывод логов сборки:

```
┌ ⛵ image basicapp
│ ┌ Building stage basicapp/dockerfile
│ │ basicapp/dockerfile  Sending build context to Docker daemon  69.63kB
│ │ basicapp/dockerfile  Step 1/16 : FROM python:3.9.1-alpine3.12
│ │ basicapp/dockerfile   ---> b90fa0d7cbd1
│ │ basicapp/dockerfile  Step 2/16 : WORKDIR /app
│ │ basicapp/dockerfile   ---> Using cache
<..>
│ │ basicapp/dockerfile  Successfully built 02a0a425890a
│ │ basicapp/dockerfile  Successfully tagged a1cbf6dc-343f-4a77-b846-d0f12a700cb7:latest
│ ├ Info
│ │       name: werf-guided-django:a473b87e1ad65f102fa90f8c6647b03056e5ae95ff1ef3c5e7fd2c31-1605597979927
│ │       size: 953.1 MiB
│ └ Building stage basicapp/dockerfile (21.94 seconds)
└ ⛵ image basicapp (22.04 seconds)

Running time 22.07 seconds
```

## Запуск

Запустим собранный образ с помощью [werf run]({{ site.docsurl }}/documentation/cli/main/run.html):

```shell
werf run --docker-options="-d -p 8000:8000 --restart=always" -- runserver 0.0.0.0:8000
```

Обратите внимание, что мы задаем [параметры docker](https://docs.docker.com/engine/reference/run/) опцией `--docker-options`, а саму команду запуска указываем после двух дефисов.

_Вы также можете заметить, что и вызов `werf run` осуществляет сборку, т.е. предварительно запускать сборку на самом деле не обязательно._

Теперь приложение доступно локально на порту 8000:

![](/guides/images/django/100_10_app_in_browser.png)

## Внесение новых изменений

Мы будем постоянно дорабатывать приложение. Посмотрим, как это правильно делать на примере произвольных изменений в коде приложения:

{% snippetcut name="project/example/urls.py" url="#" %}
{% raw %}
```python
from django.urls import include, path
from django.http import HttpResponse
from rest_framework import routers
from . import views

router = routers.DefaultRouter()
router.register(r'labels', views.LabelViewSet)

urlpatterns = [
   path('', include(router.urls)),
   path('api-auth/', include('rest_framework.urls', namespace='rest_framework')),
   path('test', lambda x: HttpResponse('Our changes'))
]
```
{% endraw %}
{% endsnippetcut %}

 1. Остановите ранее запущенный `werf run` (нажав Ctrl+C в консоли, где он запущен).
 2. Запустите его заново: 
    ```shell
    werf run --docker-options="-d -p 8000:8000 --restart=always" -- runserver 0.0.0.0:8000
    ```
 3. Произошла ошибка:
    ```
    Error: phase build on image basicapp stage dockerfile handler failed: the file "project/example/urls.py" must be committed

    You might also be interested in developer mode (activated with --dev option) that allows you to work with staged changes without doing redundant commits. Just use "git add <file>..." to include the changes that should be used.

    To provide a strong guarantee of reproducibility, werf reads the configuration and build's context files from the project git repository and eliminates external dependencies. We strongly recommend to follow this approach. But if necessary, you can allow the reading of specific files directly from the file system and enable the features that require careful use. Read more about giterminism and how to manage it here: https://werf.io/v1.2-ea/documentation/advanced/configuration/giterminism.html.
    ```

В описанном сценарии **перед шагом 1 забыли сделать коммит** в Git.

{% offtopic title="А как правильно и зачем такие сложности?" %}
1. Внести изменения в код.
2. Сделать коммит:
   ```shell
   git add .
   git commit -m "wip"
   ```
3. Перезапустить `werf run`:
    ```shell
    werf run --docker-options="-d -p 8000:8000 --restart=always" -- runserver 0.0.0.0:8000
    ```
4. Посмотреть на результат в браузере.

Как мы уже упоминали в начале статьи, werf работает в режиме [гитерминизма]({{ site.docsurl }}/documentation/advanced/configuration/giterminism.html). Жёсткая связка с Git необходима для того, чтобы гарантировать воспроизводимость вашего решения. Подробнее о том, как работает эта механика _гитерминизма_, а также о режиме разработчика с флагом `--dev` мы расскажем в главе «Необходимо знать», а пока что — сфокусируемся на сборке и доставке до кластера.

{% endofftopic %}

<div id="go-forth-button">
    <go-forth url="20_cluster.html" label="Подготовка кластера" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
