[Оригинал](http://www.mingw.org/wiki/Interoperability_of_Libraries_Created_by_Different_Compiler_Brands)

# Взаимодействие библиотек, созданных разными компиляторами

## Введение
Объектные файлы и статические библиотеки, созданные различными компиляторами (или значительно различающимися релизами одного компилятора) обычно не могут быть скомпонованы друг с другом. Это касается не только MinGW: многие другие компиляторы взаимносовместимы. По возможности собирайте все из исходного кода одной версией одного компилятора.

Динамически библиотеки (DLL) имеют особенности. Иногда вы можете  скомпоновать DLL, собранную одним компилятором и приложение, собранное другим. Это возможно если библиотеки написаны на C, даже если приложение написано на C++. Например, программы, скомпилированные MinGW C++, обычно компонуются с C runtime library, предоставляемой Windows. Библиотеки DLL, написанные на C++, тоже работают, пока вы взаимодействуете с ними только через интерфейс C, описанный через extern "C". Иначе вы, возможно, получите ошибки компоновщика, так как различные компиляторы осуществляют [mangling](https://en.wikipedia.org/wiki/Name_mangling) (преобразование) имен C++ по-разному.

# Почему различные компиляторы могут быть несовместимы
Иногда люди удиивляются, почему компиляторы не используют одну и ту же схему для [name-mangling](https://en.wikipedia.org/wiki/Name_mangling). Возможно, программы компоновались бы успешно, но, скорее всего, выходили бы из строя при обращении к DLL. Для реальной совместимости требуется общий двоичный интерфейс приложения (ABI), а имена ссылок в бинарных файлах - всего лишь одна из частей ABI. Вот некоторые соображения на эту тему:
- http://ou800doc.caldera.com/SDK_porting/binary_cplusplus_compat.html
- One compiler offers 3200 different ABIs according to this page: http://www.boost.org/libs/config/config.htm#source
- Согласно Страуструпу(The Annotated C++ Reference Manual, 7.2.1c, page 122):
```
Если две реализации C++ для одной системы используют различные последовательноси вызовов (calling sequences) или несовместимы на уровне компоновки по другим причинам, было бы неблагоразумно использовать одинаковые кодировки сигнатур.
```
Разработчики намеренно использовали разные схемы name-mangling, полагая, что лучше просто сказать "нет" во время компоновки, чем обеспечить запуск кода и позволить возникать ошибкам во время исполнения (run time).

Хоть  GNU g++ и может компоноваться с MSVC C++ libraries сейчас и производить MSVC++ совместимые библиотеки, это не означает, что они будут способны работать во время исполнения по причине динамической природы C++. Некоторые возможные причины:
- Простая проблема с name mangling, которую можно обойти, задав явно .def файл.
- Различные вопросы выравнивания структур, требующие настроек компилятора.
- Фундаментальный конфликт моделей исключений и памяти:--
- new/delete или malloc/free в MSVC DLL не будут взаимодействовать  с Cygwin newlib new/delete или malloc/free. Невозможно освободить память, выделенную с использованием чужих new/malloc.
- Медленная модель исключенией GNU SJLJ (GCC-3.x and earlier), совместима с моделью исключений MSVC++, однако новая модель DWARF2 не будет совместима (используется начиная с GCC-4) - исключения, возникшие в MSVC DLL не будут пойманы приложением Cygwin и наоборот.

## Обход проблем
Статья [http://aegisknight.org/cppinterface.html](https://github.com/ilyayunkin/LibsInteroperability/blob/master/BinaryCompatibleCpp.md) Описывает продвинутую технику, которая может помочь вам обойти проблемы name-mangling и компоновки; Будьте внимательны, следуя руководству, если решите его испробовать.
