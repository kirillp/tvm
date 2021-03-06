---
title: TeaVM &mdash; инструмент для создания веб-фронтэнда на Java, Kotlin и Scala
---

Довольно давно я опубликовал на Хабре [статью](https://habrahabr.ru/post/240999/),
где рассказал про [свой проект](http://teavm.org/).
С тех пор много всего произошло с TeaVM,
в том числе одна важная вещь, про которую речь пойдёт ниже и ради которой я решил снова написать на Хабр.
Но для начала я кратко напомню, про что мой проект.

Итак, TeaVM &mdash; это компилятор байт-кода Java в JavaScript.
Идея создания TeaVM пришла мне, пока я работал full-stack Java разработчиком и использовал для написания фронтэнда GWT.
В те времена (а это где-то лет 5 назад) не были широко распространены инструменты вроде node.js, webpack, babel, TypeScript;
Angular был в первой версии, а альтернатив вроде React и vue.js не было вообще.
Тогда ещё на полном серьёзе люди тестировали сайты в IE7 (а некоторые, кому не повезло с заказчиками, даже IE6).
В целом, экосистема JavaScript была гораздо менее зрелой, чем сейчас, и без боли писать на JavaScript было нельзя.

GWT мне нравился тем, что на фоне всего этого он казался адекватным решением, хотя и не лишённым своих недостатков.
Основные проблемы были таковы:

* Скорость пересборки оставляла желать лучшего.
  GWT ну очень медленный.
  Запустили компиляцию и пошли пить чай с печеньками.
* GWT на вход получает исходники на Java поэтому,
  во-первых, медленно работает  (парсить и резолвить Java-код &mdash; это задача непростая),
  во-вторых, не поддерживает ничего, кроме Java,
  в-третьих, он сильно отстаёт даже в поддержке новых версий самой Java.
* В GWT просто ужасный фреймворк для создания UI.
  С одной стороны, он пытается абстрагироваться от DOM, с другой стороны, не всегда идёт в этом до конца,
  абстракции протекают, а поправить это прямым вмешательством в DOM непросто, т.к. приходится
  прорываться через толстые слои абстракции.
  Кроме того, пока весь цивилизованный мир шёл по пути декларативного описания разметки
  (WPF в .NET, JavaFX, Flex, упомянутые уже современные фреймворки для JavaScript),
  GWT по старинке заставлял собирать UI из виджетов. 

В целом, мне не казалось, что сама по себе идея написания веб-приложений на Java &mdash; плоха.
Все недостатки GWT были от того, что Google, как мне кажется, просто не вложили достаточных ресурсов в его развитие.
Я предпочитал мириться с недостатками GWT, лишь бы не переходить на JavaScript.

И тогда подумал &mdash; а почему бы не использовать в качестве входного материала байт-код Java?
Байт-код сохраняет много информации об исходной программе,
настолько много, что декомпиляторы умудряются потом почти точь-в-точь восстановить исходники.
При этом javaс делает всю самую сложную работу по генерации байт-кода,
поэтому на генерацию JavaScript должно потратиться совсем немного времени.
Плюсом получаем поддержку других языков для JVM и почти бесплатную поддержку новых версий Java
(байт-код гораздо консервативнее, чем язык Java).


## Что интересного произошло с проектом

Итак, я уже говорил, что с проектом произошло много всего интересного, 
вот самые важные моменты, о которых хотелось бы рассказать:

* Появилась поддержка тредов (threads).
  В JavaScript их нет совсем, WebWorkers &mdash; это не про то, поскольку, в отличие от тредов,
  они не имеют разделяемого состояния, т.е. нельзя создать объект в одном воркере и передать его другому,
  не скопировав.
  Вместо этого TeaVM умеет преобразовывать код методов так, что их выполнение можно прервать в некоторых
  заданных точках (примерно аналогично тому, как это делает babel, когда видит await).
  Это позволило сделать некоторый вариант кооперативной многозадчности, когда один тред, добравшись
  до какой-нибудь долгой IO-операции (ну или просто какого-нибудь `Thread.sleep()`),
  приостанавливается и позволяет выполниться другому треду.
* Появилась поддержка WebAssembly, пока экспериментальная.
  Поддерживается произвольный байт-код, но не поддерживается часть методов JDK, которые сильно
  завязаны на JVM (это прежде всего различный reflection).
  Так же совсем отсутствует интероп с чем-либо; чтобы вызвать JavaScript, надо очень долго поплясать с бубном.
  Вообще, ради поддержки WebAssembly мне пришлось написать свой GC и stack unwinding,
  настолько он сейчас низкоуровневый.
* Появился свой фреймворк для написания веб-фротнэнда на TeaVM, и это та самая новость,
  из-за которой я написал данную статью.
  Впрочем, подробности &mdash; ниже.


## Веб-фреймворк

Веб-фреймворк для TeaVM называется Flavour и он целиком написан на Java.
Недавно я опубликовал первую версию (за номером 0.1.0) на Maven Central.

Идеологически он напоминает современные Angular 2 и Vue.js, но построен целиком на идиомах, близких Java-разработчику.
Например, все компоненты Flavour представлены обычными Java-классами, размеченными аннотациями,
нет никаких отдельных props и state, или каких-то специальных объектов, которые должны инкапсулировать
изменяющиеся свойства.
Язык HTML-шаблонов полностью статически типизирован, любое обращение к свойству объекта или вызов
обработчика события проверяются во время компиляции и, например, опечатки в названии свойств
просто не дадут скомпилировать проект.

Для коммуникации с сервером Flavour предлагает использовать интерфейсы, размеченные аннотациями JAX-RS,
а данные передаются с помощью DTO, которые в свою очередь размечаются аннотациями Jackson.
Это должно быть удобно разработчикам на Java, которые уже знают и, может быть, используют эти
API в своих проектах.

Созданием Flavour я занялся, так как оказалось, что просто с TeaVM невозможно сделать что-то интересное.
Это же голый компилятор, который позволяет сгенерировать JavaScript, имеющий дело с не менее голым DOM.
Как на таком можно написать по-настоящему крутое приложение?

Возникает закономерный вопрос: а зачем создавать фреймворк, если существуют имеющиеся: React, Angular, Vue.js?
Можно же просто воспользоваться JavaScript интеропом и ничего не изобретать.
Конечно же, я думал об этом.
Но нет, всё оказывается намного хуже, чем кажется на первый взгляд.
Эти фреймворки построены вокруг идиом динамически типизированного JavaScript,
а в нём и объект с нужным набором свойств можно создать из воздуха, и понадеяться, что у "класса" объекта
есть магический метод с нужным названием.
Вообще, в мире JavaScript, создатели фреймворков не привыкли думать про типизацию.
Преодолеть это можно написанием всяческих адаптеров, обёрток, препроцессоров, генераторов.
Но в итоге получится система посложнее исходных фреймворков, поэтому было решено написать свой.


## Немного примеров

### Создание проекта

Конечно, все интересующиеся могут почитать документацию на [сайте](http://teavm.org/).
Но чтобы было проще и быстрее ощутить вкус Flavour, покажу небольшой пример здесь.

Итак, создать проект можно с помощью Maven:

      mvn archetype:generate \
        -DarchetypeGroupId=org.teavm.flavour \
        -DarchetypeArtifactId=teavm-flavour-application \
        -DarchetypeVersion=0.1.0

Собирается сгенерированный проект, как и ожидается, командой `mvn package`.


### Шаблонизатор

Страничка описывается двумя файлами &mdash; кодом класса, который описывает её поведение
(предоставляет данные для отображения, содержит обработчики событий) и HTML-шаблоном.
В созданном проекте уже имеется пример странички, но я всё-таки приведу ещё один пример.
Вы можете заменить сгенерированный из архетипа пример или просто добавить ещё два файла к имеющимся.
Вот так выглядит код класса странички:

    @BindTemplate("templates/fibonacci.html")
    public class Fibonacci {
        private List<Integer> values = new ArrayList<>();

        public Fibonacci() {
            values.add(0);
            values.add(1);
        }
 
        public List<Integer> getValues() {
            return values;
        }

        public void next() {
            values.add(values.get(values.size() - 2) + values.get(values.size() - 1));
        }
    }

А вот так &mdash; её шаблон:

    <ul>
      <!-- values - это сокращение для this.getValues() -->
      <std:foreach var="fib" in="values">
        <li>
          <html:text value="fib"/>
        </li>
      </std:foreach>
      <li>
        <!-- на самом деле, event:click может просто выполнить какой-то кусочек кода,
             но для удобства рекомендуется обработчик события умещать в метод,
             а из шаблона просто вызывать его -->
        <button type="button" event:click="next()">Show next</button>
      </li>
    </ul>

`std:foreach`, `html:text` и `event:click` &mdash; это компоненты Flavour.
Пользователь может описывать свои компоненты
(интересующиеся, как именно, могут почитать об этом в [документации](docs/flavour/custom-components.html)), 
при этом они могут либо вручную отрисовывать свой DOM, либо делать это через шаблон.
К слову, в этих компонентах нет ничего особенного, они не реализуются компиляторной магией.
При желании вы можете написать свои аналоги.
Для иллюстрации можете ознакомиться с кодом [html:text](https://github.com/konsoletyper/teavm-flavour/blob/master/templates/src/main/java/org/teavm/flavour/components/html/TextComponent.java).

Наконец, вот как должен выглядеть код метода `main`:

    public static void main(String[] args) {
        Templates.bind(new Fibonacci(), "application-content");
    }

Вся основная магия стартует именно здесь.
Фреймворк не создаёт экземпляр класса страницы, и никак не управляет им.
Вместо этого вы сами создаёте объект и сами управляете им, а Flavour просто генерирует DOM,
вставляет его в нужное место,
отслеживает изменения данных и перерисовывает DOM согласно этим изменениям.
Кстати, Flavour не перерисовывает весь DOM, а меняет только необходимую его часть.

Хочу ещё раз отметить, что шаблолны статически типизированы.
Если ошибиться и написать `event:click="nxt()"`, то компилятор напишет сообщение об ошибке.
Кстати, такой подход позволяет ещё и генерировать более быстрый код &mdash;
Flavour не тратит время после загрузки страницы, чтобы распарсить директивы и проинициализировать байндинги;
он всё это делает во время компиляции.


### REST-клиент

Теперь мне хотелось бы показать, как Flavour может быть полезен fullstack-разработчику.
Допустим, вы используете какой-нибудь CXF в связке с JAX-RS.
Вы написали примерно такой интерфейс:

    @Path("math")
    public interface MathService {
        @GET
        @Path("integers/sum")
        int sum(@QueryParam("a") int a, @QueryParam("b") int b);
    }

и реализовали его (например, в классе `MathServiceImpl`), зарегистрировали реализацию в CXF.
У вас готов небольшой REST-сервис.
Теперь, чтобы сделать на него запрос, со стороны клиента можно написать такой код:

    MathService math = RESTClient.factory(MathService.class).createResource("api");
    System.out.println(math.sum(2, 3));

(можно увидеть в devtools, что этот код пошлёт GET-запрос на адрес `/api/math/integers/sum?a=2&b=3`.

В общем, не надо каким-то образом объяснять веб-клиенту, как правильно делать REST-запрос на нужный endpoint.
Фактически, вы уже это сделали для сервера.
Можно дальше наращивать и рефакторить REST-сервис, при этом не надо синхронизировать это со стороны сервера
и со стороны клиента &mdash; есть точка синхронизации в виде интерфейса `MathService`.

В GWT есть аналогичный механизм, GWT-RPC, но он заставляет генерировать дополнительный async-интерфейс
и использовать колбэки, тогда как TeaVM умеет преобразовывать синхронный код в асинхронный.
И GWT-RPC использует свой, ни с чем не совместимый протокол, так что создав endpoint для GWT-RPC,
вы не сможете его переиспользовать, например, для iOS-клиента.


## Что ещё интересного есть?

Разумеется, в небольшой обзорной статье я не могу рассказать обо всём вообще.
Поэтому просто упомяну, что такого интересного есть в TeaVM и Flavour,
что делает их вполне пригодными для создания качественных веб-приложений.

* Интеграция с IntelliJ IDEA.
  Запускать каждый раз Maven &mdash; это несерьёзно,
  всё-таки Maven используют для сборки на production, а не во время разработки.
  TeaVM умеет запускаться прямо из IDEA, достаточно просто нажать кнопку Build.
* Отладчик.
  Конечно, без него невозможно вообще говорить о чём-то, как о серьёзном инструменте.
  TeaVM умеет генерить стандартные source maps,
  а так же есть свой формат отладочной информации, который используется в IDEA-плагине.
* Очень быстрый компилятор.
  Я потратил усилия на оптимизацию, да и байт-код &mdash; это настолько примитивная вещь,
  что там просто нечему тормозить.
  Это не Java, где надо, например, типы выводить, с generics, вариантностью и captured-типами.
  Кроме того, запуская компилятор из IDEA, я применяю ряд техник для уменьшения времени компиляции,
  таких как запуск компилятора из демона и кэширование информации для последующих сборок.
  Всё это позволяет добиться вполне комфортной скорости пересборки JavaScript.
* Хороший оптимизатор, способный отрезать от внушительного по размеру JDK
  очень небольшой кусочек, необходимый для нормальной работы приложения,
  да ещё и сделать его быстрым.
* Роутинг.
  Flavour умеет парсить и генерировать ссылки,
  которые с точки зрения API &mdash; это вполне себе статически типизированные интерфейсы.
* Валидация форм.
* Модальные окошки с блокирующим API, как в Swing. 


## Зачем Java в вебе?

Действительно, как я уже писал, экосистема разработки JavaScript стала в наше время вполне зрелой.
Разработчику проще не использовать всякие тяжеловесные инструменты вроде GWT,
а научиться настраивать инструменты, ставшие стандартами де-факто и писать на современном языке с большим количеством
продвинутых фич, или даже на современном статически типизированном языке (TypeScript).

Проблема в том, что всё это надо изучать.
Изучать не просто синтаксис языка (на это опытный разработчик потратит несколько дней),
изучать много всего &mdash; библиотеки, идиомы, инструменты разработчика.
Уверен, что вчерашний опытный Java-разработчик возьмёт и разберётся со всем этим за пару недель,
но вопрос в том, насколько хорошо он успеет разобраться?
Сможет ли он писать действительно хороший код?
Но даже если разберётся, и сможет, есть ещё проблемы.
Во-первых, разработчику придётся переключать контекст между Java и JavaScript.
Во-вторых, так разработчику придётся потратить больше времени на настройку инструментов.

Ну и у меня есть вопрос к Java-коммьюнити.
Почему-то же есть со стороны JavaScript такое настойчивое движение в сторону бэкэнда?
Подозреваю, что ровно из тех соображений, которые я привёл выше.
Почему бы не быть такому же движению в обратном направлении?
Вот сообщество JavaScript frontend разработчиков скооперировалось и породило backend экосистему вокруг node.js.
Чем мы, сообещество разработчиков на Java, хуже?
Есть такой миф, что якобы JavaScript шустрый и легковесный, 
а Java &mdash; большая и тяжеловесная, и не подходит для создания фронтэнда.
На самом деле, своим проектом я пытаюсь доказать, что это именно что миф, и что правильно приготовленная
Java тоже может быть маленькой и шустрой.
Тому пример &mdash; реализация TodoMVC, которая занимает 125kb
(попробуйте написать TodoMVC на React или Angular 2 и посмотрите, какой здоровенный у вас получится бандл).


## В заключение

Если вы заинтересовались Flavour, вот вам ещё немного материала для изучения:

* [Документация](http://teavm.org/docs/intro/overview.html) на английском;
* [TodoMVC](https://github.com/konsoletyper/teavm-flavour-examples-todomvc) на Kotlin;
* [Сапёр](https://github.com/konsoletyper/teavm-flavour-minesweeper) на Java;
* [Пример CRUD, близкий к реальному](https://github.com/konsoletyper/teavm-flavour/tree/master/example),
  с REST, валидацией, выпадающими календариками, модальными окнами и многим другим;
* Конечно же, всегда можно спрашивать меня, если что-то показалось непонятным.

Мне очень хотелось бы получить обратную связь.
Интересен ли вам мой проект, хотели бы вы его попробовать для написания небольшого приложения?
Нужно ли мне ещё публиковать статьи, и если да, о чём именно вы хотели бы в нём прочитать?
Я могу публиковать туториалы по Flavour, могу рассказывать, как он устроен внутри,
как работает TeaVM.
Что из этого вам интереснее?
