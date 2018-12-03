[Оригинал](http://www.mingw.org/wiki/Interoperability_of_Libraries_Created_by_Different_Compiler_Brands)

# Взаимодейсмтвие библиотек, созданных разными компиляторами

## Введение
Объектные файлы и статические библиотеки, созданные различными компиляторами (или даже значительно различающимися релизами одного компилятора) обычно не могут быть скомпанованы друг с другом. Эта ситуация не специфична для MinGW: многие другие компиляторы взаимносовместимы. По возможности собирайте все из исходного кода одной версией одного компилятора.

Динамические библиотеки(Dll's) немного отличаются. Иногда вы можете  скомпоновать DLL, собранную одним компилятором и приложение, собранное другим. Это работает прекрасно если библиотеки написаны на C, даже если приложение написано на C++. Например, MinGW C++ программы обычно компонуются с C runtime library, предоставляемой Windows. DLL, написанные на C++ так же работают, пока вы взаимодействуете с нимитолько через интерфейс C, описанный с extern "C". Иначе вы, возможно, получите ошибки компоновщика, так как различные компиляторы осуществляют mangling имен C++ по-разному.

# Почему различные компиляторы могут быть несовместимы
Иногда люди удиивляются, почему компиляторы не используют одну и ту же схему для name-mangling. Это может сделать соединение успешным, но скорее всего даст вам программу, которая выйдет из строя при вызове в DLL. Для реальной совместимости ссылок требуется общий двоичный интерфейс приложения, а определение имен - всего лишь одно соображение среди многих. Вот неполный список:
- http://ou800doc.caldera.com/SDK_porting/binary_cplusplus_compat.html
- One compiler offers 3200 different ABIs according to this page: http://www.boost.org/libs/config/config.htm#source
- Согласно Страуструпу(The Annotated C++ Reference Manual, 7.2.1c, page 122):
```
Если две реализации C++ для одной системы используют различные последовательноси вызовов (calling sequences) или несовместимы на уровне компоновки по другим причинам, было бы неблагоразумно использовать одинаковые кодировки сигнатур.
```
Разработчики традиционно использовали намеренно разные схемы смены имен, полагая, что лучше просто сказать "нет" во время компоновки, чем делать какую-то простую работу кода и позволять возникать проблемы во время исполнения (run time).

Даже хотя  GNU g++ может компоноваться с  MSVC C++ libraries сейчас и может производить MSVC++ совместимые библиотеки, это не означает, что они будут способны работать во время исполнения по причине динамической природы C++. Некоторые возможные причины:
- The simple name mangling issue which it may be possible to circumvent with an explicit .def file.
- Different structure alignment issues which need the correct compiler options (-mms-bitfields, ...).
- A fundamental conflict of underlying exception and memory models:--
- A new/delete or malloc/free in a MSVC DLL will not co-operate with a Cygwin newlib new/delete or malloc/free. One cannot free space which was allocated in a function using a different new/malloc at all.
- An exception raised by an MSVC DLL will not be caught by a Cygwin executable, and vice versa.
- The slow GNU SJLJ exception model, (used in GCC-3.x and earlier), is compatible with the MSVC++ model, but the new DWARF2 model, (which will be used by GCC-4.x), will be incompatible.

## Обобщим проблем
Статья http://aegisknight.org/cppinterface.html Описывает продвинутую технику, которая может помочь вам обойти проблемы name-mangling и компоновки; Будьте внимательны, следуя руководству, если решите его испробовать.
