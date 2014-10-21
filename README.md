Aspect Injector Reference
========================

### Features ###
- Compile-time injection
- No runtime dependencies
- Advice debugging
- Interface injection
- Injection into Methods, Properties, Events
- Ability to terminate method execution 
- Ability to catch exceptions (try-catch wrapper)
 
### Demo ##

This
```C#
namespace Test
{
    class LogMethodCallAspect
    {
        [Advice(InjectionPoints.Before, InjectionTargets.Method)]
        public void BeforeMethod()
        {
            Console.WriteLine("Method executing");
        }
    }

    class MyClass
    {
        [Aspect(typeof(LogMethodCallAspect))]
        public void Do()
        {
            Console.WriteLine("Here I am!");
        }
    }
}
```
will be translated to 
```C#
namespace Test
{
	class LogMethodCallAspect
	{
		public void BeforeMethod()
		{
			Console.WriteLine("Method executing");
		}
	}

	class MyClass
	{
		private readonly LogMethodCallAspect __a$_LogMethodCallAspect;
		public void Do()
		{
			this.__a$_LogMethodCallAspect.BeforeMethod();
			Console.WriteLine("Here I am!");
		}
		public MyClass()
		{
			this.__a$_LogMethodCallAspect = new LogMethodCallAspect();
		}
	}
}
```

### Concept ###

Aspect is a class which contains a set of advices - methods which should be injected to certain points in the code. Each advice has mandatory attributes which define a kind of target class members (constructor, getter, setter, regular method etc.) and join points - points in the code where this advice should be injected (before target member, after or both). Aspects and advices are marked with appropriate attributes. For example, we have a class with one method marked as advice:
```C#
class TraceAspect
{
	private int count;
	
	[Advice(InjectionPoints.Before, InjectionTargets.Method)]
	public void CallCountTrace()
	{
		Console.WriteLine("Call #{0}", count);
		count++;
	}
} 
```

Having it we can apply this aspect to any method or a set of methods of some class:
```C#
//Method CallCountTrace of TraceAspect instance will be called at the beginning of Calculate() 
[Aspect(typeof(TraceAspect))]
public void Calculate() { }

//Method CallCountTrace of TraceAspect instance will be called at the beginning of Load() and Save()
[Aspect(typeof(TraceAspect))]
class Container
{
    public string Name { get; set; }

    public void Load() { }
    public void Save() { }
}

//Will not work - CallCountTrace() advice is applicable to regular methods only
[Aspect(typeof(TraceAspect))]
public string Name { get; set; }
```
Please note that there will be only one instace of an aspect per target class regardless of number of affected members. So in the example above Container class will have only one instance of TraceAspect, so both Load() and Save() will increment the same call counter.


### Attributes ###

**AspectAttribute**

Indicates that the aspect of the specified type should be applied to a specific class member or every class member matching the specified filter.
```C#
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class | AttributeTargets.Property | AttributeTargets.Event, AllowMultiple = true)]
public sealed class AspectAttribute : Attribute
```
Parameters

|Name |Type |Description  |
|-----|-----|-------------|
|Type |Type |Specifies the class of the aspect which should be applied to the target member | 
|CustomData |object[] |Any custom data which then can be used by the aspect's implementation
|NameFilter |string | Specifies a string which is used to filter target class members: all members containing the specified string will be processed by this aspect.
|AccessModifierFilter| AccessModifiers | Allows to filter target class members by their access modifiers: all members having one of the specified modifiers will be processed by this aspect. This attribute can work together with NameFilter. In case both NameFilter and AccessModifierFilter are set, only class members matching both filter conditions are chosen for processing. 

**AspectFactoryAttribute**

TBD
```C#
[AttributeUsage(AttributeTargets.Method)]
public sealed class AspectFactoryAttribute : Attribute
```
**AdviceAttribute**

Marks methods of an aspect class which should be injected to the target classes according to the matching rules. Specifying it on class level is equal to marking all public class methods with this attribute.   
```C#
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false)]
public sealed class AdviceAttribute : Attribute
```
Parameters

|Name |Type |Description  |
|-----|-----|-------------|
|Points|InjectionPoints|Point in the target class member, where the current advice should be injected.|
|Targets|InjectionTargets|Kind of target class members to which the current advice should be injected.|

**AdviceArgumentAttribute**

Is used to tell the injector which data should be passed to advice method parameters. Every parameter of advice method should have this attibute set.
```C#
[AttributeUsage(AttributeTargets.Parameter, AllowMultiple = false)]
public sealed class AdviceArgumentAttribute : Attribute
```
Parameters

|Name |Type |Description  |
|-----|-----|-------------|
|Source|AdviceArgumentSource| A kind of a source from which specific advice parameter should be populated |

**AdviceInterfaceProxyAttribute**

Specifies an interface which will be automatically implemented by any target class associated with the aspect. All calls to interface methods on a target class will be redirected to the corresponding aspect instance, so any aspect class having this attribute must implement the specified interface explicitly.   
```C#
[AttributeUsage(AttributeTargets.Class)]
public sealed class AdviceInterfaceProxyAttribute : Attribute
```
Parameters

|Name |Type |Description  |
|-----|-----|-------------|
|Interface|Type| Interface which should be implemented by any target class |

The following example shows how to create an aspect which will automatically implement INotifyPropertyChanged on all target classes and inject raising PropertyChanged event to all property setters:
```C#
[AdviceInterfaceProxy(typeof(INotifyPropertyChanged))]
public class NotifyPropertyChangedAspect : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;

    [Advice(InjectionPoints.After, InjectionTargets.Setter)]
    public void RaisePropertyChanged(
        [AdviceArgument(AdviceArgumentSource.Instance)] object targetInstance,
        [AdviceArgument(AdviceArgumentSource.TargetName)] string propertyName)
    {
        var handler = PropertyChanged;
        if(handler != null)
        {
            handler(targetInstance, new PropertyChangedEventArgs(propertyName));
        }
    }
}
```
### Enumerations ###
<br/>
**AccessModifiers**

Is used to specify target members filtering criteria for AspectAttribute, the meaning of the values correspond to the member access modifiers in C#.

|Name |
|:----|
|Private|
|Protected|
|Internal|
|ProtectedInternal|
|Public|

**AdviceArgumentSource**

Is used to specify the source from which specific advice parameter should be populated. Corresponding required parameter types are shown in the table below.

|Name|Parameter type|Description|
|:---|:-------------|:----------|
|Instance| object | Target class instance.
|TargetName| string | The name of the target member
|TargetArguments| object[] | The array of target member arguments.
|TargetReturnValue| object | The return value of the target member.
|TargetException| Exception | An exception occurred in the target.
|AbortFlag| ref bool | A flag through which an advice can abort execution of the current target. Is appliable only for "Before" injection point and non-constructor targets.
|CustomData| object | The data specified in CustomData property of the corresponding AspectAttribute

**InjectionPoints**

Is used to specify the point in the target class member, where the current advice should be injected.

|Name|Description
|:---|:----------
|Before| Before the target body is executed.
|After| After the target body is executed.
|Exception| On any exception occured inside the target method.


**InjectionTargets**

Is used to specify the kind of target class members to which the current advice should be injected.

|Name|
|:---|
|Constructor|
|Method|
|Getter|
|Setter|
|EventAdd|
|EventRemove|