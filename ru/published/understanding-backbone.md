От jQuery к Backbone по шагам
====================================

Я бы свидетелем многих трудностей перехода на
[Backbone.js](http://backbonejs.org/). В этом посте буду постепенно
рефакторить кусочек кода, перенося его с моего старого стиля написания
JavaScript'а, на более правильный Backbone.js, использующий модели,
коллекции, виды и события. Надеюсь, что этот процесс даст вам крепкое
понятие основных абстракций Backbone.js.

Давайте начнем с кода, работать с которым мы будем в течении всего поста:

```javascript
$(document).ready(function() {
    $('#new-status form').submit(function(e) {
        e.preventDefault();

        $.ajax({
            url: '/status',
            type: 'POST',
            dataType: 'json',
            data: { text: $('#new-status').find('textarea').val() },
            success: function(data) {
                $('#statuses').append('<li>' + data.text + '</li>');
                $('#new-status').find('textarea').val('');
            }
        });
    });
});
```

А вот и готовое приложение:
[Monologue](http://monologue-js.herokuapp.com/). Оно основанно на
[крутой JavaScript презентации](http://opensoul.org/blog/archives/2012/05/16/the-plight-of-pinocchio/)
[Брэндона Киперса (Brandon Keepers)](http://opensoul.org). Однако, в то
время как его основное внимание уделено тестированию и шаблонам
проектирования — посредством которых он делает офигенные штуки — мое же
уделено изучению основных абстракций Backbone.js, по одному на шаг.

В принципе все что делает приложение, это дает вам написать какой-нибудь
текст, затем отправить, и отображает его в списке, пока текстовое
поле не будет готово для нового ввода. Переводя все это в код, мы
начинаем с ожидания загрузки DOM, устанавливаем слушатель отправки, и
при отправке формы передаем текст на сервер, добавляем ответ в список
статусов и очищаем поле ввода.

Но в чем проблема-то? Этот код делает кучу вещей одновременно. Слушает
события страницы, пользователя, сети, производит ввод-вывод,
обрабатывает пользовательский ввод, парсит ответ, и проводит
HTML-шаблонизацию. Все это в 16 строчках. Похоже, что
я написал этот JavaScript год назад. Походу поста я буду постепенно
приводить код к моему стилю, следующему принципу единой ответственности
(single responsibility principle) и становящимуся проще для тестирования,
сопровождения и расширения.

Вот три вещи, которых мы хотим достигнуть:

* Мы хотим по возможности вынести весь код из `$(document).ready`, таким
  образом можно вызывать код самим — нам только нужно загружать
  приложение, когда подготовится DOM. В текущем состоянии код почти
  невозможно тестировать.
* Мы хотим придерживаться принципа единой ответственности и сделать так,
  чтобы код можно было использовать повторно и легко тестировать.
* Мы хотим разорвать связь между DOM и Ajax.

Разделение DOM и Ajax
-----------------------

Мы начнем с отделения Ajax и DOM друг от друга, первым шагом которого
будет создание функции `addStatus`:

```diff
+function addStatus(options) {
+    $.ajax({
+        url: '/status',
+        type: 'POST',
+        dataType: 'json',
+        data: { text: $('#new-status textarea').val() },
+        success: function(data) {
+            $('#statuses ul').append('<li>' + data.text + '</li>');
+            $('#new-status textarea').val('');
+        }
+    });
+}
+
 $(document).ready(function() {
     $('#new-status form').submit(function(e) {
         e.preventDefault();

-        $.ajax({
-            url: '/status',
-            type: 'POST',
-            dataType: 'json',
-            data: { text: $('#new-status textarea').val() },
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
+        addStatus();
     });
 });
```

Тем не менее, `data` и `success` работают с DOM. Мы можем разорвать эту
связь, посылкой их в качестве аргументов функции `addStatus`:

```diff
 function addStatus(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
-        data: { text: $('#new-status textarea').val() },
-        success: function(data) {
-            $('#statuses ul').append('<li>' + data.text + '</li>');
-            $('#new-status textarea').val('');
-        }
+        data: { text: options.text },
+        success: options.success
     });
 }

 $(document).ready(function() {
     $('#new-status form').submit(function(e) {
         e.preventDefault();

-        addStatus();
+        addStatus({
+            text: $('#new-status textarea').val(),
+            success: function(data) {
+                $('#statuses ul').append('<li>' + data.text + '</li>');
+                $('#new-status textarea').val('');
+            }
+        });
     });
 });
```

Однако, я хочу обернуть эти статусы в объект и писать `statuses.add`
вместо `addStatus`. Для этого можно использовать
[constructor pattern with prototypes](http://addyosmani.com/resources/essentialjsdesignpatterns/book/#constructorpatternjavascript)
для представления "класса" Statuses:

```diff
-function addStatus(options) {
+var Statuses = function() {
+};
+Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
-}
+};

 $(document).ready(function() {
+    var statuses = new Statuses();
+
     $('#new-status form').submit(function(e) {
         e.preventDefault();

-        addStatus({
+        statuses.add({
             text: $('#new-status textarea').val(),
             success: function(data) {
                 $('#statuses ul').append('<li>' + data.text + '</li>');
                 $('#new-status textarea').val('');
             }
         });
     });
 });
```

Создание вида
---------------

Наш обработчик ввода имеет одну зависимость — переменную `statuses` и
все остальное сосредоточенное в нем на DOM. Давайте переместим
обработчик ввода и все остальное внутри него в его собственный класс
`NewStatusView`:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };

+var NewStatusView = function(options) {
+    var statuses = options.statuses;
+
+    $('#new-status form').submit(function(e) {
+        e.preventDefault();
+
+        statuses.add({
+            text: $('#new-status textarea').val(),
+            success: function(data) {
+                $('#statuses ul').append('<li>' + data.text + '</li>');
+                $('#new-status textarea').val('');
+            }
+        });
+    });
+};
+
 $(document).ready(function() {
     var statuses = new Statuses();

-    $('#new-status form').submit(function(e) {
-        e.preventDefault();
-
-        statuses.add({
-            text: $('#new-status textarea').val(),
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
-    });
+    new NewStatusView({ statuses: statuses });
 });
```

Теперь мы загружаем наше приложение только тогда, когда DOM готов, а все
остальное вынесено за пределы `$(document).ready`. Предпринятые шаги
дали нам два компонента, которые легче тестировать и которые имеют более
четко определенные обязанности. Тем не менее, еще многое можно улучшить.
Давайте начнем с разделения обработчика ввода из `NewStatusView` в его
собственный метод `addStatus`:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };

 var NewStatusView = function(options) {
-    var statuses = options.statuses;
+    this.statuses = options.statuses;

-    $('#new-status form').submit(function(e) {
-        e.preventDefault();
-        statuses.add({
-            text: $('#new-status textarea').val(),
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
-    });
+    $('#new-status form').submit(this.addStatus);
 };
+NewStatusView.prototype.addStatus = function(e) {
+    e.preventDefault();
+
+    this.statuses.add({
+        text: $('#new-status textarea').val(),
+        success: function(data) {
+            $('#statuses ul').append('<li>' + data.text + '</li>');
+            $('#new-status textarea').val('');
+        }
+    });
+};

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Запустив это в Chrome, получаем следующую ошибку:

    Uncaught TypeError: Cannot call method 'add' of undefined

Мы получили эту ошибку, потому что `this` в конструкторе и методе
`addStatus` находится в разных контекстах, т.к. jQuery на самом деле
вызывает его позже, когда пользователь отправляет форму. (Если вы не до
конца понимаете как работает `this`, рекомендую к прочтению
[Understanding JavaScript Function Invocation and “this”](http://yehudakatz.com/2011/08/11/understanding-javascript-function-invocation-and-this/).)
Для решения этой проблемы можно использовать
[`$.proxy`](http://api.jquery.com/jQuery.proxy/), создающий функцию, где
`this` всегда тоже, что и контекст, который указывается вторым
аргументом.

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;

-    $('#new-status form').submit(this.addStatus);
+    var add = $.proxy(this.addStatus, this);
+    $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
             $('#statuses ul').append('<li>' + data.text + '</li>');
             $('#new-status textarea').val('');
         }
     });
 };

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Давайте вместо работы непосредственно с DOM, создадим callback методы
`success`, что сделает callback легче для чтения и работы:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;

     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

+    var that = this;
+
     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
-            $('#statuses ul').append('<li>' + data.text + '</li>');
-            $('#new-status textarea').val('');
+            that.appendStatus(data.text);
+            that.clearInput();
         }
     });
 };
+NewStatusView.prototype.appendStatus = function(text) {
+    $('#statuses ul').append('<li>' + text + '</li>');
+};
+NewStatusView.prototype.clearInput = function() {
+    $('#new-status textarea').val('');
+};

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Так будет намного легче тестировать и работать, когда наш код будет
расти. Мы все еще далеки от пути понимания Backbone.js.

Добавление событий
-------------

Для следующего шага нам необходимо познакомиться с нашей первой
частичкой Backbone: событиями. События это просто способ сказать:
"Привет, я хочу быть в курсе того или иного действия" и "Привет, знаешь
что? Действие которое ты ждал, произошло!" Мы привыкли к этой идее из DOM
-событий jQuery, таких как, например слушание `click` и `submit`.

```javascript
$('form').bind('submit', function() {
  alert('User submitted form');
});
```

Документация Backbone.js описывает
[`Backbone.Events`](http://backbonejs.org/#Events) следующее: *"События
это модули, которые могут быть подмешаны в любой объект, дающий ему
право биндить и триггерить кастомно именнованные события."* Документация
также показывает как использовать
[Underscore.js](http://underscorejs.org/), чтобы создать диспетчер
событий (event dispatcher), т.е. компонент на который мы можем биндить и
треггерить события:

```javascript
var events = _.clone(Backbone.Events);
```

С этим маленьким кусочком функциональности, мы можем иннициировать
событе на callback-функции `success`, вместо вызова метода. Мы также
можем указать в конструкторе, какой метод мы хотим вызвать, когда
событие проинициализируется:

```diff
+var events = _.clone(Backbone.Events);
+
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;

+    events.on('status:add', this.appendStatus, this);
+    events.on('status:add', this.clearInput, this);
+
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

-    var that = this;
-
     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
-            that.appendStatus(data.text);
-            that.clearInput();
+            events.trigger('status:add', data.text);
         }
     });
 };
 NewStatusView.prototype.appendStatus = function(text) {
     $('#statuses ul').append('<li>' + text + '</li>');
 };
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Теперь вместо функции `addStatus`, отвечающей за обработку успешного
исхода, мы можем указать в конструкторе, что мы хотим произвести, в тот
момент, когда добавится статус. `addStatus` должен нести ответственнгсть
только за коммунникацию с бекендом, а не обновлять DOM.

Так как мы больше не работаем с видом в callback-функции`success`, можно
переместить инициализацию события в метод `add` класса `Statuses`:

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
-Statuses.prototype.add = function(options) {
+Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
-        data: { text: options.text },
-        success: options.success
+        data: { text: text },
+        success: function(data) {
+            events.trigger('status:add', data.text);
+        }
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;

     events.on('status:add', this.appendStatus, this);
     events.on('status:add', this.clearInput, this);

     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

-    this.statuses.add({
-        text: $('#new-status textarea').val(),
-        success: function(data) {
-            events.trigger('status:add', data.text);
-        }
-    });
+    this.statuses.add($('#new-status textarea').val());
 };
 NewStatusView.prototype.appendStatus = function(text) {
     $('#statuses ul').append('<li>' + text + '</li>');
 };
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Ответственность представлений
-----------------------------

Глядя на `appendStatus` и `clearInput` в `NewStatusView`, мы видим, что
эти методы сфокусированы на двух различных DOM элементах, `#statuses` и
`#new-status`, соответственно. В
[приложении](http://monologue-js.herokuapp.com/) я покрасил их в разные
цвета, так что вы можете видеть различия. Работа с двумя элементами в
представлении не соотвествует принципам, которые я описал в этом посте
[ответственность представлений](https://open.bekk.no/a-views-responsibility/).
Давайте вынесим `StatusesView` из `NewStatusView` и пусть он будет
отвечать за `#statuses`. Разделение этих ответственностей теперь особенно
просто — с функцией обратного вызова `success` это было бы гораздо
сложнее.

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;

-    events.on('status:add', this.appendStatus, this);
     events.on('status:add', this.clearInput, this);

     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

     this.statuses.add($('#new-status textarea').val());
 };
-NewStatusView.prototype.appendStatus = function(text) {
-    $('#statuses ul').append('<li>' + text + '</li>');
-};
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };

+var StatusesView = function() {
+    events.on('status:add', this.appendStatus, this);
+};
+StatusesView.prototype.appendStatus = function(text) {
+    $('#statuses ul').append('<li>' + text + '</li>');
+};
+
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
+    new StatusesView();
 });
```

Т.к. теперь каждое представление ответственно только за один HTML
элемент, мы можем указать их при инстанциировании представлений:

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;
+    this.el = $('#new-status');

     events.on('status:add', this.clearInput, this);

     var add = $.proxy(this.addStatus, this);
-    $('#new-status form').submit(add);
+    this.el.find('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

-    this.statuses.add($('#new-status textarea').val());
+    this.statuses.add(this.el.find('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
-    $('#new-status textarea').val('');
+    this.el.find('textarea').val('');
 };

 var StatusesView = function() {
+    this.el = $('#statuses');
+
     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
-    $('#statuses ul').append('<li>' + text + '</li>');
+    this.el.find('ul').append('<li>' + text + '</li>');
 };

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
     new StatusesView();
 });
```

Наши представления `NewStatusView` и `StatusesView` все еще сложно
тестировать, потому что они зависят от HTML, например, чтобы найти
`$('#statuses')`. Для устранения этого мы можем в тот момент, когда
создаем экземпляр представления, передать их DOM зависимости.

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;
-    this.el = $('#new-status');
+    this.el = options.el;

     events.on('status:add', this.clearInput, this);

     var add = $.proxy(this.addStatus, this);
     this.el.find('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

     this.statuses.add(this.el.find('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
     this.el.find('textarea').val('');
 };

-var StatusesView = function() {
-    this.el = $('#statuses');
+var StatusesView = function(options) {
+    this.el = options.el;

     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
     this.el.find('ul').append('<li>' + text + '</li>');
 };

 $(document).ready(function() {
     var statuses = new Statuses();
-    new NewStatusView({ statuses: statuses });
-    new StatusesView();
+    new NewStatusView({ el: $('#new-status'), statuses: statuses });
+    new StatusesView({ el: $('#statuses') });
 });
```

Теперь это легко тестировать! С этим изменением мы можем использовать
[jQuery trick](http://api.jquery.com/jQuery/#jQuery2) для тестирования
представлений. Вместо того, чтобы инициализировать наши представления
передачей например в `$('#new-status')`, мы можем перейти в нужный HTML,
обернутый в jQuery, например `$('<div><form>…</form></div>')`. Затем
jQuery создаст на лету необходимые DOM элементы. Это обеспечит
невероятно быстрые тесты — в моем текущем проекте наши около 200 тестов
исполнялись менее одной секунды.

Нашим следующим шагом будет введение хелпера, чтобы немного причесать
наши представления. Вместо того, чтобы писать `this.el.find`, можно
создать простой хелпер, позволяющий писать `this.$`. С этим маленьким
изменением, мы как-будто будем говорить, "Я хочу использовать jQuery
для локального поиска чего-нибудь в этом преставлении, вместо
глобального во всем HTML. Это очень просто добавить:

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = function(options) {
     this.statuses = options.statuses;
     this.el = options.el;

     events.on('status:add', this.clearInput, this);

     var add = $.proxy(this.addStatus, this);
-    this.el.find('form').submit(add);
+    this.$('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

-    this.statuses.add(this.el.find('textarea').val());
+    this.statuses.add(this.$('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
-    this.el.find('textarea').val('');
+    this.$('textarea').val('');
 };
+NewStatusView.prototype.$ = function(selector) {
+    return this.el.find(selector);
+};

 var StatusesView = function(options) {
     this.el = options.el;

     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
-    this.el.find('ul').append('<li>' + text + '</li>');
+    this.$('ul').append('<li>' + text + '</li>');
 };
+StatusesView.prototype.$ = function(selector) {
+    return this.el.find(selector);
+};

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Тем не менее, добавление этой функциональности для каждого представления
мучительно. Это одна из причин использовать повторную функциональность
[представлений Backbone.js](http://backbonejs.org/#View).

Getting started with views in Backbone
--------------------------------------

With the current state of our code, it's just a couple of lines of
change needed to add Backbone.js views:

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

-var NewStatusView = function(options) {
-    this.statuses = options.statuses;
-    this.el = options.el;
-
-    events.on('status:add', this.clearInput, this);
-
-    var add = $.proxy(this.addStatus, this);
-    this.$('form').submit(add);
-};
+var NewStatusView = Backbone.View.extend({
+    initialize: function(options) {
+        this.statuses = options.statuses;
+        this.el = options.el;
+
+        events.on('status:add', this.clearInput, this);
+
+        var add = $.proxy(this.addStatus, this);
+        this.$('form').submit(add);
+    }
+});
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();

     this.statuses.add(this.$('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
     this.$('textarea').val('');
 };
 NewStatusView.prototype.$ = function(selector) {
     return this.el.find(selector);
 };

 var StatusesView = function(options) {
     this.el = options.el;

     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
     this.$('ul').append('<li>' + text + '</li>');
 };
 StatusesView.prototype.$ = function(selector) {
     return this.el.find(selector);
 };

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

As you can see from the code, we use `Backbone.View.extend` to create a
new view class in Backbone. Within `extend` we can specify instance
methods such as `initialize`, which is the name of the constructor.

Now that we have started the move over to Backbone.js views, let's go on
and move both views fully over:

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
         this.el = options.el;

         events.on('status:add', this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
-    }
+    },
+
+    addStatus: function(e) {
+        e.preventDefault();
+
+        this.statuses.add(this.$('textarea').val());
+    },
+
+    clearInput: function() {
+        this.$('textarea').val('');
+    },
+
+    $: function(selector) {
+        return this.el.find(selector);
+    }
 });
-NewStatusView.prototype.addStatus = function(e) {
-    e.preventDefault();
-
-    this.statuses.add(this.$('textarea').val());
-};
-NewStatusView.prototype.clearInput = function() {
-    this.$('textarea').val('');
-};
-NewStatusView.prototype.$ = function(selector) {
-    return this.el.find(selector);
-};

-var StatusesView = function(options) {
-    this.el = options.el;
-
-    events.on('status:add', this.appendStatus, this);
-};
-StatusesView.prototype.appendStatus = function(text) {
-    this.$('ul').append('<li>' + text + '</li>');
-};
-StatusesView.prototype.$ = function(selector) {
-    return this.el.find(selector);
-};
+var StatusesView = Backbone.View.extend({
+    initialize: function(options) {
+        this.el = options.el;
+
+        events.on('status:add', this.appendStatus, this);
+    },
+
+    appendStatus: function(text) {
+        this.$('ul').append('<li>' + text + '</li>');
+    },
+
+    $: function(selector) {
+        return this.el.find(selector);
+    }
+});

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Now that we use Backbone.js views we can remove the `this.$` helper, as it
already exists in Backbone. We also no longer need to set `this.el`
ourselves, as Backbone.js does it automatically when a view is instantiated
with an HTML element.

```diff
 var events = _.clone(Backbone.Events);

 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };

 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
-        this.el = options.el;

         events.on('status:add', this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

         this.statuses.add(this.$('textarea').val());
     },

     clearInput: function() {
         this.$('textarea').val('');
     },
-
-    $: function(selector) {
-        return this.el.find(selector);
-    }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
-        this.el = options.el;
-
         events.on('status:add', this.appendStatus, this);
     },

     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     },
-
-    $: function(selector) {
-        return this.el.find(selector);
-    }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Let's use a model
-----------------

The next step is introducing models, which are responsible for the
network traffic, i.e. Ajax requests and responses. As Backbone nicely
abstracts Ajax, we don't need to specify `type`, `dataType` and `data`
anymore. Now we only need to specify the URL and call `save` on the
model. The `save` method accepts the data we want to save as the first
parameter, and options, such as the `success` callback, as the second
parameter.

```diff
 var events = _.clone(Backbone.Events);

+var Status = Backbone.Model.extend({
+    url: '/status'
+});
+
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
-    $.ajax({
-        url: '/status',
-        type: 'POST',
-        dataType: 'json',
-        data: { text: text },
-        success: function(data) {
-            events.trigger('status:add', data.text);
-        }
-    });
+    var status = new Status();
+    status.save({ text: text }, {
+        success: function(model, data) {
+            events.trigger('status:add', data.text);
+        }
+    });
 };

 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;

         events.on('status:add', this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

         this.statuses.add(this.$('textarea').val());
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         events.on('status:add', this.appendStatus, this);
     },

     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Handling several models
-----------------------

Now that we have introduced models, we need a concept for a list of
models, such as the list of statuses in our application. In Backbone.js
this concept is called a collection.

One really cool thing about collections is that they have scoped events.
Basically, this just means that we can bind and trigger events directly
on a collection instead of using our `events` variable — our events will
live on `statuses` instead of `events`. As we now start firing events
directly on statuses there's no need for "status" in the event name, so
we rename it from "status:add" to "add".

```diff
-var events = _.clone(Backbone.Events);
-
 var Status = Backbone.Model.extend({
     url: '/status'
 });

-var Statuses = function() {
-};
-Statuses.prototype.add = function(text) {
-    var status = new Status();
-    status.save({ text: text }, {
-        success: function(model, data) {
-            events.trigger("status:add", data.text);
-        }
-    });
-};
+var Statuses = Backbone.Collection.extend({
+    add: function(text) {
+        var that = this;
+        var status = new Status();
+        status.save({ text: text }, {
+            success: function(model, data) {
+                that.trigger("add", data.text);
+            }
+        });
+    }
+});

 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;

-        events.on("status:add", this.clearInput, this);
+        this.statuses.on("add", this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

         this.statuses.add(this.$('textarea').val());
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
+        this.statuses = options.statuses;
+
-        events.on("status:add", this.appendStatus, this);
+        this.statuses.on("add", this.appendStatus, this);
     },

     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
-    new StatusesView({ el: $('#statuses') });
+    new StatusesView({ el: $('#statuses'), statuses: statuses });
 });
```

We can simplify this even more by using Backbone's `create` method. It
creates a new model instance, adds it to the collection and saves it to
the server. Therefore we must specify what type of model the collection
should handle. There are two things we need to change to use Backbone
collections:

1. When creating the status we need to pass in an options hash with the
   attributes we want to save, instead of only passing the text.
2. The built in `create` also triggers an "add" event, but rather than
   passing only the text, as we have done so far, it passes the newly
   created model. We can get the text from the model by calling
   `model.get("text")`.

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });

 var Statuses = Backbone.Collection.extend({
-    add: function(text) {
-        var that = this;
-        var status = new Status();
-        status.save({ text: text }, {
-            success: function(model, data) {
-                that.trigger("add", data.text);
-            }
-        });
-    }
+    model: Status
 });

 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;

         this.statuses.on("add", this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

-        this.statuses.add(this.$('textarea').val());
+        this.statuses.create({ text: this.$('textarea').val() });
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;

         this.statuses.on("add", this.appendStatus, this);
     },

-    appendStatus: function(text) {
+    appendStatus: function(status) {
-        this.$('ul').append('<li>' + text + '</li>');
+        this.$('ul').append('<li>' + status.get("text") + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses'), statuses: statuses });
 });
```

As with `el` earlier, Backbone.js automatically sets `this.collection`
when `collection` is passed. Therefore we rename `statuses` to
`collection` in our view:

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });

 var Statuses = Backbone.Collection.extend({
     model: Status
 });

 var NewStatusView = Backbone.View.extend({
-    initialize: function(options) {
+    initialize: function() {
-        this.statuses = options.statuses;
-
-        this.statuses.on('add', this.clearInput, this);
+        this.collection.on('add', this.clearInput, this);

         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

-        this.statuses.add({ text: this.$('textarea').val() });
+        this.collection.create({ text: this.$('textarea').val() });
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
-    initialize: function(options) {
+    initialize: function() {
-        this.statuses = options.statuses;
-
-        this.statuses.on('add', this.appendStatus, this);
+        this.collection.on('add', this.appendStatus, this);
     },

     appendStatus: function(status) {
         this.$('ul').append('<li>' + status.get('text') + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
-    new NewStatusView({ el: $('#new-status'), statuses: statuses });
+    new NewStatusView({ el: $('#new-status'), collection: statuses });
-    new StatusesView({ el: $('#statuses'), statuses: statuses });
+    new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

Evented views
-------------

Now, let's get rid of that nasty `$.proxy` stuff. We can do this by
letting Backbone.js [delegate our events](http://backbonejs.org/#View-delegateEvents)
by specifying them in an `events` hash in the view. This hash is of the
format `{"event selector": "callback"}`:

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });

 var Statuses = Backbone.Collection.extend({
     model: Status
 });

 var NewStatusView = Backbone.View.extend({
+    events: {
+        'submit form': 'addStatus'
+    },
+
     initialize: function() {
         this.collection.on('add', this.clearInput, this);
-
-        var add = $.proxy(this.addStatus, this);
-        this.$('form').submit(add);
     },

     addStatus: function(e) {
         e.preventDefault();

         this.collection.create({ text: this.$('textarea').val() });
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function() {
         this.collection.on('add', this.appendStatus, this);
     },

     appendStatus: function(status) {
         this.$('ul').append('<li>' + status.get('text') + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), collection: statuses });
     new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

Escape it!
----------

As our last step we're going to prevent XSS-exploits. Instead of using
`model.get('text')` let's use the
[built-in escape handling](http://backbonejs.org/#Model-escape) and
write `model.escape('text')` instead. If you're using
[Handlebars](http://handlebarsjs.com/),
[Mustache](https://github.com/janl/mustache.js) or similar templating
engines, you might get this functionality out of the box.

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });

 var Statuses = Backbone.Collection.extend({
     model: Status
 });

 var NewStatusView = Backbone.View.extend({
     events: {
         "submit form": "addStatus"
     },

     initialize: function(options) {
         this.collection.on("add", this.clearInput, this);
     },

     addStatus: function(e) {
         e.preventDefault();

         this.collection.create({ text: this.$('textarea').val() });
     },

     clearInput: function() {
         this.$('textarea').val('');
     }
 });

 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         this.collection.on("add", this.appendStatus, this);
     },

     appendStatus: function(status) {
-        this.$('ul').append('<li>' + status.get("text") + '</li>');
+        this.$('ul').append('<li>' + status.escape("text") + '</li>');
     }
 });

 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), collection: statuses });
     new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

And we're done!
---------------

This is our final code:

```javascript
var Status = Backbone.Model.extend({
    url: '/status'
});

var Statuses = Backbone.Collection.extend({
    model: Status
});

var NewStatusView = Backbone.View.extend({
    events: {
        'submit form': 'addStatus'
    },

    initialize: function() {
        this.collection.on('add', this.clearInput, this);
    },

    addStatus: function(e) {
        e.preventDefault();

        this.collection.create({ text: this.$('textarea').val() });
    },

    clearInput: function() {
        this.$('textarea').val('');
    }
});

var StatusesView = Backbone.View.extend({
    initialize: function() {
        this.collection.on('add', this.appendStatus, this);
    },

    appendStatus: function(status) {
        this.$('ul').append('<li>' + status.escape('text') + '</li>');
    }
});

$(document).ready(function() {
    var statuses = new Statuses();
    new NewStatusView({ el: $('#new-status'), collection: statuses });
    new StatusesView({ el: $('#statuses'), collection: statuses });
});
```

And [here's](http://monologue-js.herokuapp.com/?step=22) the application
running with the refactored code. Yeah, it's still the exact same
application from a user's point of view.

However, the code has increased from 16 lines to more than 40, so why do
I think this is better? Because we are now working on a higher level of
abstraction. This code is more maintainable, easier to reuse and extend,
and easier to test. What I've seen is that Backbone.js helps improve the
structure of my JavaScript applications considerably, and in my
experience the end result is often less complex and has fewer lines
of code than my "regular JavaScript".

Want to learn more?
-------------------

By now you should understand far more of Backbone.js than when you did
an hour ago. There are some great resources for learning Backbone.js out
there, but there's also a whole lot of crap. Actually, the
[Backbone.js documentation](http://backbonejs.org/) is superb as soon as
you have a better understanding of how the framework works at a higher
level. Addy Osmani's
[Developing Backbone.js Applications](http://addyosmani.github.com/backbone-fundamentals/)
is another good source.

Reading the
[annotated source](http://documentcloud.github.com/backbone/docs/backbone.html)
of Backbone.js is also a great way to get a better understanding of how
it works. I found this to be an enlightening experience when I had used
Backbone.js for some weeks.

If you want to get started with Require.js in a Backbone app, you can
check out
[the follow-up](https://github.com/kjbekkelund/writings/blob/master/published/step-by-step-modules.md)
to this blog post.

And, lastly, if you want to get a better understanding of the creation
of a framework such as Backbone.js, I recommend the magnificent
[JavaScript Web Applications](http://www.amazon.com/JavaScript-Web-Applications-Alex-MacCaw/dp/144930351X)
by [Alex MacCaw](http://alexmaccaw.com/).
