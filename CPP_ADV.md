# C++ advanced
## rvalue-reference
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
  return (rand() % 2) ? a : b;
}
```
### move semanics
```
int a;

int& b = 42;  // error
int& c = a;   // ok

int&& d = a;  // error
int&& e = 42; // ok

int&& m = std::move(a); // ok

int&& q = m; // error - именованные ссылки всегда lvalue
```
### x-value
```
mytype prvalue();
mytype& lvalue();
mytype&& xvalue();

void foo(mytype const&);
void foo(mytype&&);

mytype test() {
  foo(prvalue()); // mytype&&
  foo(lvalue());  // mytype const&
  foo(xvalue());  // mytype&&
  
  return prvalue(); // RVO
  return lvalue();  // no-RVO
  return xvalue();  // no-RVO
}
```

Передача rvalue в конструктор, компилятором не читается:
```
mytype x = mytype(prvalue()); // не вызывает лишний раз конструктор
```
