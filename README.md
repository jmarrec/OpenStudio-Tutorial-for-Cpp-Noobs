# OpenStudio coding lesson for C++ noobs

You've decided to try to add or modify the source code of OpenStudio, but you have little to none experience in C++?
Well, first, you're awesome, thanks for caring enough and contributing to this open source project.
Second, these littles notes I wrote during my learning journey should help you clarify a couple of fundamental things. It shouldn't be a complete substitute for a proper tutorial though.

## File types

In `OS-Core\src\model` you're seeing three file types for each class:

* Class.cpp
* Class.hpp
* Class_Impl.hpp

For the first two, it's a quick search to see that .hpp means header file, and .cpp is a C++ file. The main purpose is separating the interface from the implementation. The header declares "what" a class (or whatever is being implemented) will do, while the cpp file defines "how" it will perform those features. 

You **declare** - or *prototype* - methods in the header, and you **implement** them in the cpp file.

This reduces dependencies so that code that uses the header doesn't necessarily need to know all the details of the implementation and any other classes/headers needed only for that. This will reduce compilation times and also the amount of recompilation needed when something in the implementation changes.

In the header files you typically 'prototype' the class definition so that all of your class members can be used, not just ones defined before the current call.

### PIMPL

What's the _Impl.hpp then? Well, that's called "Pointer to Implementation Idiom" or **PIMPL**.

That's getting a little bit complicated, but the idea is to have the Implementation functions (the ones that actually do the work) declared in the `Class_Impl.hpp`. The `Class.hpp` will declare the "public" functions that will typically just be forwarding to the Class_Impl functions (Class_Impl might have a bunch more function than Class, not all Class_Impl functions are meant to be available to the public). Both are implemented in the .cpp.

You're not seeing it are you? Let's see a practical example (amputed for clarify, this won't run as is...)

```c++
// =========== MyClass_Impl.hpp ============
class MyClass_Impl {
  public:
    void bar();
    void foo(bool b);
};
// =========== MyClass_Impl.hpp ============



// ============= MyClass.hpp ===============

class MyClass {
  public:
    void bar();
};

// ============= MyClass.hpp ===============



// ============= MyClass.cpp ===============

// Implementation functions: does stuff
void MyClass_Impl::foo(bool b) {
  if (b) {
    cout << "Babar!" << endl;
  } else {
    cout << "I'm going to murder you!!!" << endl;
  }
}
void MyClass_Impl::bar() {
  foo(true);
}

// Detail/Public: this function will be available in the API, and just forwards to the Implementation function
void MyClass::bar() {
  // MyClass:bar just "forwards" to MyClass_Impl::bar()
  getImpl<detail::MyClass_Impl>()->bar();
}
```

You're still not seeing why that's a good thing to have two classes leading to three files and a much more verbose code base? That's also, ok, neither am I really.

A couple of things are worth noting though:

* Only the methods of Class.hpp will be available to the API, meaning the bindings (ruby, python, C#, java)
* This allows you to add/remove methods to the pimpl class, without modifying the external class's header file.
* It separates the implementation from the interface. This is usually not very important in small size projects. But, in large projects and libraries, it can be used to reduce the build times significantly.

### Reading materials

* [Why have header files and .cpp files in C++?](http://stackoverflow.com/questions/333889/why-have-header-files-and-cpp-files-in-c)
* [Why should the PIMPL idipm be used?](http://stackoverflow.com/questions/60570/why-should-the-pimpl-idiom-be-used)


## Put some sugar in your syntax

Here are a couple of things you'll see over and over in the files and that you won't necessarilly understand if you have no experience in C++ syntax

### Casing, Getters and Setters

OpenStudio follows the [lowerCamelCase](https://fr.wikipedia.org/wiki/CamelCase) notation for methods.

OpenStudio offers getters and setters:

* The getter function, eg myClass.value(), will just fetch the 'value'
* The setter function, eg `myClass.setValue()` or `myClass.resetValue()`, will modify the 'value'

### const

The `const` keyword, for 'constant' can be used for three different things:

1. Declare a constant variable

    const int x = 5;
    x = 10; // This will fail because x is not mutable!

2. Pass a const argument to a function

    void myfunc(const char x)
    {
      char y = x;  // OK
      x = y;       // Failure: The x argument cannot be modified within the function scope
    }
    
3. Inside a class, prevents the modification of the class member

    class MyClass {
        double x;
    }
    int MyClass::myfunc() const
    {
      x = 10; // Fails: You cannot modify the member of MyClass inside this function
    }
    
Basically: all OpenStudio getters are followed by const, because they get value and you want to make sure you absolutely forbid modification inside. In general, arguments to setters are passed with the keyword const (see 2.).

Example:

    // Example of a getter, we want to ensure we don't modify the class
    boost::optional<ThermalZone> ElectricLoadCenterStorageSimple_Impl::thermalZone() const {
      return getObject<ModelObject>().getModelObjectTarget<ThermalZone>(OS_ElectricLoadCenter_Storage_SimpleFields::ZoneName);
    }
    
    // Example of a setter, we explicitly forbid our specific ThermalZone 'thermalZone' to be modified
    bool ElectricLoadCenterStorageSimple_Impl::setThermalZone(const ThermalZone& thermalZone) {
      return setPointer(OS_ElectricLoadCenter_Storage_SimpleFields::ZoneName, thermalZone.handle());
    }

### By Val or By Ref, that is the question

You might have noticed in the last example of a setter the ampersand sign `&`. This brings us to the difference of passing by value or by reference.

The default is to pass by value. adding `&` to the type passes by Reference. Let's see a simple example to illustrate.

```c++
#include <iostream>

using namespace std;

double passByValue(double a) {
  a = 10;
  return a;
}
double passByRef(double& a){
  a = 10;
  return a;
}

int main(){

  double x = 0;

  passByValue(x);
  cout << "With ByVal: " << x << endl;
  // "With ByVal: 0": x has NOT been modified because we passed by val

  passByRef(x);
  cout << "With ByRef: " << x << endl;
  // "With ByRef: 10": x has been modified, because we passed by reference with the '&'
  
  return 0;
}
```

### virtual and override

These 'keywords' are used when a class (MyClass) inherits from a base class (BaseClass): the methods of BaseClass are available to MyClass.

I'll never explain virtual as well as this massively upvoted answer: [Why do we need virtual functions in C++](http://stackoverflow.com/questions/2391679/why-do-we-need-virtual-functions-in-c). So go ahead, read that and come back.

As far as override, apparently it's not a keyword, it's a special kind of specifier (just like const). Clearly we don't care about whether it's a specified or a keyword... But what is does is two things:

* It shows the reader of the code that "this is a virtual method, that is overriding a virtual method of the base class."
* it tells the compiler to make sure that you ARE indeed overwritting a virtual function of a parent class. The compiler will throw an error if you are in fact not overwritting a parent method.


### Reading materials

* [Is the 'override' keyword just a check for a overriden virtual method?](http://stackoverflow.com/questions/13880205/is-the-override-keyword-just-a-check-for-a-overriden-virtual-method)
* [What is the 'override' keyword in C++ used for?](http://stackoverflow.com/questions/18198314/what-is-the-override-keyword-in-c-used-for)
* A website that translate some (gibberish) C++ declarations in english: http://www.cdecl.org/


## Some common errors

There are two types of errors:

* When the build fails: usually you'll get an error message pointing you to the line where it failed, and you should be able to quickly find the mistakes (most often than not, you're missing a `;` or you have made a typo)
* Then there are other errors, where it builds, but it's still not doing what you want

Build error usually a because of missing semi-colons, typos, bad return types of functions between definition (in header) and implementation, and generally anything you might have messed up by copy pasting (you do that a lot, like I said it's verbose).

Here's a running list of some I have encountered and spent too much time finding the root cause for:

* Unable to set a pointer to another object: Check the .idd, make sure it's right. Common example: in EnergyPlus they use `\object-list ZoneNames` whereas in OpenStudio it's `\object-list ThermalZoneNames`... That one got me REAL bad.
* Not seeing some methods or something? Make sure you have ALL the includes you need, including MODELOBJECT_TEMPLATES and SWIG_TEMPLATE (in the *.i files)
    * Not seeing your new object in the bindings? You probably forgot the template thing for your class
    * Not seeing only a couple methods? If you have a base class, you probably forgot the template for the base class
* Usually I just find an object that's similar to the one I'm adding, and I do a search in src/model for this object name, and I go file by file to make sure I have the same include (CMakeList, *.i file, ConcreteModelObjects.cpp, etc). Also refer to the "Adding Objects to openstudio.docx" and make sure you haven't skipped a step. 
* Forward translation or Reverse translation completes but you have misplaced objects or missing ones? Typically that's because you have made a copy paste and forgot to make the necessary field changes. See how easy it is the make such a mistake:

```c++
//MinimumHeatRecoveryWaterFlowRate
{
  auto value = generatorMCHPHX.minimumHeatRecoveryWaterFlowRate();
  idfObject.setDouble(Generator_MicroTurbineFields::MinimumHeatRecoveryWaterFlowRate,value);
}

//MaximumHeatRecoveryWaterFlowRate
{
  auto value = generatorMCHPHX.maximumHeatRecoveryWaterFlowRate();
  // Here the field is wrong! So the maximumHeatRecoveryWaterFlowRate will overwrite the minimumHeatRecoveryWaterFlowRate
  idfObject.setDouble(Generator_MicroTurbineFields::MinimumHeatRecoveryWaterFlowRate,value);
}
```

