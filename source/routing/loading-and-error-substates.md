Роутер Ember позволяет вам обеспечивать обратную связь, когда маршрут загружает данные, и когда при загрузке возникает ошибка.

## Подсостояния `loading`

Во время выполнения hooks `beforeModel`, `model` и `afterModel`  данным может потребоваться время для загрузки. Технически роутер приостанавливает переход, пока не завершатся обещания, которые возвращены из каждого hook.

Рассмотрим следующий пример:

`app/router.js`
```js
Router.map(function() {
  this.route('slow-model');
});
```

`app/routes/slow-model.js`
```js
export default Ember.Route.extend({
  model() {
    return this.store.findAll('slowModel');
  }
});
```

Если вы переходите в `slow-model`, в hook `model` запрос может занять много времени для завершения. В течение этого времени интерфейс пользователя не дает вам какой-либо обратной связи о происходящем. Если вы переходите по этому маршруту после полного обновления страницы, интерфейс пользователя будет абсолютно пустым, так как в действительности вы не совершили полный вход по какому-либо маршруту и еще не отобразили шаблоны. Если вы заходите в `slow-model` с другого маршрута, то будете видеть шаблоны из предыдущего маршрута, пока модель не закончит загрузку, а затем резко загрузятся все шаблоны для `slow-model`.

Как же можно обеспечить визуальную обратную связь во время перехода?

Просто определите шаблон под названием `loading` (и опционально соответствующий маршрут), к которому Ember будет переходить. Промежуточный переход в подсостояние загрузки происходит сразу (синхронно), URL не обновляется, и, в отличие от других переходов, текущий активный переход не сбрасывается.

Когда основной переход в `slow-model` завершается, программа покидает маршрут `loading`, и продолжается переход к `slow-model`:

Для вложенных маршрутов, это выглядит примерно так:

`app/router.js`
```js
Router.map(function() {
  this.route('foo', function() {
    this.route('bar', function() {
      this.route('slow-model');
    });
  });
});
```

Ember по очереди будет искать в иерархии шаблон `routeName-loading` или `loading`, начиная с `foo.bar.slow-model-loading`:

1. `foo.bar.slow-model-loading`
2. `foo.bar.loading` или `foo.bar-loading`
3. `foo.loading` или `foo-loading`
4. `loading` или `application-loading`
  
Важно отметить, что для самого `slow-model` Ember не будет искать шаблон `slow-model.loading`, но для остальной части иерархии допустим любой синтаксис. Это может быть полезно для создания индивидуального загрузочного экрана для крайнего маршрута вроде `slow-model`.

### Событие `loading`

Если вы возвращаете обещание из hooks `beforeModel`/`model`/`afterModel`, и оно не разрешается сразу, на этом маршруте будет запущено событие [`loading`](http://emberjs.com/api/classes/Ember.Route.html#event_error).

`app/routes/foo-slow-model.js`
```js
export default Ember.Route.extend({
  model() {
    return this.store.findAll('slowModel');
  },
  actions: {
    loading(transition, originRoute) {
      let controller = this.controllerFor('foo');
      controller.set('currentlyLoading', true);
    }
  }
});
```

Если обработчик `loading` не определен на конкретном маршруте, событие продолжит распространяться выше родительского маршрута перехода, предоставляя маршруту `application` возможность обработать его.

При использовании обработчика `loading` мы можем применять обещание перехода, чтобы знать, когда загрузка события завершится:

`app/routes/foo-slow-model.js`
```js
export default Ember.Route.extend({
  ...
  actions: {
    loading(transition, originRoute) {
      let controller = this.controllerFor('foo');
      controller.set('currentlyLoading', true);
      transition.promise.finally(function() {
          controller.set('currentlyLoading', false);
      });
    }
  }
});
```

## Подсостояния `error`

В случае возникновения ошибок во время перехода Ember предоставляет подход, который аналогичен подсостояниям `loading`.

Как и исходные обработчики события `loading`, обработчики `error` будут искать для входа подходящее подсостояние ошибки, если она будет обнаружена.

`app/router.js`
```js
Router.map(function() {
  this.route('articles', function() {
    this.route('overview');
  });
});
```

Так же как и с подсостоянием `loading`, при выдаче ошибки или обещания с ошибкой, которые вернулись из hook `model` маршрута `articles.overview` (или `beforeModel`/`afterModel`), Ember будет искать шаблон ошибки или маршрут в следующем порядке:

1. `articles.overview-error`
2. `articles.error` или `articles-error`
3. `error` или `application-error`

Если что-то из вышеперечисленного найдено, роутер сразу перейдет в это подсостояние (без обновления URL). «Причина» ошибки (то есть вызванное исключение или значение обещания с ошибкой) будет передана этому состоянию ошибки в качестве его `model`.

Если соответствующих подсостояний ошибки не обнаружено, в журнале будет записано сообщение об ошибке.

### Событие `error`
 
Если hook `model` маршрута `articles.overview` возвращает обещание, которое завершено с ошибкой (например, сервер вернул ошибку, что пользователь не авторизовался, и т. д.), сработает событие [`error`](http://emberjs.com/api/classes/Ember.Route.html#event_error), которое распространится дальше. Это событие `error` можно обработать и использовать, чтобы отобразить сообщение об ошибке, перенаправить на страницу авторизации и т. д.

`app/routes/articles-overview.js`
```js
export default Ember.Route.extend({
  model(params) {
    return this.store.findAll('problematicModel');
  },
  actions: {
    error(error, transition) {
      if (error) {
        return this.transitionTo('errorPage');
      }
    }
  }
});
```

По аналогии с событием `loading` вы можете управлять событием `error` на уровне приложения, чтобы не писать один и тот же код для нескольких маршрутов.