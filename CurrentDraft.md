# Comments on the Implementation #

Generally speaking, there are two requirements for a compiler of an interpreted language:

  1. the compiled code should run faster ; // compared to the interpreted version
  1. it should be 100% compatible with the interpreted version . // "interpreted" не повторять

// "Strictly speaking" поправить
Strictly speaking, making it run faster is not a problem. After all, we ( I mean -- the programmers) are writing compilers for 50 years now, and the main tricks for compiler optimization are well-known. So read the [Dragon Book](http://en.wikipedia.org/wiki/Dragon_Book) and go on )

// "The big problem" тоже можно было бы поправить
The big problem -- or, actually, a huge number of small problems -- is to be compatible. We don't really need a very efficient compiler for a slightly different language, if the other language differs from the one we are used to.
For example, currently there exist at least three compilers for a Python-alike language to C / C++ , all being severely incompatible with standard Python.

  1. Pyrex
  1. Cython
  1. Shed Skin

From the above, Shed Skin produces the fastest code, while two others are more commonly used (Pyrex -- for creation of the extension modules and calling C code, and Cython (which is in turn Pyrex-based) is extensively used in the [sagemath](http://en.wikipedia.org/wiki/Sage_%28mathematics_software%29) project).

For an illustration of the incompatibility problem, let me give a simple example of code that successfully runs by the interpreter, but won't be compiled correctly by any of the three compilers mentioned above:

```

```
if system == 'Windows':

    def have_windows():

        return True

else:

    def have_windows():

        return False

```
```

That is, the `have_windows()` function is created dynamically and will have different code -- or, in the terms of the Python interpreter, the '`have_windows`' variable will be assigned different code object values.

Of course I have no doubt that this example code could be compiled with any of the mentioned compilers after some refactoring. But the real problem is that there are _a lot_ of constructs that could employ the dynamic nature of the language. The conclusion is that **the translated program must entirely support the Python run-time environment.**

I am lazy enough to reproduce Python internals myself. Luckily, I don't need to: take the interpreter's run-time, bind it to the generated C code .. and watch the translated program now calls another Python module, who never thought or intended to compile to C. So I take all CPython internals as the run-time .. and realize that all I do is make another Python module that compiles other modules to binary executable code ( via C ) .


**Note one. Why C?**

In general, a program directly translated to binary executable code could be always made to be faster -- and is almost guaranteed to be smaller -- than the same program, compiled from an intermediate C representation. There's nothing to discuss here, except for two problems: first, there are many different architectures and processor models, not all of them being PC-compatible ; second, if all the subroutines are written in C -- should we bother then trying to call them from the binary code ?

История Psyco - вам например. Отличное решение, близкая к совершенству работа.  Вот там - да, там генерится двоичный код напрямую. Но. Проблему с кроссплатформенностью Psyco поимел. Вернее... (Гусары, молчать!). И двоичный код там естественен, так как в Psyco на ассемблере переписана часть библиотеки поддержки и интерпретатора. Но сейчас автор Psyco не поддерживает его для новых версий CPython, а занимается JT компилятором в PyPy. Уже несколько лет занимается...



То бишь о чем я... Да, о проблемах трансляции Питон-программ.
**Note two. Why byte-code?**

Другой момент, когда возникает несовместимость -- когда интерпретатор и компилятор используют разные синтаксические и/или семантические анализаторы. Кроме того, что это избыточно и коряво, это еще и гарантированный способ получить несовместимость. (И именно исходя из этого, на переходе с 2.6 на 3.0, команда CPython  отцепила хвостовой локомотив от поезда - модуль compiler исключен из дистрибутива.)

А между тем тот же Shed-skin этот модуль использует.

Есть интерфейс на уровне AST-дерева. Можно было бы использовать его. Но я предпочел прийти на готовенькое и в качестве входных данных для трансляции использовать питоновский байт-код. Да, это "немножко" неудобно. Но по крайней мере всегда можно открыть соответствующую строчку в интерпретаторе и посмотреть точную семантику. Причем в терминах С.

Правда, прежде чем странслировать в С, байт-код пришлось немного рекомпилировать до уровня выражений и структурных операторов (результат рекомпиляции можно посмотреть в файлах с расширением .pycmd. Это своего рода промежуточный результат трансляции.) А полученный псевдокод уже транслируем в С. С СОХРАНЕНИЕМ ДИНАМИЧЕСКОЙ СРЕДЫ ПИТОНА !!!

А сохранение среды означает использование в качестве основной структуры данных PyFrameObject объекта Питона. Вообще в интерпретаторе ее получает в качестве единственного параметра ф-я PyEval\_EvalFrame. Она содержит в себе ссылки на локальные и глобальные переменные, на выполняемый байт-код, на константы. Короче говоря, она полностью определяет контекст вычислений. И интерпретатор берет ее, раскрывает ее в локальные С-переменные, и обрабатывает код в ней.

Примерно так же работают и сгенеренные на основании байт-кода С-функции. (С некоторым отличием - вместо динамического моделирования стека значений при вычислении выражений исползуется статическое отображение стека на временные С-переменные типа `PyObject *`.) И тут мы получаем бонус -- раз структура данных стандартная, то передача/прием параметров происходят почти автоматом. То есть передача параметров между вызовами происходит пусть не очень быстро, но невидимо. Если кому интересно, попробуйте в Cython странслировать десятистрочную ф-ю с пятью параметрами. Увидите, что прием аргументов занимает больше места, чем собственно код ф-и.