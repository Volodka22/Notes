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
## move semanics
```
int a;

int& b = 42;  // error
int& c = a;   // ok

int&& d = a;  // error
int&& e = 42; // ok

int&& m = std::move(a); // ok

int&& q = m; // error - именованные ссылки всегда lvalue
```
## x-value
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
## intrusive контейнеры
Чтобы не плодить next-prev'ы и не выделять доп памяти под ноды
```
tepmlate <typename Tag>
struct list_element {
  list_element* prev;
  list_element* next;
}

struct unit 
  : list_element<struct all_units_tag>,
    list_element<struct selected_units_tag> 
{   
}

intrusive_list<unit, all_units_tag> all_units;
intrusive_list<unit, selected_units_tag> selected_units;

```

##  shared_ptr
### aliasing_constructor

Суть хака в том, что мы создаем shared_ptr, которое ссылается на счетчик от машины и на колесо. Как итог, машина не будет удалена, пока есть расшаренное колесо (такой подход бывает нужен, когда часть не имеет смысла без целого) 

```
struct wheel
{
    int a = 42;
};

struct vehicle
{
    std::array<wheel, 4> wheels;
};

int main()
{
    std::shared_ptr<wheel> w;

    std::shared_ptr<vehicle> v(new vehicle);    
    w = std::shared_ptr<wheel>(v, &v->wheels[2]);
}
```


### weak_ptr
shared_ptr - увеличивают число сильных ссылок, продлевая жизнь самому объекту
weak_ptr   - увеличивает число слабых ссылок, продлевая жизнь своей ноде, после удаления объекта нода указывает на nullptr
Пример парного использования:
```
struct widget
{};

std::shared_ptr load_widget(int id);

std::shared_ptr get_widget(int id) {
  static std::map<int, std::weak_ptr<widget>> cache;
  auto sp = cache[id].lock();
  if (!sp) cache[id] = sp = load_widget(id);
  return sp;
}

```
Минус очевиден - в мапе мнорго бесполезных ссылок. Решение - написать deleter:
```
struct cache
{
private:
    typedef std::map<int, std::weak_ptr<widget>> objects_t;
    struct deleter;

public:
    std::shared_ptr<widget> alloc(int value) {
      auto it = objects.find(value);
      if (it != objects.end()) {
        std::shared_ptr<widget> obj = it->second.lock();
        assert(obj);
        return obj;
      }

      it = objects.insert(it, {value, std::weak_ptr<widget>()});

      std::shared_ptr<widget> obj;
      try {
          obj = std::shared_ptr<widget>(new object(value), deleter(this, it));
      }
      catch (...) {
          objects.erase(it);
          throw;
      }
    
      it->second = obj;
      return obj;
    }

    size_t size() const {
      return objects.size();
    }
    
private:
    objects_t objects;
};

struct cache::deleter
{
    deleter(cache* p, objects_t::const_iterator it) 
              : p(p)
              , it(it)
    {}
    
    void operator()(widget* obj) const {
      p->objects.erase(it);
      delete obj;
    }
    
private:
    cache* p;
    objects_t::const_iterator it;
};

```
Лучше, но все еще много кусков памяти аллоцированно. Давай менять:
```
struct cache
{
private:
   struct object_container;
   typedef std::map<int, std::weak_ptr<object_container>> objects_t;
   /.../
};

struct cache::object_container
{
    object_container(object_container const&) = delete;
    object_container& operator=(object_container const&) = delete;

    object_container(cache* p, objects_t::const_iterator it, int value)
          : p(p)
          , it(it)
          , obj(value)
    {}
    
    ~object_container() {
        if (it != p->objects.end())
            p->objects.erase(it);
    }
    
private:
    cache* p;
    objects_t::const_iterator it;
    object obj;

    friend struct cache;
};

std::shared_ptr<object> cache::alloc(int value)
{
    std::shared_ptr<object_container> cont;

    auto it = objects.find(value);
    if (it != objects.end())
    {
        cont = it->second.lock();
        assert(cont);
    }
    else
    {
        cont = std::make_shared<object_container>(this, objects.end(), value);
        cont->it = objects.insert(it, {value, cont});
    }

    return std::shared_ptr<object>(cont, &cont->obj);
}
/.../
```

### enable_shared_from_this
Можно создать метод делающий shared_ptr от this:
```
  struct mytype : std::enable_shared_from_this<mytype> {
      std::shared_ptr<mytype> foo() {
          return shared_from_this();
      }
  }
```

## perfect forwarding

```
using type1 = int&;
usint type2 = type1&;
static_assert(std::is_same_v<type1, int&>);
```
& & -> & (ссылка на ссылку == ссылка)

&& & -> &

& && -> &

&& && -> &&

lvalue всегда выигрывает.

```
/* f с несколькими перегрузками */

template<typename T>
void g(T&& x) {
  f(forward<T>(x)); // тоже что и static_cast<T&&>(x);
}

int main() {
  g(42); // T -> int
  int x = 43;
  g(x); // T -> int&; а как известно применение && к int& - это int& 
}
```

```
template<typname.. T>
void f(T... args) {
  g(z(args...)); // g(z(arg0, arg1...)
  g(z(args)....); // g(z(arg0), z(arg1), ...)
}
```

### trailing return type

```
char f(int&);
std::string f(std::string);

??? g(T&& x) {
  return f(std::forward<T>(x));
}
```

Два решения:
```
template<class T>
T&& declval() noexcept;

template<typename T>
decltype(f(declval<T>())) g(T&& x) {
  return f(std::forward<T>(x));
}
```

Идея -> то что внутри decltype() не вычисляется, как и sizeof()

```
template<typename T>
auto g(T&& x) -> decltype(f(std::forward<T>(x))) {
  return f(std::forward<T>(x));
}
```

Использовать новый синтаксис

ИЛИ новейший

```
template<typename T>
decltype(auto) g(T&& x) {
  return f(std::forward<T>(x));
}
```

## Статический и динамеческий полиморфизм

Статический:
```
template <typename T> 
struct less {
  bool operator()(T const & a, T const& b) const {
    return a < b;
  }
}
```
Динамический:
```
bool int_less(int a, int b) {
  return a < b;
}
```

Статически компаратор предсказывается куда лучше, поэтому быстрее. Но можем получить много копий одного объекта в некоторых случаях

## Лямбды

```
int x, y, z;
[x, &y, z](){}
[=, &y](){}
[&,x](){}
[=](){}
[&](){}
```
[&] и [=] значат - забрать все клнтекстные переменные по ссылке/значению

```
[]<typename T>(T a, T b) { return a < b; }; // можно использовать template
[](auto a, auto b) {...} // а можно auto (можно использовать и в функциях)
```

## constexpr

Указывается перед **функцией**, давая компилятору сигнал, что стоит попробовать посчитать ёё в компаил тайм. 
Если во время компиляции посчитать не получилось, то будет вычислено во рантайме. С каждой новой версией компилятора 
ограничения на то, что можно писать внутри констекспр функций послабляют (сейчас нельзя реинтрепрет каст, goto и ...). 
Если внутри функции есть мемори лики или UB, то будет считаться в рантайм.

Указывается перед **переменной**, если хотим подставлять ее в шаблоны, например.
Есть еще constinit - это как constexpr, только не делает переменную константной. 

Еще есть consteval - для вычиления только во время компиляции (нигде не используется, кроме одной функции в библиотеке).

## Концепты

В первую очередь придуманы, чтобы сделать понятней ошибки при компиляции у функций с шаблонными параметрами. Во вторую сильно упрощают написание некоторых штук, а также несколько быстрее SFINAE.

**Примеры:**

```
template <typename T>
concept destructible = std::is_nothrow_destructible_v<T>;

template <typename T>
requires destructible<T>
void foo(T&) {}

template <destructible T>
void bar(T&) {}

void baz(destructible auto&) {}
```
Все три функции эквивалентны.

Сравниваются концепты между собой через приведение к нормальной форме. Должны ссылаться на один и тот же концепт для равенства.

```
template <typename U, typename V>
concept half_same_as = std::is_same_v<U, V>;


template <typename U, typename V>
concept same_as = half_same_as<U, V> && half_same_as<V, U>;
```

## Многопоточность

```
#include <thread>

int main() {
  thread th([] {
    std::cout << "H";
  });
  
  th.join(); // -> запустить
  // th.detach(); -> запустить без привязки к имени
  
}
```

### мьютексы

```
#include <mutex>

std::mutex m;

int f() {
  m.lock();
  // крит секция
  m.unlock();
}
```

Есть обертка std::lock_guard lock(mutex). Она вызывает unlock() в своем деструкторе (помогает с пробрасыванием ошибок)

Также есть recursive_mutex, которая позволяет одному потоку несколько раз залочить один и тот же тред (но анлокоа после должно быть столько же)

###  std::condition_variable

```
 std::condition_variable cv;

void push(T x) {
  std::lock_guard lock(m)
  // ...
  cv.notify_one();
}

T pop(T x) {
  std::unique_lock lock(m); // можно лочить и анлочить назад
  cv.wait(lock, [] { return !q.empty() });
  
  // ...
}

```
