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

Интерфейс - Класс, содержащий только чисто виртуальные методы и не имеющий реализации. Это просто протокол фзаимодействия между объектами.

Фабрика - Нечто, создающее объекты. В этой статье будем использовать одну глобальную функцию в качестве фабрики.

Граница DLL - Линия между кодом, инстанцированным в DLL и кодом в вызывающем процессе. В некоторых случаях код может быть по обе стороны границы, например встроенная функция в заголовочном файле, который используется в DLL и исполняемом файле. Функция фактически создается по обе стороны границы. Таким образом, если встроенная функция имеет статические переменные, будет создано две переменные: одна в приложении, одна в DLL и какая используется - зависит от того, кто вызывает эту функцию.

## Первая попытка
Предположим, вы хотите создать переносимый API и хотите реализовать его в DLL. Создадим класс Window, который может представлять собой окно в нескольких разных системах окон: Win32, MFC, wxWindows, Qt, Gtk, Aqua, X11, Swing (gasp) и т. д. Сделаем несколько попыток создать интерфейс до тех пор, пока он не будет работать в разных реализациях, компиляторах и настройках компилятора.

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
Я не собираюсь показывать реализацию, поскольку я предполагаю, что вы уже знаете, как это сделать. Существует одна вопиющая проблема с этим интерфейсом: предполагается, что вы используете базовый API Win32. То есть он содержит HWND как частный член, который вводит зависимость между нашим классом Window и Win32 SDK. Одним из возможных решений является использование идиомы pImpl для удаления частных членов класса из определения класса. Вы можете прочитать больше об этом в другом месте [1](http://www.gotw.ca/publications/mill04.htm), [2](http://www.gotw.ca/publications/mill05.htm), [3](http://www.gotw.ca/gotw/028.htm) и [4](http://c2.com/cgi/wiki?PimplIdiom). Кроме того, вы не можете добавлять новые члены в класс без нарушения бинарной совместимости, так как изменяется размер экземпляра класса.

Возможно, самая важная проблема с этим подходом заключается в том, что методы не являются виртуальными. Таким образом, они реализованы как специально названные функции, которые принимают указатель this в качестве первого аргумента. К сожалению, я не знаю каких-либо двух компиляторов, которые одинаково управляют именами методов. Поэтому не рассчитываете, что ваша DLL работает с исполняемым файлом, скомпилированным с другим компилятором!

## Попытка 2
Если вы имеете опыт в области объектно-ориентированного программирования, вы знаете, что каждый класс можно разбить на два понятия: интерфейс и фабрику. Фабрика - это механизм создания объектов, и интерфейс позволяет вам общаться с ними. Следующая версия Window.h отделяет эти понятия. Обратите внимание, что вам больше не нужно экспортировать класс (но вы должны экспортировать фабричную функцию!), Так как он абстрактен, все вызовы методов проходят через таблицу виртуальных функций объекта, а не через прямую ссылку на DLL. Только вызов функции Factory вызывает непосредственно в DLL.

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
Так гораздо лучше! Код, использующий объект окна, не заботится о том, к какому конкретному классу относится этот объект. 
Однако, все еще есть проблема:различные компиляторы по-разному работают с именами, поэтому функция CreateWindow в DLL, созданных разными компиляторами, будет иметь разные имена. То есть, если DLL собирается Visual C++ 6, вы не сможете ее использовать в Borland C++, и наоборот. К счастью, стандарт C++ позволяет отключить модифицирование имен, используя extern "C".

Вы могли заметить еще одну проблему в этом коде. Разные компиляторы по-разному реализуют стандартную библиотеку C++. Иногда пользователи сами подменяют стандартную библиотеку компилятора иной реализацией. Т.к. вы не можете положиться на бинарную совместимость STL объектов между компиляторами, вы не можете безопасно их использовать в изнтерфейсе DLL.

Если когда-нибудь будет создан C++ ABI для Windows, он должен конкретно описывать взаимодействие с каждым классом стандартной библиотеки. Но я не думаю, что это случится в  ближайшее время.

Последняя проблема здесь - незначительная. По соглашению, методы COM и функции DLL используют соглашение о вызовах __stdcall. Мы можем исправить это с помощью макроса CALL, который я определил выше. (Вы захотите переименовать его в своем проекте.)

## Ревизия 3
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

Мы почти у цели! Этот частный интерфейс скорее всего будет работать в большинстве ситуаций. Однако, виртуальный деструктор делает ситуацию немного интереснее. Т.к. COM не использует виртуальных деструкторов, вы не можете положиться на то, что разные компиляторы будут работать с ними одинаково. Однако, вы можете заменить виртуальный деструктор виртуальным методом, который выполнит **delete this**. Таким образом и конструктор, и деструктор будут находиться по одну сторону от границы DLL.Например, если вы попытаетесь использовать отладочную DLL-библиотеку VC ++ 6 вместе с релизным исполняемым файлом, либо ПО аварийно завершит работу, либо вы столкнетесь с предупреждениями типа «Значение ESP не сохранено при вызове функции». Эта ошибка возникает из-за того, что отладочная версия библиотеки времени выполнения VC ++ имеет другой аллокатор, чем релизная версия. Поскольку эти два аллокатора несовместимы, мы не можем выделить память на одной стороне границы DLL и удалить ее на другой.

«Но чем виртуальный деструктор отличается от другого виртуального метода?» спросите вы. Виртуальные деструкторы не несут ответственности за освобождение памяти, используемой объектом: они просто вызываются для выполнения необходимой очистки перед освобождением объекта. Исполняемый файл, который использует вашу DLL, попытается освободить память самого объекта. С другой стороны, метод destroy () отвечает за освобождение памяти, поэтому все вызовы new и delete остаются на одной стороне границы DLL.

Так же следует поместить деструктор интерфейса в секцию protected, чтобы было невозможно случайно использовать delete с интерфейсом.

## Ревизия 4
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
Т.к. в коде отсутствуют конструкции, которые не были бы определены в COM, код должен прекрасно работать в любой комбинации компиляторов и их настроек. К сожалению, не все так идеально. Вы должны помнить об удалении объекстов с использованием **object->destroy();**, что не так интуитивно как **delete object;**. Возможно, еще хуже, что вы не можете использовать **std::auto_ptr** с этими объектами. auto_ptr предпочитает использовать delete. Есть ли выход? Да! Сейчас будет немного странно... Вы можете перегрузить оператор delete для интерфейса, при этом вызывая destroy(). Т.к. оператор delete получает указатель на void, вы должны будете предположить, что никогда не вызываете Window :: operator delete для всего, что не является Window. Это довольно безопасное предположение. Вот реализация оператора:

```
  void operator delete(void* p) {
    if (p) {
      Window* w = static_cast<Window*>(p);
      w->destroy();
    }
   }
```
Выглядит отлично. Вам снова доступен auto_ptr и вы все еще имеете стабильный ABI. Когда вы снова пересоберете и протестируете ваш код (вы ведь  тестируете?), вы заметите переполнение стека в WindowImpl::destroy! Что происходит? Если вы помните, метод destroy() просто вызывает **delete this**. Поскольку интерфейс перегружает оператор delete, цепочка вызовов зацикливается. Решение этой частной проблемы состоит в перегрузке оператора delete в классе реализации для использования глобального оператора delete:

```
  void operator delete(void* p) {
    ::operator delete(p);
  }
```
## Последние штрихи
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
## Финальная версия
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
  
## Итоги
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
