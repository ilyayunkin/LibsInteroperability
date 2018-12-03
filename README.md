[Оригинал](http://www.mingw.org/wiki/Interoperability_of_Libraries_Created_by_Different_Compiler_Brands)

# Взаимодейсмтвие библиотек, созданных разными компиляторами

## Введение
Объектные файлы и статические библиотеки, созданные различными компиляторами (или даже значительно различающимися релизами одного компилятора) обычно не могут быть скомпанованы друг с другом. Эта ситуация не специфична для MinGW: многие другие компиляторы взаимносовместимы. По возможности собирайте все из исходного кода одной версией одного компилятора.

Динамические библиотеки(Dll's) немного отличаются. Иногда вы можете  скомпоновать DLL, собранную одним компилятором и приложение, собранное другим. Это работает прекрасно если библиотеки написаны на C, даже если приложение написано на C++. Например, MinGW C++ программы обычно компонуются с C runtime library, предоставляемой Windows. DLL, написанные на C++ так же работают, пока вы взаимодействуете с нимитолько через интерфейс C, описанный с extern "C". Иначе вы, возможно, получите ошибки компоновщика, так как различные компиляторы осуществляют mangling имен C++ по-разному.

# Почему различные компиляторы могут быть несовместимы
Иногда люди удиивляются, почему компиляторы не используют одну и ту же схему для name-mangling. Это может сделать соединение успешным, но скорее всего даст вам программу, которая выйдет из строя при вызове в DLL. Для реальной совместимости ссылок требуется общий двоичный интерфейс приложения, а определение имен - всего лишь одно соображение среди многих. Вот неполный список:
- http://ou800doc.caldera.com/SDK_porting/binary_cplusplus_compat.html
- One compiler offers 3200 different ABIs according to this page: http://www.boost.org/libs/config/config.htm#source
- According to Stroustrup (ARM, 7.2.1c, page 122):
```
If two C++ implementations for the same system use different calling sequences,
or in other ways are not link compatible, it would be unwise to use identical encodings
of type signatures.
```
Implementors have traditionally used deliberately different name-mangling schemes, figuring it's better to 'just say no' at link time than to make some simple code work and let the issues emerge at run time.

Even though GNU g++ can link MSVC C++ libraries now, and can produce MSVC++ compatible libraries/DLLs, this does not mean that they will be able to work at run-time due to the dynamic nature of C++. Some possible reasons for this are:--

The simple name mangling issue which it may be possible to circumvent with an explicit .def file.
Different structure alignment issues which need the correct compiler options (-mms-bitfields, ...).
A fundamental conflict of underlying exception and memory models:--
A new/delete or malloc/free in a MSVC DLL will not co-operate with a Cygwin newlib new/delete or malloc/free. One cannot free space which was allocated in a function using a different new/malloc at all.
An exception raised by an MSVC DLL will not be caught by a Cygwin executable, and vice versa.
The slow GNU SJLJ exception model, (used in GCC-3.x and earlier), is compatible with the MSVC++ model, but the new DWARF2 model, (which will be used by GCC-4.x), will be incompatible.
Circumventing the Issues
The article at http://aegisknight.org/cppinterface.html describes an advanced technique which may help you to work around the name-mangling and linking problems; be sure to follow its guidelines carefully, if you try it.
