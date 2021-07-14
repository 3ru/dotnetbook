# Время жизни сущностей

Один из вопросов, которые могут очень сильно влиять на производительность наших приложений -- это политика выбора типа сущности: класс или структура, и контроль за их временем жизни. Ведь архитекторы платформы не просто так выделили для нас эти два типа данных. Это деление в первую очередь обусловлено возможностями оптимизации приложений, архитектура которых учитывает особенности обоих групп типов.

[>]: Эти два слова: структура и класс, могу поспорить, многих заставляют вздрогнуть и мысленно спросить: "что, опять?..". Однако, прошу поверить мне на слово: за время чтения книги вы так же как и я сможете проникнуться их свойствами. 

Как будет сказано в главе [Ссылочные и значимые типы данных](../ReferenceTypesVsValueTypes.md), огромным преимуществом значимых типов данных является то, что их не надо аллоцировать. Т.е. другими словами если кто-то располагает экземпляр значимого типа в локальных переменных или параметрах метода, то расположение идёт на стеке (не выделяя дополнительной памяти в куче). Эта операция -- та, о которой стоит мечтать, т.к. именно она максимально быстра и эффективна. Если же структура располагается в полях ссылочного типа (класса), то под нее операция выделения памяти также не будет вызывана: ведь она является структурной частью этого ссылочного типа и фактически аллоцируется вместе с аллокацией самого ссылочного типа (являясь его частью). Однако всё гораздо сложнее с ссылочными типами. Ведь, если речь идет о них, то мы имеем целый набор сложностей при выделении памяти под их экземпляры. Причём, что самое печальное, а может быть даже обидное -- от нас почти никак не зависит, на какой из алгоритмов выделения памяти мы напоремся: на самый быстрый вариант из четырёх или же на самый тяжеловесный.

## Время жизни ссылочных типов

### Общий обзор

Для целостности картины при дальнейшем чтении рассмотрим особенности времени жизни экземпляров ссылочных типов данных. Ссылочные типы обладают следующими свойствами в вопросе времени собственной жизни:

  - У ссылочных типов в отличии от значимых детерминированное начало жизни. Другими словами они порождаются тогда и только тогда, когда кто-либо запросил их создание оператором `new`;
  - Однако, они имеют недетерминированное освобождение: мы не знаем, когда произойдет освобождение памяти из под них. Мы не можем вызвать GC для конкретного экземпляра даже для случая с `Large Objects Heap`, где эта операция могла бы быть вполне уместна;

Эти два свойства дают нам немного пищи для размышлений:

  - экземпляры классов уничтожаются в случайное время в неопределенно отдаленном будущем;
  - их уничтожение обуславливается утерей ссылок на них;
  - поэтому с одной стороны это значит, что операция освобождения последней ссылки на объект превращается в детерменированную операцию "удаления" объекта из зоны видимости приложения. Объект ещё существует, но недосягаем для всего приложения;
  - однако, с другой стороны мы далеко не всегда в курсе, какое именно обнуление ссылки будет последним, что лишает нас свойства детерменированности в обнулении последней ссылки.

Еще одним очень важным свойством является наличие виртуального метода финализации объекта. Этот метод вызывается во время срабатывания сборщика мусора: т.е. в некотором неопределенном будущем. И необходим данный метод для одного: корректного закрытия неуправляемых ресурсов, которыми владеет объект в тех и только тех случаях, когда что-то пошло не так (например, было выброшено необработанное исключение, сломавшее возможность дойти до вызова `Dispose()`) и программа более не сможет самостоятено это сделать (код, который отвечает за освобождение данных ресурсов никогда более не вызовется вследствие срабатывания исключительной ситуации). И, поскольку время вызова данного метода ровно как и освобождение памяти из под объекта от нас не зависят, его вызов также не является детерминированным. Мало того, он является асинхронным, т.к. осуществяется в отдельном потоке во время исполнения приложения. Это важно помнить, т.к. если, например, ваше приложение имеет логику повторной попытки работы с ресурсом и если произошла какая-то ошибка (например, `ThreadAbortException`), в результате которой по результатам предыдущей попытки не был вызван `Dispose()` и ресурсы "повисли" в очереди на финализацию, то это значит, что вы не сможете открыть этот ресурс (например, файл), пока не отработает очередь на финализацию, в которой этот ресурс будет освобождён.

> Однако, там где есть неопределенность, программисту всегда хочется внести определенность и как результат, возник интерфейс `IDisposable`, речь о котором пойдет чуть позже, в следующей главе. Я сейчас могу сказать лишь одно: он реализуется если необходимо, чтобы внешний код мог самостоятельно отдать команду на освобождение ресурсов объекта. Т.е. детерминированно сообщить объекту, что он более не нужен.

Таким образом с точки зрения программиста, использующего объект возникает псевдодетерменированное разрушение объекта: перевод объекта в разрушенное состояние при помощи метода `Dispose()`. Память из-под объекта не освободжается, ссылка на объект всё еще выставлена: просто необходимо его разрушение в конкретной точке приложения. Например, потому что этот объект обязан отписаться от каких-то событий либо же освободить какой-то ресурс. 

### Перерождение как оптимизация

Однако, если заглянуть с другой стороны: что такое начало и конец времени жизни сущности? Мир программного обеспечения -- мир виртуального, который обусловлен законами, созданными нами самими. Т.е. другими словами, время начало жизни сущности -- это время, которое сообщили нам, что сущностью можно пользоваться. Правильно? Кто нам мешает начать управлять этим поведением? Давайте введем понятие перерождения сущности: когда, закончив своё существование, сущность возродится для переиспользования кем-то другим.

Начало времени жизни - по идее - это когда под объект выделена память и когда он переведен в корректное состояние. Заходя с конца процесса инициализации, как с наиболее понятного нам с вами, переводом в корректное состояние объекта занимается конструктор объекта. Именно он инициализирует все пустые поля осмысленными значениями, которые наделяют его жизнью. Но что такое "выделение памяти"? В терминологии оператора `new` языка C# - это запрос адреса оперативной памяти, который размечен под объект определенного типа. Т.е. выделение в некотором внешнем массиве (не более того) памяти под поля объекта + его системные поля (`SyncBlockIndex`, VMT) и отдача адреса этого участка коду вашей программы. Другими словами... это чисто программная вещь - выделить вам память под что-то. Резальтат точно таких же алгоритмов, которые мы с вами пишем ежедневно.

Мы можем сделать новый слой управления памятью. Где не будет сборок мусора, а окончание жизни объекта станет детерминированным - для нас. Этот способ называется по-разному. Чаще всего - pooling. Реже - кэшированием. Т.е. использование коллекций объектов, их выдача по запросу и забор обратно в коллекцию, когда объект более не нужен. 

Простейший пулинг представлен ниже:

```csharp
public static class ObjectsPool<T> where T : class, new()
{
    private Queue<T> objects;
    private const MinPoolSize = 32;

    static ObjectsPool()
    {
      objects = new Queue<T>(MinPoolSize);
    }

    public static T Get()
    {
      if(objects.TryDeqeue(out var instance))
      {
        return instance;
      }
      return new T();
    }

    public static void Return(T instance) => objects.Enqeue(instance);
}
```

Этот пул - максимально простой пул "без обязательств". Т.е. объекты, выданные им вовсе не обязательно возвращать в него. Если не вернуть, ничего страшного не произойдёт: они просто будут собраны GC. Минус у него один: `ObjectsPool<T>.Get()` может вернуть "грязный" объект. Т.е. уже кем-то проинициализированный. А для того чтобы он был очищен, придётся делать реализацию `IDisposable`. Но поскольку мы пока что рассматриваме теорию, останавливаться на этом пока не будем.

Зато он демонстрирует новую возможность: детерминированное время начала жизни сущности (когда надо - запросили из пула), детерминированное окончание (когда больше не нужен - вернули обратно в пул) и отсутствие сборки мусора (ссылка есть либо из пула либо у вас - пока объект в работе). Помимо всего прочего если вы забыли вернуть объект в пул ничего страшного не случится: он будет собран GC.

Но позвольте, Станислав, скажите вы. Вы тут пожонглировали мячиками, написали пару методов и назвали это - новым слоем управления памятью. Ну в общем так и есть! Что мы видим? Можно получить новый объект? Можно! Пользоваться им можно? Можно! А освободить память можно? Да, тоже можно! И чем не менеджер памяти? Простоват? И прекрасно. Глуповат? Зато быстрый. А за более сложными примерами мы еще обратимся позже: в главе по практикам использования пройденной теории.

### Неинициализарованное состояние как консистентное

Что же касается структур, то тут возникает много мыслей относительно их времён создания и удаления. Суть структуры - не только структурировать некий участок памяти, разбивая его на поля. Но и быть структурной частью чего-либо. Ведь она ложится среди полей класса, переменных или параметров (там, где она объявлена) "на месте", как обычное целое число, структурируя тем самым эту память. А потому нет выделения памяти конкретно под структуру. Она просто объявляется и всё. Инициализация структуры -- обычное обнуление памяти. И пользуясь этим, можно сделать так, что конструктор не понадобится.

Структуры обладают конструктором по-умолчанию, который нельзя переопределить. Можно написать так:

```csharp
SomeStruct structInstance1 = default;
SomeStruct structInstance2 = new SomeStruct();
```

И это будет легальной инициализацией, а обе строчки - идентичны. Мало того, оператор `new` введен просто для консистентности с классами, а потому может возникать путаница: зачем сначала создавать, а потом копировать? Незачем. Запись означает - есть проинициализированный нулями участок памяти. Однако этим свойством можно пользоваться при проектировании структур. Возьмем широко известную структуру `CancellationToken`:

```csharp
struct CancellationToken
{
   private CancellationTokenSource m_source;
}

var token = default(CancellationToken);
```

Здесь `token` просто проинициализирован нулями. А значит, на `m_source == null` можно опираться как на токен, который никогда не перейдёт в состояние `IsCancellationRequested == true`. Т.е. как на токен, который невозможно перевести в сигнальное состояние.

Уход структуры из жизни происходит вместе с уходом из жизни хранящей её сущности: фрейма вызова метода, когда происходит выход из метода, или же класса, когда тот теряет свою последнюю ссылку. 

### В защиту текущего подхода платформы .NET

Мы никогда не задумывались (а может только я?) над тем, что было бы, будь всё по-другому: если бы память освобождалась детерминированно на уровне CLR. Текущий подход с автоматической памятью, когда мы не задумываемся, где выделять объекты и когда их освобождать нам не всегда нравится: ведь бывают случаи, когда готовых к освобождению объектов накапливается слишком много и их освобождение тормозит всё приложение. Однако, в защиту текущего подхода давайте немного отвлечемся на сценарии, о которых иногда начинаешь задумываться, мечтая сменить текущий набор алгоритмов:

  - если вместо того чтобы освобождать объекты по срабатыванию GC мы будем освобождать их с потерей последней ссылки, что произойдёт? Вот наш код присваивает некоторой переменной `null`. Тогда получается, что на каждом присвоении необходимо проверять, идёт ли присвоение `null` или какой-либо другой рабочей ссылки. Если да, то надо понять, последняя ли это была ссылка. Каждый раз считать входящие с кучи ссылки, перебирая все объекты SOH/LOH -- дорого. Значит, надо чтобы каждый объект считал все входящие ссылки сам: инкрементируя и декрементируя счётчик на каждой операции. Это -- дополнительное место + дополнительные действия. Плюс ко всему получается, что мы уже не можем сжимать кучу: после каждого присваивания это делать слишком дорого: подходит только метод `Sweep`. Как мы видим, уже на поверхности всплывает очень много проблем, не говоря уже о подробностях;
  - если ввести оператор `delete`, чтобы как в C++ освобождать объекты по требованию, дополнительно воскрешает деструктор, как средство детерминированного освобождения памяти: ведь если мы освобождаем объект оператором `delete`, необходимо таким же образом освобождать те объекты, которыми этот объект владеет. Значит, необходим метод, который будет вызываться при разрушении объекта: деструктор экземпляра типа. Это приведет к увеличению сложности разработки и удорожанию сопровождения программ: утечки будут постоянно. Плюс ко всему прочему возникнет путаница при освобождении памяти: мы лишаемся возможности освобождать её в последней точке использования. Т.е. теперь мы должны это делать в строго отведенном месте.
  - если вводить смешанный алгоритм: в целом чтобы работало как сейчас, но чтобы был оператор `delete`. Например, вы мне скажете, вам захочется освобождать массивы данных, которые были использованы под скачивание изображений ровно в определенный момент. Потому что если наше приложение качает изображения друг за другом и при этом они достаточно быстро становятся не нужны, то мы вхолостую выделяем кучу памяти, которая быстро копится и приводит к вызову GC. Это особенно актуально для мобильных приложений на Xamarin и элемента управления "виртуальный список", где при быстром скролле изображений они в больших количествах грузятся, а потом становятся ненужными. Если удалять их сразу, то не будет ситуации с большим GC, который испортит анимацию прокрутки. Однако, тут возникнут сложности для GC. При ручном освобождении памяти, последняя в свою очередь станет фрагментирована и может перестать вмещать в себя те массивы данных, которые вы запросите под следующие изображения. Как следствие -- всё равно произойдет GC. Если блоки памяти с ручным управлением располагать в LOH, то ручное освобождение хорошо "ляжет" на его алгоритмы. Однако, всё также будет приводить к фрагментации и дальнейшему срабатыванию полного GC. Единственно верное решение -- использовать пул массивов и Span/Memory для доступа к поддиапазону индексов. Но тогда зачем вводить `delete`?

Тогда получается, что текущее решение -- прекрасно и надо просто научиться им правильно пользоваться. Этим мы чуть позже и займемся.

### Предварительные выводы

Из всего сказанного можно увидеть, что у любого объекта есть некоторое время его существования. Это может показаться тривиальной мыслью, которая лежит на поверхности, но не все так однозначно:

  - важно понимать, как, когда и при каких иных условиях *объект создается*. Ведь его создание занимает некоторое не всегда короткое время. И вы не можете заранее угадать, по какому алгоритму он будет создан: простым переносом указателя в случае наличия места в allocation context, вследcтвии необходимости расширения или переноса allocation context в памяти, необходимости сжатия эфимерного сегмента или же необходимости создания нового эфимерного сегмента с его полным структурированием. Возможно, при большом траффике таких объектов стоит задуматься над пуллингом таких объектов. Тогда время их создания станет *предсказуемым*, а GC не будет обращать на них слишком много внимания; 
  - также стоит понимать, насколько "популярным" будет объект *во время его жизни* и как долго он будет существовать: какое количество иных объектов будет на него ссылаться и как долго. Этот фактор влияет как на сборку мусора, фрагментацию кучи, время создания других объектов и что самое интересное -- на время обхода графа объектов в фазе маркировки достижимых объектов сборщиком мусора. Например, при том же пуллинге, объекты, которые там долго находятся, уходят во второе поколение на веки вечные. Но при инициализации они могут получить ссылки на младшее поколение и тогда включается механизм карт при сборке мусора. Если пул заполняется по требованию (как в нашем простом примере чуть выше по тексту), его объекты будут перемешаны с объектами, созданными другим кодом. Что замедлит анализ ссылок в младшие поколения.  
  - а также, что логично и очень важно: как объект будет *достигать* состояния освобождения (состояние выброшенности звучит грустно). Это значит, будет ли осуществляться детерминированное его разрушение или нет. Например, при помощи `IDisposable.Dispose`
  - и освобождаться -- быть подхваченным Garbage Collector'ом с дальнейшей возможностью вызова финализатора.

Каждый из этих этапов определяет производительность приложения и логичность его архитектуры. Далее мы рассмотрим каждый этап в отдельности.