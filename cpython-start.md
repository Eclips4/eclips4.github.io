Эталонной реализацией языка Python является именно
интерпретатор CPython.

В этой статье я расскажу о том, как же добавить
свою функцию в `builtins`.
Интерпретатор имеет некоторое количество встроенных в него функций и типов,
которые всегда доступны. Именно это и есть `builtins`.
Примером такой функции служит `print`, а примером такого типа служит `bool`.


Для начала, давайте сделаем `git clone` исходного кода интерпретатора.

`git clone --depth 1 --branch 3.11 https://github.com/python/cpython`

Давайте перейдем в директорию `cpython`

`cd cpython`
Теперь давайте определимся с тем, что же будет делатьт наша функция.

В `builtins` присутствует функция `abs`, её задача - вызывать у объекта dunder-method(магический метод)
`__abs__`. Обычно `__abs__` используют для того, что бы вернуть модуль какого-то числа.
Помимо dunder-method'a `__abs__` у числовых типов обычно присутсвует `__neg__`, который отрицает число.
Но в `builtins` функции `neg` которая бы вызывала этот dunder нет!
Именно реализацией этого мы с вами сегодня и займемся!

Стоит знать, что на самом деле `-number` вызывает `__neg__`, но отдельной функции под это нет.
Возможно с функцией это даже не очень-то и красиво, но мы с вами делаем это исключительно в научных целях.

Теперь давайте перейдем в `Objects/abstract.c`
После инклюдов, реализуем функцию `PyNumber_Negate`:
```c
PyObject *
PyNumber_Negate(PyObject *object) {
    if (object == NULL) {
        return null_error();
    }
    PyNumberMethods *methods = Py_TYPE(object)->tp_as_number;
    if (methods && methods->nb_negative) {
        PyObject *res = methods->nb_negative(object);
        assert(_Py_CheckSlotResult(object, "__neg__", res != NULL));
        return res;
    }
    return type_error("bad operand type for neg(): '%.200s'", object);
}
```
Эта функция получается в виде аргументов указатель на PyObject,
далее проверяем, не указывает ли `object` на `NULL`
Следущим шагом мы получаем методы `object`.
`if` Нам нужен что бы проверить, есть ли метод `nb_negative` и `__neg__` у объекта.
Если таковых нет, то возвращаем исключение `type_error("bad operand type for neg(): '%.200s'", object)
`
Иначе возвращаем `ng_negative` объекта, можно считать что это `__neg__`.

Давайте перейдем в `Include/abstract.h`
и объявим наш `PyNumber_Negate`:
```c
PyAPI_FUNC(PyObject *) PyNumber_Negate(PyObject *o);
```

Теперь давайте перейдём в `Python/bltinmodule.c`
Именно здесь <em>регистрируются</em> `builtins`.
Реализуем функцию `builtin_neg`
```c
static PyObject *
builtin_neg(PyObject *modle, PyObject *x){
    return PyNumber_Negate(x);
}
```
Это и есть реализация нашего будущего `neg`.
Она вызывает функцию `PyNumber_Negate`, которая в свою очередь и делает все проверки, и в случае их успеха, возвращает
нам `__neg__`.

Теперь нам придёться зарегистрировать наш `builtin_neg`! Вскоре мы вернемся к `Python/bltinmodule.c`.

Открываем файл `Python/clinic/bltinmodule.c.h`.
Здесь мы defin'им макрос `BUILTIN_NEG_METHODDEF`.
Но для начала, нам нужно написать для него небольшую документацию, которая будет отображаться при `help(neg)`.
```c
PyDoc_STRVAR(builtin_neg__doc__,
"neg($module, x, /)\n"
"--\n"
"\n"
"Return the negate value of the argument.");
```
А далее и дефайн макроса.
```c
#define BUILTIN_NEG_METHODDEF \
    {"neg", (PyCFunction)builtin_neg, METH_O, builtin_neg__doc__},
```

Суммарно это выглядит вот так:
```c
PyDoc_STRVAR(builtin_neg__doc__,
"neg($module, x, /)\n"
"--\n"
"\n"
"Return the negate value of the argument.");
#define BUILTIN_NEG_METHODDEF \
    {"neg", (PyCFunction)builtin_neg, METH_O, builtin_neg__doc__},
```
В дефайне макроса, как мы видим мы передаём название билтина `neg`, и функцию реализующую его `(PyCFunction)builtin_neg`
Теперь вернемся к `Python/bltinmodule.c`
Здесь нам нужно найти `static PyMethodDef builtin_methods[]`, и добавить в него наш макрос `BUILTNI_NEG_METHODDEF`
После добавления, `static PyMethodDef builtin_methods[]` будет выглядеть как-то так:
```c
static PyMethodDef builtin_methods[] = {
    {"__build_class__", _PyCFunction_CAST(builtin___build_class__),
     METH_FASTCALL | METH_KEYWORDS, build_class_doc},
    BUILTIN___IMPORT___METHODDEF
    BUILTIN_ABS_METHODDEF
    BUILTIN_ALL_METHODDEF
    BUILTIN_ANY_METHODDEF
    BUILTIN_ASCII_METHODDEF
    BUILTIN_BIN_METHODDEF
    BUILTIN_NEG_METHODDEF // а вот и наш макрос!
    // здесь еще куча макросов и остального
};
```
Ну вот и всё, теперь нам осталось как-то запустить наш изменённый интерпретатор.
Для этого, нам придётся его собрать из исходников
Для начала, перемещаемся в корень директории `cpython`

Если у вас UNIX like OS, то для того что бы начать сборку вам нужно ввести в терминал следующее:
```
./configure --with-py-debug
```
Флаг `--with-py-debug` означает что мы собираем Python для дебага, и там не будет некоторых оптимизаций, что, конечно, ускорит сборку.
Далее вводим `make -j 2` за что отвечает `-j 2` можно прочитать в `man make`

Если у вас Windows, нужно ввести следующее:
```
PcBuild/build.bat -c Debug
```
Здесь мы опять же собираем питон с флагом `-c Debug`, что бы сборка была быстрее.

Всё! Готово. И да, команды описанные выше - могут выполняться долго, всё зависит от вашего железа.

Давайте создадим в корне директории `cpython` файл `test_interpreter.py`, и протестируем наш `neg()`

```python
help(neg)
```
Запустим наш `test_interpreter.py` с помощью нашего нового интерпретатора
```c
./python test_interpreter.py
Help on built-in function neg in module builtins:

neg(x, /)
    Return the negate value of the argument.

```
Прекрасно! Та самая небольшая документация что мы писали, работает.
Теперь изменим `test_interpreter.py`:
```python
print(neg(1))
print(neg(-1))
```
```
./python test_interpreter.py
-1
1
```
Отлично. Всё работает как нужно. Теперь давайте проверим как это будет работать
с классами, которые определит пользователь.
```python
class SomeClass:
    def __init__(self, number: int) -> None:
        self.number = number
    def __neg__(self):
        return -2 * self.number
```
Мы реализовали `__neg__` у класса, который будет возвращать `-2 * number`
Давайте проверим!

```python
test = SomeClass(5)
print(neg(test))
```
```
./python test_interpreter.py
-10
```
Чудно! Всё работает ровно так как мы ожидали.
Теперь, попробуйте реализовать какую-нибудь не столь бесполезную функцию как та что описана в статье =)

Если у вас возникли трудности, вы можете взглянуть на этот форк CPython: https://github.com/Eclips4/cpython_articles/tree/3.11
В частности, на [этот](https://github.com/Eclips4/cpython_articles/commit/97965a553096e1e6121ec032ab26a2794a40f315) коммит.

Спасибо за внимание.