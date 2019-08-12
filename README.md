# SysConfigFile 2.0

[project]:https://github.com/mazzy-ax/SysConfigFile
[license]:https://github.com/mazzy-ax/SysConfigFile/blob/master/LICENSE
[ax2009]:ax2009
[ax2012]:ax2012
[ax4]:ax4
[XPath]:https://ru.wikipedia.org/wiki/XPath

[SysConfigFile][project] &ndash; это класс, который позволяет получать значения параметров из конфигурационных xml-файлов в [Microsoft Dynamics AX 2009][ax2009], [Microsoft Dynamics AX 2012][ax2012] и [Axapta 4.0][ax4].

Как правило, на проектах используется несколько экземпляров (инстансов) Аксапты - рабочая, тестовая, для разработки и т.д. Причем зачастую рабочий экземпляр полностью копируется в один из тестовых экземпляров (и база, и приложение). Кроме того, зачастую создается экземпляр-зеркало рабочего экземпляра. Зеркало используется для BI-отчетов.

Получается, что в разных экземплярах и база, и приложение может сопадать. Но при этом все равно нужно обеспечить различную работу этих экземпляров. Например, только рабочий экземпляр может рассылать письма, тестовым экземплярам запрещено запрашивать внешние системы или отвечать на запросы внешних систем. А зеркало, например, не должно ничего разносить.

Такие параметры экземпляра (инстанса) удобно хранить централизовано во внешнем файле в отдельном каталоге на сервере.

## Класс SysConfigFile

Класс `SysConfigFile` предполагает:

1. параметры хранятся централизовано на сервере в небольших xml-файлах с расширением `.config`
1. имя config-файла по умолчанию `Axapta.config` (имя можно задать в `new` или в конструкторе)
1. каталог по умолчанию `%Appl%\Config` (каталог можно задать в `new` или в конструкторе, изменить можно в parm-методе)
1. параметр хранится как xml-элемент, а значение параметра хранится как текстовое значение xml-элемента. Например, `<name>Microsoft Dynamics AX</name>`
1. пустые xml-элементы трактуются как `true` методом `getBoolean` и как пустое значение методом `get`. См. [Пример config-файла](#Пример-config-файла)
1. валидность config-файл можно проверить при помощи xsd-схемы. xsd-схему можно хранить в ресурсах Аксапты или в xsd-файле рядом с config-файлом. Также можно явно указать схему в конструкторе

В простейшем случае программисту достаточно написать название параметра, значение которого он хочет получить. Но для сложных случаев в качестве названия параметра можно использовать [XPath-выражения][XPath].

Формат XML сам по себе не ограничивает уникальность элементов внутри, поэтому класс `SysConfigFile` позволяет получать как одно найденное значение (первое), так и список всех найденных значений. Кроме того, метод `getBy` позволяет получить значения в нужном порядке, а не в порядке размещения в xml-файле. См. также раздел [Направления для развития](#Направления-для-развития).

Класс `SysConfigFile` будет работать даже в том случае, если config-файл отсутствует. В этом случае get-методы будут возвращать значения по умолчанию. Однако класс позволят программисту явно сказать, что параметр обязательно должен присутствовать в файле при помощи `assert`-, `ensure`- и `check`- методов. Обратите внимание, что статический метод `::Value` бросает исключение, а статический метод `::ValueOrDefault` возвращает значение по умолчанию, если параметра нет в конфигурационном файле. См. [Устойчивость](#Устойчивость).

Для работы с XML-файлами класс `SysConfigFile` [пока](#Направления-для-развития) использует `XmlDocument`. `XmlDocument` позволяет искать элементы внутри этого файла при помощи [XPath], но загружает xml-файл целиком в память. Поэтому следите за размером config-файла, не позволяйте этому файлу стать слишком большим. См. [Несколько разных config-файлов](#Несколько-разных-config-файлов) и [Кэширование](#Кэширование).

## Пример config-файла

```xml
<?xml version="1.0" encoding="utf-8"?>
<config>
    <id>PROD</id>
    <name>Microsoft Dynamics AX</name>
    <reportTemplateFolder>\\dax\template\</reportTemplateFolder>

    <sender>Axapta</sender>
    <sender email="note">Notification server</sender>
    <sender email="mail">Company name</sender>

    <AOS>
        <batch serverId="01@AOS" />
        <batch serverId="01@RESERV">true</batch>
    </AOS>
</config>
```

## Пример X++

Предположим, что в каталоге `%Appl%\Config\` содержится файл `Axapta.config` с текстом из примера выше. Тогда код будет работать так (см. тестовый метод `SysConfigFileTest.testExample`):

```java
SysConfigFile config = SysConfigFile::construct();

config.get('id');                       // 'PROD'
config.get('notFound');                 // ''
config.ensureExists('notFound').get();  // бросит исключение, поскольку
                                        // в данном примере параметр notFound отсутствует

config.getBoolean('AOS/batch');         // true
config.getBoolean('notFound');          // false

config.getAll('sender');                // ['Axapta','Notification server','Company name']
config.getAll('AOS/batch/@serverId');   // ['01@AOS','01@RESERV']
config.getAll('notFound');              // connull()

config.getBy(['sender[@email="mail"]','sender[not(@email)]']);  // 'Company name'
config.getBy(['sender[@email="other"]','sender[not(@email)]']); // 'Axapta'
```

Если нужно получить только одно значение, то можно написать и так:

```java
SysConfigFile::value('name');           // 'Microsoft Dynamics AX'
```

Два статических метода `Value` и `ValueOrDefault` по разному работают с отсутствующими в config-файле параметрами:

```java
SysConfigFile::valueOrDefault('notFound','DefaultValue');       // 'DefaultValue'

SysConfigFile::value('notFound');                               // бросит исключение
```

## Несколько разных config-файлов

Когда параметров становится очень много, то удобно разместить параметры в разных config-файлах. Например, отдельный config на каждый модуль. Размещение параметров в разных конфигах с одной стороны снижает вероятность конфликтов при обновлении параметров, с другой стороны делает чтение и кэширование config-файла более эффективным.

Если параметры хранятся в отдельном файле `PrintSrv.config`, то достаточно в конструкторе или в статическом методе написать:

```java
SysConfigFile::construct('PrintSrv').get('id');

SysConfigFile::value('id', 'PrintSrv');
```

При желании можно указать и каталог, где хранится конфиг (см. тестирующий класс `SysConfigFileTest`). По умолчанию класс использует каталог `Config` внутри каталога приложения. К каталогу приложения `%Appl%` гарантировано имеют доступ все AOS кластера. Если админы не запретили специально доступ к каталогам внутри `%Appl%`, то к подкаталогу `%Appl%\Config` доступ скорее всего будет.

## Валидация

### Валидация по xsd-схеме

Класс `SysConfigFile` может выполнить валидацию config-файла по [xsd-схеме](https://ru.wikipedia.org/wiki/XML_Schema_(W3C)). <https://www.w3.org/XML/Schema>

Чтобы выполнить валидацию, программист должен явно вызвать хотя бы один из методов: `ensureFileValid`, `assertFileValid` или `checkFileValid`.
Если методы не вызваны, то класс выполняют только [проверку синтаксиса](#Валидация-синтаксиса-config-файла) при чтении config-файла.

Xsd-схема может хранится централизовано в ресурсах AOT или в xsd-файле рядом с config-файлом, а может задаваться как параметр при создании класса. Класс определяет название ресурса в методе `schemaResourceName`, а название и путь xsd-файла в методе `schemaPath`.

### Проверка синтаксиса config-файла

Класс `SysConfigFile` использует аксаптовские классы для работы с xml. Эти классы всегда выполняют проверку xml синтаксиса при загрузке текста - так называемую проверку [well formed xml](https://en.wikipedia.org/wiki/Well-formed_document).

### Валидация по dtd не выполняется

Аксаптовские xml-классы не выполняет валидацию по DTD (выполняется только проверка синтаксиса DTD как подмножества синтаксиса xml).

С некоторого времени Microsoft считает работу с DTD опасной (https://msdn.microsoft.com/en-us/magazine/ee335713.aspx).

Во-первых из-за "xml bomb" (гуглите).

Во-вторых, потому что способ проверки задает создатель внутри этого xml-файла В далекие 2000е в этом не видели проблему, но уже в 2010е стали считать, что доверять создателю файла слишком наивно и опасно. Поэтому, в аксаптовских xml-классах есть метод `prohibitDtd`. По умолчанию он возвращает true.

*У автора не получилось включить процессинг `DTD` в аксапте даже передав `false` в метод `prohibitDtd`. [Дайте знать](#Помощь-проекту), если у вас получится.*

## RunOn = Server

`SysConfigFile` &ndash; серверный класс (параметр RunOn = Server в свойствах AOT). Так сделано для того, чтобы:

* класс гарантировано читал config-файл с сервера
* не нужно было расшаривать каталог с конфигами в сети (в том числе, VPN)
* не путаться с путями к файлами (на клиенте, локально на сервере, сетевые пути)
* не беспокоиться о правах доступа пользователей и AOSов к конфигам (особенно в кластере)

Но это значит, что вызов этого класса из Аксаптовского **клиента** приведет к потенциально длительному клиент-серверному взаимодействию. Постарайтесь работать с этим классом из серверных объектов и методов.

Чтобы уменьшить накладные расходы на передачу данных между клиентом и сервером, метод `getAll` возвращает `container`, а не составной объект. По этой же причине метод `getBy` принимает `container` как параметр.

Если же Аксаптовскому клиенту нужно получить значения большого количества параметров, то создайте дополнительный специализированный метод, который упакует передаваемые данные в `container`. См. также [Направления для развития](#Направления-для-развития).

## Кэширование

Класс кэширует и найденные, и отсутствующие значения в глобальном кэше. Поэтому повторный `get` возвратит значение очень быстро, без дополнительного чтения config-файла.

Поскольку `SysConfigFile` &ndash; это серверный класс, то он будет использовать серверный глобальный кэш. В [ax4] и [ax2009] в методе `globalCache` я предлагаю использовать `appl.globalCache`. В [ax2012] я предлагаю использовать `classFactory.globalObjectCache`. На своем проекте вы можете изменить код и использовать кэши из `appl`, `infolog` или `classFactory`.

Класс `SysConfigFile` может работать в `startup` процедурах, когда глобальные переменные `appl`, `info`, `classFactory` еще не инициализированы. На этом этапе класс создает свой локальный кэш, который будет жить пока живет объект `SysConfigFile`.

Кроме того, каждый объект класса `SysConfigFile` физически читает config-файл только один раз при необходмости, а потом хранит `xmlRoot` пока живет. Поэтому, если вам нужно получить несколько значений из конфига, старайтесь вызывать методы `get`, а не статические методы `::value`. Это минимизирует число физических чтений config-файла.

## SysConfigFile - неизменяемый объект

Класс `SysConfigFile` создает неизменяемые ([Immutable](https://en.wikipedia.org/wiki/Immutable_object)) объекты &ndash; все значимые параметры определяются при создании объекта, хранятся в переменных объекта и не изменяются пока объект "живет".

Некоторые внутренние переменные класса вычисляют свое значение при необходимости ([Lazy initialization](https://en.wikipedia.org/wiki/Lazy_initialization)) и не меняют значение после вычисления.

В частности, класс читает содержимое config-файла и xsd-схемы один раз при первом обращении к файлу и схеме, а в дальнейшем переиспользует уже прочитанные файлы. Соответственно значения параметров читаются из config-файла только один раз и хранятся в глобальном кэше. `SysConfigFile` возвращает значение из кэша, если оно там есть. В противном случае читает значение из xml-файла.

## Прямой доступ к xml-содержимому

Программист может получить исходный текст конфига и исходный текст xsd-схемы при помощи методов `file()` и `schema()`.

Кроме того, класс содержит приватные методы `xmlDocument()` и `xmlSchema()`, которые возвращают ссылки на объекты в памяти. Если вы хотите получить непосредственный доступ к xml-содержимому, то сделайте эти методы публичными и работайте с содержимым xml.

Однако помните, что `SysConfigFile` &ndash; серверный. Не обращайтесь к методам `xmlDocument()` и `xmlSchema()` из клиента &ndash; клиент-серверная передача отдельных объектов сопровождается огромными накладными расходами.

Также учитывайте, что в следующих версиях класс `SysConfigFile` может использовать [другие способы работы с xml](#Направления-для-развития).

## Устойчивость

Класс `SysConfigFile` разработан так, чтобы возвращать осмысленный результат в ситуациях, которые могут трактоваться как ошибки. Класс минимизирует вероятность возникновения исключений и в явном виде бросает исключения только в `ensure`- методах. Однако само ядро Аксапты может выбросить исключение (exception) во время парсинга xml в методах `XmlDocument::newFile()` и `XmlSchema::newFile()` если:

* xml-файл или xsd-схема содержит незакрытые теги, неправильно вложенные теги, `"`, `&`, `<`, `>` или другие специальные символы
* xsd-схема содержит непредусмотренные [стандартом](https://www.w3.org/XML/Schema) элементы, атрибуты или значения

Обработка исключений в классических Аксаптах имеет особенность: catch блоки внутри транзакции не обрабатываются, Аксапта выпрыгивает за самый внешний transaction-блок и ищет catch уже там (см. раздел `Exception handling inside transaction` в документации). Поэтому обработка исключений становится не надежной, если код может выполняться внутри транзакции.

Поэтому если config-файлы содержат синтаксически правильный xml ([well-formed xml](https://www.w3.org/TR/xml/#sec-well-formed) + [статья](http://softwaremaniacs.org/blog/2005/10/02/well-formed-valid-conformant/)), то исключения могут возникнуть только после вызова `ensure`-, `assert`- методов и статического метода `::Value`, а остальные методы вернут более-менее осмысленный результат.

## Зависимости

В версии 2.0 класс `SysConfigFile` использует только стандартный функционал Аксапты и несколько `.net` методов, которые обернуты в необходимые `CodePermission`. Поэтому вам не придется ни импортировать дополнительные проекты, ни устанавливать дополнительные `CodePermissions` в свой код.

Класс `SysConfigFile` использует:

* System.IO.File::Exists
* System.String::Copy
* System.String.Trim

Тестовый класс `SysConfigFileTest` использует:

* System.IO.Path::GetTempPath
* System.IO.Directory::CreateDirectory
* System.IO.Directory::Delete

## Направления для развития

* использовать `.net` классы для работы с xml вместо Аксаптовских (проверить производительность, потребление/утечку памяти и работу `.net` GC в связке с GC-Аксапты разных версий)
  * читать из [зашифрованных разделов xml](https://docs.microsoft.com/en-us/dotnet/standard/security/how-to-encrypt-xml-elements-with-asymmetric-keys), если получится перейти на `.net` xml
* дальнейшая минимизация трафика между клиентом и сервером Аксапты: принять несколько параметров в контейнере и возвратить контейнер значений (развить метод `getAll`)
  * возможно, стоит вернуть упакованный map
* добавить `getDate`, `getTime`, `getDateTime`, которые будут умно конвертировать строку формата, использующийся xsd-схемами `xs:date`, `xs:time`, `xs:dateTime`, в Аксаптовские объекты с типом `Date`, `Time` и `DateTime` соответственно
  * Насколько я понимаю, это один из вариантов формата [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601): `yyyy-mm-ddThh:mm`
  * Подумать в каком методе и как правильно учесть часовой пояс `yyyy-mm-ddThh:mmZ###` (а как для ax4?)
  * Подумать стоит ли делать три метода или один (а как для ax4?)
  * см. также `Global::xmlstring`
* добавить работу с `namespace` в config-файле (нужно ли?)
* сделать класс, который сможет читать параметры из реестра, а не из файла. В этом случае администраторы смогут раскатывать значения по серверам через групповые политики

## Известные проблемы

### 1. XML чувствителен к регистру в запросе, кэш не чувствителен к регистру

*[Дайте знать](#Помощь-проекту), если у вас есть предложения по данному пункту.*

Прежде всего, привыкшие к Аксапте программисты постоянно забывают о регистрозависимости XML. И постоянно делают попытки получить `name` вперемешку с `Name`. А это разные элементы с точки зрения XML.

На XML накладывается Аксаптовский НЕ зависимый от регистра кэш. Поэтому следующие строки кода будут вести себя по разному (внезапно!):

```java
SysConfigFile::value('Name');   // бросит исключение, поскольку xPath запрос не найдет такого раздела в XML-файле
SysConfigFile::value('name');
```

```java
SysConfigFile::value('name');   // 'Microsoft Dynamics AX'
SysConfigFile::value('Name');   // 'Microsoft Dynamics AX', а не исключение! поскольку параметр успешно найден в кэше
```

Эта проблема проявляется только в [ax4] и [ax2009], очень редко и не особо мешает использовать данный (в целом полезный) класс. Класс `SysGlobalObjectCache` в [ax2012] использует регистрозависимые ключи и не имеет данной проблемы. Поэтому я решил опубликовать класс с этой проблемой.

### 2. Хочется работать с config-файлами на клиенте

*[Дайте знать](#Помощь-проекту), если у вас есть предложения, которые не снижают ни производительность, ни уровень гарантий в основном сценарии работы.*

`SysConfigFile` &ndash; серверный класс. Это фича. Но иногда нужно работать с config-файлами на клиенте (POS-терминалы, PinPad, ключи безопасности и прочее оборудование, сертификаты и софт, привязанный к пользовательскому компьютеру). Конечно, с клиентскими config-файлами можно работать как с обычными xml-файлами. Если же хочется получить сервис `SysConfigFile` на клиенте, то можно:

* создать дубль `SysConfigFileClient`
* изменить свойство `RunOn`
* изменить `WinAPIServer` на `WinAPI`
* изменить метод `parmDefaultDirectory` или явно указывать каталог перед запросом значений из конфига

## Благодарности

Спасибо [Сергею Чечкину](sergey@chechkin.com) за похоже единственно возможный в аксаптовских классах способ верификации xml по схеме. Спасибо всем за конструктивное обсуждение проекта.

## Disclaimer

* Названия классов и методов, иерархия и порядок вызовов в наборе классов будут по возможности сохраняться, но это не гарантируется - в будущих версиях SysEnumerators все может измениться.
* Код в xpp-файлах конвертирован из xpo только для удобства использования человеком. Оригиналом является код в xpo-проектах, отличия между xpo и xpp всегда трактуются в пользу текста из xpo-проектов.
* Проект выложен "как есть" под лицензией [MIT][license]: вы можете использовать данный код как угодно безо всяких отчислений, автор не дает никаких гарантий и не несет ответственности за возможный эффект от использования кода на проектах.

## Прочее

* проект сознательно сделан для классических версий Аксапты
* в проекте сознательно не используется xmldocs
* README и комментарии сознательно сделаны на русском языке
* ошибки в ensure методах записаны простой строкой на русском языке и не используют меток

## ChangeLog

* [CHANGELOG.md](CHANGELOG.md)
* <https://github.com/mazzy-ax/SysConfigFile/releases>

## Помощь проекту

Буду признателен за ваши замечания, предложения и советы по проекту как в разделе [Issues](https://github.com/mazzy-ax/SysConfigFile/issues), так и в виде письма на адрес <mazzy@mazzy.ru>

Мазуркин Сергей (mazzy)
