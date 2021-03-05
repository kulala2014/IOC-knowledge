# IOC-knowledge

控制反转和依赖注入
1.控制反转
在传统的程序设计中，当某一个类需要另外一个类的协助时，通常由调用者来创建协助者的实例。
{
  public class ClassA
  {
    public void DoWork()
    {
      var b = new ClassB();
      b.DoStuff();
    }
  }
  public class ClassB
  {
    public void DoStuff()
    {
      Console.Writeline("Help to do some work for other works");
    }
  }
}
带来的问题是，ClassA和ClassB实现了强耦合，ClassB实例构造函数的变动都会带来ClassA的改动。
所谓的控制反转(IoC)就是现在创建协助类ClassB实例的工作不再有ClassA来完成，而是由其他类或容器来完成。

2.依赖注入
创建协助类的工作通常由其他容器来完成，然后注入到调用者，这被称为依赖注入（Dependence Injection）

3.依赖注入的关键
  a.根据依赖倒置原则，我们应该依赖于抽象来编程而不是实现。
  b.在容器中注册依赖关系，由容器来负责创建依赖关系的实例。
  c.将抽象的类注入到需要它的构造函数中。
  
4.依赖注入实现上述例子
  {
    Public Interface IThing
    {
      public void DoStuff();
    }
    public class ClassA
    {
      private readonly IThing _dependencyClass;
      public void DoWork(IThing ClassB)
      {
        _dependencyClass = ClassB;
        _dependencyClass.DoStuff();
      }
    }
    public class ClassB: IThing
    {
      public void DoStuff()
      {
        Console.Writeline("Help to do some work for other works");
      }
    }
  }
  
  class Programm
  {
    static void Main(string[] args)
    {
      IThing thing = new ClassB();
      var classA = new ClassA(thing);
      classA.DoWork();
    }
  }
  
5..net core之前的版本中我们用到的依赖注入容器有NInject和AutoFac,通过配置由容器来负责创建和管理服务。

6..net core开始，引入了官方的依赖注入容器Microsoft Dependency Injection(DI)容器总的IServiceCollection.
通常，Microsoft DI容器需要在 Startup 类中配置，在这里，我们可以使用ConfigureServices方法像容器注册服务，在应用程序托管生命周期的早期，将调用ConfigureServices方法，它有一个参数
IServiceCollection，这个参数在初始化应用程序时传入。
public class Startup
{
  public void ConfigureService(IServiceCollection services)
  {
    services.AddSingleton<IThing, ClassB>();
  }
}
对于控制台应用程序我们可以使用Microfosft DependencyInjection.

我们开始注册我们的服务，但是我们需要一个IServiceCollection, 定义如下：
    //
    // 摘要:
    //     Specifies the contract for a collection of service descriptors.
    public interface IServiceCollection : IList<ServiceDescriptor>, ICollection<ServiceDescriptor>, IEnumerable<ServiceDescriptor>, IEnumerable
    {
  
    }
 IServiceCollection没有定义任何成员，而是从IList<ServiceDescriptor>, ICollection<ServiceDescriptor>, IEnumerable<ServiceDescriptor>, IEnumerable 派生。
  在微软的依赖注入扩展中有一个默认的实现：ServiceCollection:
  public class ServiceCollection:IServiceCollection, ICollection<ServiceDescriptor>, IEnumerable<ServiceDescriptor>, IEnumerable, IList<ServiceDescriptor>
  {
    private readonly List<ServiceDescriptor> _descriptors = new List<ServiceDescriptor>();
    public int Count => _descriptors.Count;
    public bool IsReadOnly => false;
    public ServiceDescriptor this[int index]
    {
      get {return _descriptors[index];}
      set {_descriptors[index] = value;}
    }
    ....
   }
  
  它有一个私有的List集合：_descriptors,里面是ServiceDescriptor。
  
  
  注册2个服务：
     var serviceCollection = new ServiceCollection();

    serviceCollection.AddSingleton<ClassA>();
    serviceCollection.AddSingleton<IThing, ClassB>();
    
   上面的AddSingleton方法不在IServiceCollection, 和ServiceCollection定义，他是IServiceCollection的扩展方法，定义在扩展类ServiceCollectionServiceExtensions中。
  
  7.服务生命周期
  在Microsoft依赖注入框架中，可以使用三种生命周期注册服务，分别是单例（Singleton），瞬时（Transient），作用域（Scoped）：
  Singleton服务的特点是我们不会创建多个服务实例，只会创建一个实例，保存在DI容器中知道程序退出
  Transient每次使用时，DI容器都时创建一个新的实例。
  Scoped在一个作用域内，会使用同一个实例，像EF Core的DBContext上下文就被注册为作用域服务， 可以理解为每一个request级别创建一次，同一个http       request会在一个scopedzong
  
  8.注册服务时会发生什么：
  ServiceCollection.AddSingleton<IThing, ClassB>();
  AddSingleton方法如下：
  public static IServiceCollection AddSingleton(this IServiceCollection services, Type serviceType, Type implementationType)
  {
    if (services == null)
    {
        throw new ArgumentNullException(nameof(services));
    }
    if (serviceType == null)
    {
        throw new ArgumentNullException(nameof(serviceType));
    }
    if (implementationType == null)
    {
        throw new ArgumentNullException(nameof(implementationType));
    }
    
    return Add(services, serviceType, implementationType, ServiceLifetime.Singleton);
  }
  
  AddSingleton function call a private function Add, and input a Enum ServiceLifetime.Singleton.
  
  private static IServiceCollection Add(IServiceCollection collection, Type serviceType, Type implementationType, ServiceLifetime lifetime)
  {
    var descriptor new ServiceDescriptor(serviceType, implementType, lifetime);
    collection.Add(descriptor);
    return collection;
  }
  它创建一个新的ServiceDescriptor实例，传入服务类型，实现类型（可能与服务类型相同）和生命周期，然后调用Add方法添加到列表中。
  之前，我们了解到IServiceCollection本质上是包装了List <ServiceDescriptor>, ServiceDescriptor类很简单，代表一个注册的服务，包括其服务类型，实  现类型和生命周期。
  
  9.实例注册：
  我们也可以手动new一个实例，然后传入到AddSingleton（）方法中：

  var myInstance = new ClassB();
  serviceCollection.AddSingleton<IThing>(myInstance);
  
  们还可以手动定义一个ServiceDescriptor，然后直接添加到IServiceCollection中。

  var descriptor = new ServiceDescriptor(typeof(IThing), typeof(ClassB), ServiceLifetime.Singleton);
  serviceCollection.Add(descriptor);
  
  //To Been Continue.
  
  
  
  
  

