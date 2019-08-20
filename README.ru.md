﻿## DataCache

Реализация кэша для хранения данных в ASP.NET WebForms реализована настолько удачно, 
что не требует никаких улучшений и подходит для любого класса приложений. В ранние годы платформы .NET, 
за неимением ничего лучшего, разработчики зачастую импортировали пространство имён 
`System.Web` даже в WinForms приложения, чтобы иметь возможность работы с кэшем. 

Предлагаемый повсеместно шаблон работы с кэшем очень прост и практичен:

```csharp
myClassName item = HttpRuntime.Cache["Key1"] as myClassName;
if (item == null)
{
    item = /* get value from database or from web service or from somewhere else */;
    HttpRuntime.Insert("Key1", item);
}
// using the item that has been populated from cache or from storage
```


Тем не менее, предлагаемая в ASP.NET реализация кэша не включает нескольких простых,
но востребованных, в некоторой степени, функциональностей.

##### Параметры хранения по умолчанию

Несмотря на то, что метод добавления значения в кэш
[Cache.Insert(String, Object)](https://docs.microsoft.com/en-us/dotnet/api/system.web.caching.cache.insert?view=netframework-1.1#System_Web_Caching_Cache_Insert_System_String_System_Object_)
пригоден для большинства случаев, зачастую желательно указание параметров хранения в кеше по умолчанию или 
на основе свойств web-приложения, хранящихся в файле Web.config, а иногда даже и полного отключения кеша для всего приложения.

##### Типизация данных и значения по умолчанию

ASP.NET кэш оперирует только экземплярами класса `object`. В большинстве случаев это не представляет проблемы, 
даже со значениями `Value types`, которые можно привести к `Nullable types`. Хотя дело конечно же в личном восприятии 
исходного кода, хотелось бы иметь типизированные методы извлечения данных:
```csharp
myClassName item = HttpRuntime.Cache["Key1"] as myClassName; // dafault extraction
myClassName item = DataCache.Get<myClassName>("Key1"); // typed extraction with generic methods
```
Если не устраивает типизированный вариант, то вполне можно использовать стандартный вариант кода. По сути, эти два варианта идентичны.
Разница только в дополнительной проверке типа и значений по умолчанию, которые предоставляют типизированные методы, например:
```csharp
myClassName defaultValue = new myClassName(/* init values */);
myClassName item = DataCache.Get<myClassName>("Key1", defaultValue); 
/* if there is nothing in the cache, then the item will be the defaultValue */
```

##### Deep copied data

ASP.NET кэш всегда возвращает объект, хранящийся в кэше. Это означает, что изменения какого-либо свойства извлечённого 
объекта, повлечёт также и изменение объекта хранящего в кеше, так как это один и тот же объект. Также, стоит учесть 
thread safe, так как объекты в кэше не безопасны для изменения в разных потоках. В реальном приложении потребность 
изменения объектов хранящихся в кэше возникает не часто, но возникает. Одна из удобных функциональностей заключается
в извлечении копии объекта из кэша с которой можно выполнять произвольные действия.

##### Хранение и очистка регионов данных

Для приложения кэш представляет собой коллекцию пар ключ-значение. Это просто и элегантно, но с ростом приложения
становиться затруднительно подбирать правильные ключи. Например рассмотрим вариант кэширования двух связанных таблиц базы данных, 
например Vendors и Models. Скорее всего приложению потребуются все данные из таблицы Vendors и только связанные с определённой 
записью таблицы Vendors, записи таблицы Models. Организовать хранение таких срезов данных можно например по ключу 
`Vendors` для всех записей таблицы Vendors и ключам вида `Models.[VendorID]` для связанных записей таблицы Models.
При изменении записи таблицы Vendors может потребоваться очистить не только кэш таблицы Vendors, 
но и кэши связанных записей таблицы Models.

### Implementation

Описанные функциональности не критичны и могут без особых усилий быть реализованны прямо в приложении. За время 
разработки веб-приложений на платформе ASP.NET я собрал всё нужное в одной библиотеке взяв за основу реализации 
[шаблон Заместитель](https://en.wikipedia.org/wiki/Proxy_pattern). С выходом .NET Frameworк 4.0, 
в котором представлено прострарство имён `System.Runtime.Caching` и несколько классов с новой моделью кэширования, 
потребность в стандартном кэше ASP.NET и его улучшениях конечно же отпадает. Sad but true.

### Properties and settings

```csharp
/// <summary>
/// Признак использования кэша для хранения данных
/// </summary>
public static bool IsCacheEnable;

/// <summary>
/// Тип хранения объектов в кэше
/// </summary>
public static CacheExpirationType ExpirationType;

/// <summary>
/// Время хранения объектов в кэше
/// </summary>
public static TimeSpan ExpirationTime;
```

The interval between the time the inserted object was last accessed and the time at which that object expires.

### Methods to extract data from the cache

```csharp
/// <summary>
/// Возвращает типизированный объект из глобального кэша приложений по уникальному имени
/// </summary>
/// <typeparam name="T">Тип возвращаемого объекта</typeparam>
/// <param name="key">Имя объекта в кэше</param>
/// <param name="defaultValue">Значение, возвращаемое по умолчанию, если запрашиваемого объекта нет в кэше</param>
/// <returns>Извлеченный объект, или <paramref name="defaultValue"/> - если в кэше указанного объекта не обнаружено</returns>
public static T GetData<T>(string key, T defaultValue = default(T))

/// <summary>
/// Создаёт новый типизированный объект и заполняет его свойтсвами объекта из кэша. 
/// Идентичен методу <seealso cref="GetData{T}(string, T)"/>, но вместо возращение объекта из кэша 
/// дублирует его значения в новом объекте.
/// </summary>
/// <typeparam name="T">Тип возвращаемого объекта</typeparam>
/// <param name="key">Имя объекта в кэше</param>
/// <param name="defaultValue">Значение, возвращаемое по умолчанию, если запрашиваемого объекта нет в кэше</param>
/// <returns>Извлеченный объект, или <paramref name="defaultValue"/> - если в кэше указанного объекта не обнаружено</returns>
public static T GetDeepCopiedData<T>(string key, T defaultValue = default(T))
```

### Methods to store data in the cache

```csharp
/// <summary>
/// Добавляет объект в глобальный кэш приложений используя параметры кэширования заданные по умолчнию.
/// Если объект по такому имени уже существует, то он будет переписан новым значением.
/// </summary>
/// <param name="key">Имя добавляемого объекта в кэше</param>
/// <param name="value">Значение добавляемого объекта в кэше</param>
public static void InsertData(string key, object value)

/// <summary>
/// Добавляет объект в глобальный кэш приложений с абсолютным кэшированием на заданное время.
/// Если объект по такому имени уже существует, то он будет переписан новым значением.
/// </summary>
/// <param name="key">Имя добавляемого объекта в кэше</param>
/// <param name="value">Значение добавляемого объекта в кэше</param>
/// <param name="expirationTime">Время хранения объекта в кэше</param>
public static void InsertAbsoluteExpirationData(string key, object value, TimeSpan expirationTime)

/// <summary>
/// Добавляет объект в глобальный кэш приложений со скользящим кэшированием на заданное время.
/// Если объект по такому имени уже существует, то он будет переписан новым значением.
/// </summary>
/// <param name="key">Имя добавляемого объекта в кэше</param>
/// <param name="value">Значение добавляемого объекта в кэше</param>
/// <param name="expirationTime">Время хранения объекта в кэше</param>
public static void InsertSlidingExpirationData(string key, object value, TimeSpan expirationTime)

/// <summary>
/// Добавляет объект в глобальный кэш приложений с заданными параметрами кэширования.
/// Если объект по такому имени уже существует, то он будет переписан новым значением.
/// </summary>
/// <param name="key">Имя добавляемого объекта в кэше</param>
/// <param name="value">Значение добавляемого объекта в кэше</param>
/// <param name="expirationType">Тип хранения объекта в кэше</param>
public static void InsertExpirationData(string key, object value, CacheExpirationType expirationType)

/// <summary>
/// Добавляет объект в глобальный кэш приложений с заданными параметрами кэширования.
/// Если объект по такому имени уже существует, то он будет переписан новым значением.
/// </summary>
/// <param name="key">Имя добавляемого объекта в кэше</param>
/// <param name="value">Значение добавляемого объекта в кэше</param>
/// <param name="expirationType">Тип хранения объекта в кэше</param>
/// <param name="expirationTime">Время хранения объекта в кэше</param>
public static void InsertExpirationData(string key, object value, CacheExpirationType expirationType, TimeSpan expirationTime)
```

### Methods to remove data from the cache

```csharp
/// <summary>
/// Удаляет объект из глобального кэша приложений по уникальному имени
/// </summary>
/// <param name="key">Имя объекта в кэше</param>
public static void RemoveDataByKey(string key)

/// <summary>
/// Удаляет объекты из глобального кэша приложений по частичному совпадению имени.
/// Удаляются все объекты, имена которых начинаются с указанного значения.
/// </summary>
/// <param name="keyStartsWith">Частичное имя объекта в кэше</param>
public static void RemoveAllDataByKey(string keyStartsWith)

/// <summary>
/// Удаляет все объекты из глобального кэша приложений.
/// </summary>
public static void RemoveAllData()
```

### Usage example

##### 1. Settings the default parameters when application starts

```csharp
public class Global : System.Web.HttpApplication
{
    protected void Application_Start(object sender, EventArgs e)
    {
        Settings settings = Settings.Default;
        DataCache.IsCacheEnable = settings.IsCacheEnable;
        DataCache.ExpirationType = settings.ExpirationType;
        DataCache.ExpirationTime = settings.ExpirationTime;
    }
}
```

##### 2. Implement the standard template working with cache on the Page or in DAL

```csharp
public partial class DefaultPage : System.Web.UI.Page
{
    public const string STR_CACHENAME = "something";

    protected void Page_Load(object sender, EventArgs e)
    {
        if (!this.IsPostBack)
        {
            SomethingDataModel val = DataCache.GetData<SomethingDataModel>(STR_CACHENAME);
            if (val == null) 
            {
                SomethingDataModel val = new SomethingDataModel(); // get data from ...
                DataCache.InsertData(STR_CACHENAME, val);
            }
            DataBind(val); // use data on the page
        }
    }
}
```

Если требуется задать параметры хранения в кэше отличные от заданых при старте приложения, воспользуйтесь
одним из переопределённых методов `Insert{Wich}Data`.

##### 3. In the admin panel or when the database was updated remove also cached values by key or by key that starts with

```csharp
public partial class DefaultPage : System.Web.UI.Page
{
    public const string STR_CACHENAME = "something";

    protected void Page_Load(object sender, EventArgs e)
    {
        if (this.IsPostBack)
        {
            DAL.Update(); // update the database ...
            DataCache.RemoveAllDataByKey(STR_CACHENAME); // clear cache
            Response.Redirect("Default.aspx"); // proccessing the request
        }
    }
}
```

### Support or Contact

Having questions? [Contact me](https://github.com/CanadianBeaver) and I will help you sort it out.
 
<style> .inner { min-width: 800px !important; max-width: 60% !important; }</style>