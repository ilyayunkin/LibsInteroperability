[Оригинал](https://chadaustin.me/cppinterface.html)
# Binary-compatible C++ Interfaces
Chad Austin, 2002.02.15
Updates
2003.02.21
Уточнил некоторые из моих комментариев благодаря отзывам и предложениям от Развана Сурдулеску. Для фактической совместимости с COM я добавил __stdcall в декларации метода. (По-видимому, если следовать правилам этой статьи, легко добавить привязки из вашей DLL в Delphi и VB!).

2002.04.03
Кто-то (к сожалению, я потерял свой адрес электронной почты и имя, прежде чем я смог его записать. Если вы читаете это или знаете, кто его отправил, напишите мне письмо) прислал мне еще одну рекомендацию. Суть его в том, что вы не должны использовать перегруженные методы в своих интерфейсах. Различные компиляторы будут упорядочивать их в vtable по-разному.

2002.03.27
Бен Скотт (bscott at iastate dot edu) представил отличные классы, которые упрощают использование концепций, представленных в этой статье. Просто унаследуйте классы интерфейса от DLLInterface и ваши реализации от DLLImpl! Загрузите исходный файл [здесь](https://chadaustin.me/dllinterface.h).

## Обзор
Эта статья объясняет, как создать C++ API для DLL, который будет работать между различными компиляторами и их конфигурациями (Release, Debug, ...).

## Бэкграунд
Многие платформы имеют ABI для выбранного языка программирования. Наприме, основным языком программирования BeOS является C++, поэтому C++ компилятор должен быть способен генерировать код с сохранением бинарной совместимости с C++ системными вызовами операционной системы (и классами и т.п.).

Для Windows API и ABI определены на языке C, поэтому разработчики C++  компиляторов вольны рразрабатывать C++ ABI на свое усмотрение.
Однако, в Microsoft создали объектно-ориентированный ABI для Windows, названный COM. Для упрощения использования COM, они сделали таблицы виртуальных функций (vtables) своих ABI такими, как требует COM.  Поскольку компиляторов для Windows, которые не могут работать с COM, очень мало, другие производители компиляторов обеспечили совместимость с COM vtables.

В ABI есть несколько аспектов. В этой статье обсуждаются только проблемы с использованием C ++ в Windows. Другие платформы имеют другие требования. (К счастью, поскольку большинство других платформ не так популярны, как Windows, для них имеется только один или два компилятора, и, следовательно, проблема не велика.)

## Понятия
ABI - Application Binary Interface. Бинарный интерфейс между системами. Если бинарный интерфейс меняется, обе стороны интерфейса (пользователь и реализация) должны быть перекомпилированы.

API - Application Program Interface. Интерфейс между системами, описываемый в исходном коде. Если интерфейс меняется, код, использующий его, должен быть модифицирован. Изменения API обычно влекут изменения ABI.

Интерфейс - Класс, содержащий только чистовиртуальные методы и не имеющий реализации. Это просто протокол фзаимодействия между объектами.

Factory - Something that creates objects. In this article, we'll use a single global function as our factory.
DLL Boundary - The line between code instantiated in a DLL and code in a calling process is called the DLL boundary. In some cases, code can be on both sides of the boundary: Consider an inline function in a header file that gets used in the DLL and the executable. The function is actually instantiated on both sides of the boundary. Therefore, if the inline function has a static variable, two variables will be created, one in the executable and one in the DLL, and which is used depends on whether the code in the DLL or the executable is calling the function.

## Первая попытка
Let's say you want to create a portable windowing API and you want to stick the implementation in a DLL. I'm going to create a class called Window which can represent a window in several different windowing systems: Win32, MFC, wxWindows, Qt, Gtk, Aqua, X11, Swing (*gasp*), etc... We'll walk through several attempts at creating an interface until it works across different implementations, compilers, and compiler settings.
```
// Window.h

#include <string>

#ifdef WIN32
  #ifdef EXPORTING
    #define DLLIMPORT __declspec(dllexport)
  #else
    #define DLLIMPORT __declspec(dllimport)
  #endif
  #define CALL __stdcall
#else
  #define DLLIMPORT
  #define CALL
#endif

class DLLIMPORT Window {
public:
  Window(std::string title);
  ~Window();

  void setTitle(std::string title);
  std::string getTitle();

  // ...

private:
  HWND m_window;
};
```
I'm not going to show the implementation, as I'm assuming you already know how to do that. There is one glaring problem with this interface: It assumes you're using the basic Win32 API. That is, it holds an HWND as a private member, which introduces a dependency between our Window class and the Win32 SDK. One possible solution is to use the pImpl idiom to remove the class's private members from the class definition. You can read more about that elsewhere [1], [2], [3], and [4]. Also, you cannot add new members to the class without breaking binary compatibility, as the size of the class changes.

Perhaps the most important problem with this approach is that the methods are non-virtual. Thus, they are implemented as specially named functions that take the 'this' pointer as their first argument. Unfortunately, I don't know of any two compilers that mangle method names in the same way. So don't think your DLL work with an executable compiled with another compiler!

Attempt #2
For those of you experienced in object oriented programming, you know that every class can be broken into two concepts: an interface and a factory. A factory is a mechanism for creating objects, and an interface allows you to communicate with them. The next version of Window.h will separate these concepts. Notice that you no longer need to export the class (you have to export the factory function though!), as it is abstract: all method calls go through the object's vtable, not through a direct linking to the DLL. Only the call to the factory function calls directly into the DLL.
```
// Window.h

#include <string>

class Window {
public:
  virtual ~Window() { }
  virtual void setTitle(std::string title) = 0;
  virtual std::string getTitle() = 0;
};
Window* DLLIMPORT CreateWindow(std::string title);
```
This is much better. The code that uses window objects doesn't care what actual type the window object is, just that it implements the Window interface. However, there is still a problem: Different compilers mangle symbol names differently, so the CreateWindow function in DLLs generated by different compilers will have a different names. This means that if you compile the windowing DLL with Visual C++ 6, you won't be able to use it in Borland C++, and vice versa. Fortunately, the C++ standard lets us disable symbol mangling on specified names, via extern "C".

Some of you may have noticed another problem with this code. Different compilers implement the standard C++ library differently. In the less obvious case, some people replace their compiler's implementation of the library with another (such as STLPort). Since you can't depend on STL objects being binary compatible across compilers, you cannot safely use them in your DLL interfaces.

If a C++ ABI is ever created for Windows, it will need to specify exactly how to interface with every class in the standard library, but I don't see this happening anytime soon.

The final problem here is a minor one. By convention, COM methods and DLL functions use the __stdcall calling convention. We can fix this with the CALL macro I defined above. (You'll want to rename it in your project.)

Revision 3
```
// Window.h

class Window {
public:
  virtual ~Window() { }
  virtual void CALL setTitle(const char* title) = 0;
  virtual const char* CALL getTitle() = 0;
};

extern "C" Window* CALL CreateWindow(const char* title);
```
We're almost there! This particular interface will probably work in a lot of situations. However, the virtual destructor makes things a little interesting... Since COM doesn't use virtual destructors, you can't depend on different compilers to use them identically. However, you can replace the virtual destructor with a virtual method which, in the implementation class, is implemented by delete this; This way, both construction and destruction are implemented on the same side of the DLL boundary. For example, if you try to use a VC++ 6 debug DLL alongside a release executable, you'll either crash or run into warnings like "Value of ESP not saved across function call". This error occurs because the debug version of the VC++ runtime library has a different allocator than the release version. Since the two allocators are not compatible, we cannot allocate memory on one side of the DLL boundary and delete it on the other.

"But how is a virtual destructor different from another virtual method?" Virtual destructors are not responsible for deallocating the memory used by the object: They are simply called to perform necessary cleanup before the object is deallocated. The executable that uses your DLL will try to free the object's memory itself. On the other hand, the destroy() method is responsible for deallocating memory, so all new and delete calls stay on the same side of the DLL boundary.

It's also a good idea to make the interface's destructor protected so that users of the interface can't inadvertently use delete on it.

Revision 4
```
// Window.h

class Window {
protected:
  ~Window() { }  // use destroy()

public:
  virtual void CALL destroy() = 0;
  virtual void CALL setTitle(const char* title) = 0;
  virtual const char* CALL getTitle() = 0;
};

extern "C" Window* CreateWindow(const char* title);
```
Since this code doesn't use any semantics not defined by COM, it should work flawlessly across compilers and configuration settings. Unfortunately, it's not ideal. You have to remember to delete objects with object->destroy();, which isn't nearly as intuitive as delete object;. Perhaps more importantly, you can no longer use std::auto_ptr on objects of this type. auto_ptr wants to delete the object it owns with delete object;. Is there a way to make the syntax delete object; actually call object->destroy();? Yes. Here's where things get a little weird... You can overload operator delete for the interface and have it call destroy(). Since operator delete takes a void pointer, you'll have to assume you never call Window::operator delete on anything that isn't a Window. This is a pretty safe assumption. Here's the operator implementation:

```
  void operator delete(void* p) {
    if (p) {
      Window* w = static_cast<Window*>(p);
      w->destroy();
    }
   }
```
Looks pretty good... You can now use auto_ptr again, and you still have a stable binary interface. When you recompile and test your new code (you are testing, right??), you'll notice that there is a stack overflow in WindowImpl::destroy! What's going on? If you remember how the destroy method is implemented, you'll see that it simply executes delete this;. Since the interface overloads operator delete, WindowImpl::destroy calls Window::operator delete which calls WindowImpl::destroy... ad infinitum. The solution to this particular problem is to overload operator delete in the implementation class to call the global operator delete:

```
  void operator delete(void* p) {
    ::operator delete(p);
  }
```
Finishing Touches
If your system has a lot of interfaces and implementations, you'll find that you'll want some way to automate undefining operator delete. Fortunately, this is possible too. Simply create a templated class called DefaultDelete and instead of deriving your implementation class from interface I, derive from class DefaultDelete<I>. Here's DefaultDelete's definition:
```
template<typename T>
class DefaultDelete : public T {
public:
  void operator delete(void* p) {
    ::operator delete(p);
  }
};
  ```
Final Implementation
Here is the final version of the code.
```
// Window.h

class Window {
public:
  virtual void CALL destroy() = 0;
  virtual void CALL setTitle(const char* title) = 0;
  virtual const char* CALL getTitle() = 0;

  void operator delete(void* p) {
    if (p) {
      Window* w = static_cast<Window*>(p);
      w->destroy();
    }
  }
};

extern "C" Window* CALL CreateWindow(const char* title);
// Window.cpp
#include <string>
#include <windows.h>
#include "DefaultDelete.h"

class WindowImpl : public DefaultDelete<Window> {
public:
  WindowImpl(HWND window) {
    m_window = window;
  }

  ~WindowImpl() {
    DestroyWindow(m_window);
  }

  void CALL destroy() {
    delete this;
  }

  void CALL setTitle(const char* title) {
    SetWindowText(m_window, title);
  }

  const char* CALL getTitle() {
    char title[512];
    GetWindowText(m_window, title, 512);
    m_title = title;  // save the title past the call
    return m_title.c_str();
  }

private:
  HWND window;
  std::string m_title;
};

Window* CALL CreateWindow(const char* title) {
  // create the Win32 window object
  HWND window = ::CreateWindow(..., title, ...);
  return (window ? new WindowImpl(window) : 0);
}
// DefaultDelete.h

template<typename T>
class DefaultDelete : public T {
public:
  void operator delete(void* p) {
    ::operator delete(p);
  }
};
```
  
Summary
That's about it. In closure, I'll enumerate guidelines to keep in mind when creating C++ interface. You can look back on this as a reference or use it to help solidify your knowledge.

All interface classes should be completely abstract. Every method should be pure virtual. (Or inline... you could safely write inline convenience methods that call other methods.)
All global functions should be extern "C" to prevent incompatible name mangling. Also, exported functions and methods should use the __stdcall calling convention, as DLL functions and COM traditionally use that calling convention. This way, if a user of the library is compiling with __cdecl by default, the calls into the DLL will still use the correct convention.
Don't use the standard C++ library.
Don't use exception handling.
Don't use virtual destructors. Instead, create a destroy() method and an overloaded operator delete that calls destroy().
Don't allocate memory on one side of the DLL boundary and free it on the other. Different DLLs and executables can be built with different heaps, and using different heaps to allocate and free chunks of memory is a sure recipe for a crash. For example, don't inline your memory allocation functions so that they could be built differently in the executable and DLL.
Don't use overloaded methods in your interface. Different compilers order them within the vtable differently.
References
STLPort is an alternate implementation of the STL.
SGI has another standard C++ library implementation.
The Corona image I/O library uses the techniques introduced in this article.
