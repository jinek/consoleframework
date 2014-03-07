﻿Описание работы XAML
==

XAML обрабатывается следующим образом. Парсер проходит все элементы подряд, и превращает их в объекты. Из главного элемента (root element) выпарсиваются определения используемых пространств имён. Экземпляр объекта создаётся при встрече элемента (атрибута). Далее, если это атрибут, то его значение присваивается свойству описываемого объекта сразу, а если это вложенный элемент, то его содержимое будет создано рекурсивно из того, что внутри этого элемента. Присваивание сконструированного объекта свойству вышестоящего объекта происходит в момент завершения парсинга описания его содержимого, то есть при нахождении парного закрывающего тега. В этот момент объект полностью сконфигурирован вместе со всеми вложенными объектами, и будет присвоен свойству вышестоящего объекта (либо добавлен в него, если свойство вышестоящего объекта - одна из поддерживаемых коллекций). Если же мы дошли до конца XAML-документа, то сконструированный объект возвращается в качестве результата.

## Простой пример

```
<Window>
    <Window.Content>
        <Panel>
            <TextBlock Name="text" HorizontalAlignment="Center"></TextBlock>
            <Button Name="btnMaximize" Caption="Maximize"></Button>
            <Button Name="btnRestore" Caption="Restore"></Button>
        </Panel>
    </Window.Content>
</Window>
```

Как выполняется обработка этого XAML-документа:
- Создаётся `Window`, вызывается конструктор по умолчанию.
- Встретили вложенный элемент `Window.Content`. Так как он начинается так же, как и тег текущего конфигурируемого объекта, то, это определение свойства - `Content`. Если бы вложенный тег не начинался с `Window.`, то парсер бы решил, что мы начали таким образом определять Content-свойство. Content-свойство по умолчанию имеет название `Content`, но может быть переопределено атрибутом `ContentPropertyAttribute`.
- Создаётся Panel, вызывается конструктор по умолчанию
- Встретили вложенный элемент, не начинающийся с `Panel.`. Значит, мы должны определить Content-свойство для класса Panel - в этом случае это будет свойство `UIElementCollection Children`. Все вложенные элементы (TextBlock и два Button) теперь будут определять значение этого свойства.
- Создаётся `TextBlock` (тоже конструктор по умолчанию), устанавливаются его свойства `Name` и `HorizontalAlignment`. Тип свойства `Name` - *String*, поэтому конвертеры не применяются. Тип свойства `HorizontalAlignment` - *enum HorizontalAlignment*, и к нему может быть применен стандартный механизм преобразования из String в Enum.
- `TextBlock` создан и все свойства сконфигурированы, мы встретили закрывающий тег, и теперь должны понять, куда его присвоить. Текущий конфигурируемый объект - `Panel`, и мы сейчас находимся в его Content-свойстве - `Children`. Оно реализует `IList`, поэтому созданный TextBlock добавляется в него вызовом метода `Add`.
- Аналогичным образом в Panel.Children добавляются обе кнопки
- Мы достигли закрывающего тега `</Panel>`. Это означает, что панель полностью сконфигурирована и будет присвоена свойству вышестоящего объекта - `Window.Content`. Свойство обычное (не коллекция), тип - *Control*, поэтому преобразований не нужно.
- Тег `</Window.Content>` говорит нам о том, что определение свойства `Window.Content` закончено.
- И наконец тег `</Window>` завершает конфигурирование объекта Window, который и возвращается в качестве результата.

Таким образом, приведённый XAML эквивалентен следующему коду:
```
Window window = new Window();
Panel panel = new Panel();

TextBlock textBlock = new TextBlock();
textBlock.Name = "text";
textBlock.HorizontalAlignment = HorizontalAlignment.Center;
panel.Children.Add(textBlock);

Button button1 = new Button();
button1.Name = "btnMaximize";
button1.Caption = "Maximize";
panel.Children.Add(button1);

Button button2 = new Button();
button2.Name = "btnRestore";
button2.Caption = "Restore";
panel.Children.Add(button2);

window.Content = panel;
```

## Система Content-свойств аналогична используемой в WPF

По умолчанию названием Content-свойства будет `Content`. Если вы хотите, чтобы в качестве Content-свойства выступало другое свойство, то нужно пометить класс атрибутом `ContentPropertyAttribute`:

```csharp
[ContentProperty("Controls")]
public class Grid : Control
```

## Как происходит преобразование типов и как обрабатываются коллекции

Преобразование типов производится только встроенное - из строк в числа, перечисления и некоторые встроенные
конвертеры для структур (`Thickness` - для задания `Margin`). Если нужно использовать кастомный конвертер, его нужно вызвать с помощью расширения разметки `Convert`:

```xml
<Window xmlns:x="http://consoleframework.org/xaml.xsd"
        xmlns:converters="clr-namespace:Binding.Converters;assembly=Binding">
    <Window.Resources>
        <string x:Key="testItem" x:Id="testStr">5</string>
        <converters:StringToIntegerConverter x:Key="2" x:Id="str2int"></converters:StringToIntegerConverter>
    </Window.Resources>
    <Panel>
        <TextBox MaxLenght="{Convert Converter={Ref str2int}, Value={Ref testStr}}"/>
    </Panel>
</Window>
```

Поддерживаются любые коллекции, реализующие `IList`, `ICollection<T>` или `IDictionary<string, T>`.
Если при обработке закрывающего тега выясняется, что свойство вышестоящего элемента (свойство, которое мы
определяем текущим объектом) реализует `ICollection<T>`, то текущий сконфигурированный объект будет добавлен
в эту коллекцию вызовом Add(T obj) - вместо стандартного поиска конвертера и последующего вызова сеттера.
Таким образом, для коллекций сеттер не нужен - нужен только геттер.
Аналогично обрабатываются свойства типа `IDictionary<string, T>`. Пример:

```xml
<Window.Resources>
    <item x:Key="1">Строка</item>
	<item x:Key="2">Строка 2</item>
</Window.Resources>
```

Во время обработки тега первого закрывающего тега </item> значение "Строка" будет добавлено в коллекцию
Window.Resources.

Так как в случае параметризованных свойств парсер знает тип T, то перед тем, как вызвать метод Add (это
справедливо и для коллекций, и для словарей) будет произведена проверка на совместимость типа текущего 
сконфигурированного объекта и T-типа-аргумента коллекции/словаря. Если типы не совместимы, будет выполнена
попытка поиска соответствующего конвертера. Если же свойство реализует непараметризованный `IList`, то объект будет добавлен без преобразований.

## Пространства имён

При вызове парсера XAML ему в качестве параметров передаётся набор пространств имён по умолчанию. Это список пространств имён CLR (путь к неймспейсу плюс имя сборки), в которых парсер будет искать типы создаваемых объектов (по именам тегов) и используемых расширений разметки. Все пространства имён, которые не включены в список дефолтных, должны быть указаны в декларации главного элемента XAML-документа.

Пример:

```xml
<my:Window Name="window2" Title="Очень длинное название окна"
        xmlns:x="http://consoleframework.org/xaml.xsd"
        xmlns:my="clr-namespace:ConsoleFramework.Controls;assembly=ConsoleFramework"
        xmlns:converters="clr-namespace:Binding.Converters;assembly=Binding"
        xmlns:xaml="clr-namespace:ConsoleFramework.Xaml;assembly=ConsoleFramework">
    <!-- Здесь можно использовать типы и расширения разметки из всех указанных
         пространств имён (не забывая указывать при этом префиксы) -->
</my:Window>
```

## Создание объектов произвольного типа и указание аргументов конструктора

Обычные объекты, создаваемые в XAML, требуют наличия конструктора по умолчанию. Если необходимо создать
экземпляр класса, у которого нет конструктора по умолчанию, можно создать factory-класс и сделать это
с помощью него. Но есть и встроенный класс ObjectFactory, который умеет создавать объекты любого типа,
вызывать конструктор с определёнными аргументами и наполнять его свойства значениями, аналогично тому,
как это бы работало, если бы мы создавали этот объект в XAML напрямую. Например, у нас есть класс

```
class TestClass<T>
{
    public TestClass( int intProperty ) {
        IntProperty = intProperty;
    }

    public int IntProperty { get; set; }

    public string StringProperty { get; set; }

    public T TProperty { get; set; }
}
```

С помощью ObjectFactory создать его экземпляр в XAML можно следующим образом:

```xml
<object TypeName="ConsoleFramework.Xaml.TestClass`1[System.String]">
    <int x:Key="1">66</int>
    <string x:Key="IntProperty">55</string>
</object>
```

То, что передаётся в `TypeName`, резолвится через вызов *Type.GetType(string assemblyQualifiedName)*, поэтому
возможно, что придётся указать полное имя типа вместе с именем сборки.

Устанавливая `x:Key` в число, мы говорим фабрике, что это - аргумент конструктора (с соответствующим индексом).
Если же `x:Key` не является числом, то `x:Key` интерпретируется как название свойства.
При определении значений аргументов конструкторов и свойств можно использовать все обычно доступные в этом
контексте инструменты - задание текстом или расширением разметки, или опять же с помощью расширения разметки
получить ссылку на ранее созданный объект по `x:Id`.

Работает это так: создавая экземпляр ObjectFactory, мы заполняем его Content-свойство, которое является
`Dictionary<string, object>`. Но после завершения конфигурирования этот объект заменяется тем, который мы
в нём описываем. Механика аналогична тому, как работают "примитивы" string, int и так далее, более того,
они все реализуют интерфейс `IFactory`, который позволяет подменять объект на другой (определяемой логикой
фабрики) в момент завершения разбора завершающего тега.

## Стандартные атрибуты

##### x:Id

Позволяет задавать идентификатор любого инстанцируемого внутри XAML объекта, чтобы ссылаться на него потом при помощи расширения разметки `Ref`.

##### x:Key

Задаёт ключи для коллекций `IDictionary<string, T>`.

__<p style="color: red">Важно!</p>__

Чтобы использовать стандартные атрибуты (`x:Key` или `x:Id`), в рутовом элементе XML обязательно должна 
быть ссылка на соответствующее пространство имён, причём заданное не через clr-namespace, а обычным образом - ссылкой на xaml.xsd:

```
<Window xmlns:x="http://consoleframework.org/xaml.xsd">
</Window>
```

Префикс `x` может быть заменён на любой другой.

## Стандартные расширения разметки

##### Ref
Позволяет получить ссылку на другой объект по его `x:Id`:
```
{Ref Ref=myObject}
или
{Ref myObject}
```

Поддерживаются ссылки вперёд (механизм реализации полностью аналогичен WPF - через fixup tokens).

##### Type
Позволяет получить тип (объект типа `Type`) по имени:
```
{Type TestProject1.Xaml.XamlTest+XamlObject, TestProject1, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null}
```

## Отличия от WPF

Если вы знакомы с WPF, то проще не изучать документ целиком, а просто прочитать отличия:
- Нет Attached Properties
- Не генерируется код метода `InitializeComponent()` partial класса, а создаётся объект в рантайме
- Так как не генерируется код, то нет подписки на события - как обычные, так и Attached Events
- Так как не генерируется код, то и не генерируются поля классов по `x:Name`
- Нет поддержки включения словарей (хотя это, скорее всего, будет потом добавлено)
- Вместо `x:Name` - `x:Id`
- Возможные расхождения в способах обработки преобразований, добавления в коллекции - для уточнения механизма нужно всё-таки изучить соответствующие разделы этого описания

## Отличия в синтаксисе расширений разметки
- Не поддерживаются XML-токены типа `&quot;` - если нужно что-то заэкранировать, используйте обратный слеш
- В одинарных кавычках нельзя писать неэкранированные символы `=,{}` - всё нужно экранировать
- В расширения разметки всегда передаётся один объект в качестве Data Context, а не резолвится как в WPF относительно текущего значения Dependency property `DataContext`. Это во-первых из-за отсутствия поддержки Dependency properties, а во-вторых, для простоты концепции XAML парсера и для возможности не интегрировать его с абстракциями, необходимыми именно для создания UI.