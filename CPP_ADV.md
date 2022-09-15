# C++ advanced
## rvalue-reference, move semanics
```
  mytype g() {
    return mytype();
  }
  ...
  mytype y;
  f(y);        // y - lvalue, copy
  f(mytype()); // mytype() - rvalue, no copy
  f(g());      // g() - rvalue, no copy
```
Более того, g интерпретируется следующим образом:
```
  void g(mytype* result) {
    mytype_ctor(result);
  }
```
Что помогает избегать лишнего копирования.

Такая оптимизация проходит для всех функций, которые возвращают rvalue (у lvalue могут быть проблемы). Называется **RVO - return value opimization**

Пример нормальной работы для lvalue: **NRVO - named return value opimization**. Реализован в почти всех компиляторах, но не является стандартом.
```
mytype g() {
  mytype tmp;
  // работа с tmp
  return tmp;
}

// превращается в
void g(mytype* result) {
  mytype_ctor(result);
  // работа с result  
}
```
Пример продолбов у NRVO
```
mytype g() {
  mytype a;
  mytype b;
  return (rand % 2) ? a : b;
}
```
