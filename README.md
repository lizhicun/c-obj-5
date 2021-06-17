# 第五章 构造、解构、拷贝语义学（Semantics of Construction, Destruction, and Copy）
主要讲构造，析构和拷贝在不同的类中有不同的表现。

## 5.1 “无继承”情况下的对象构造
下面的 Point 的声明，是一种所谓的 Plain Old Data 声明形式：
```
typedef struct {
  float x, y, z;
} Point;
```
考虑下面的程序片段：
```
Point global;

Point foobar()
{
  Point local;  // 既没有被构造也没有被解构
  Point *heap = new Point; // 没有 default constructor 施行于 new 运算符所传回的 Point object 上
  *heap = local; // 赋值操作只是像 C 那样的纯粹位搬移操作
  // ...stuff...
  delete heap;  // destructor 和 constructor 一样，不是没产生出来，就是没被调用
  return local;  // 最后的 return 语句传值也是一样，只是简单的位拷贝操作
}
```
* 事实上，Point的trival constructor和destructor要不是没有被定义，就是没被调用。行为如它在c中的表现一样。  
* 但global是例外，在 C 中，global 被视为“临时性的定义”，因为它没有明确的初始化操作，会放在程序 data segment 中一个“特别保留给未初始化之 global objects 使用”的空间，这块空间被称为 BSS。而 C++ 并不支持“临时性的定义”，global 在 C++ 中被视为完全定义。C 和 C++ 的一个差异就在于，BSS data segment 在 C++ 中相对不重要。C++ 的所有全局对象都被当作“初始化过的数据”来对待。

### 抽象数据类型
Point 的另一种声明，提供了完整的封装性，但没有 virtual function：
```
class Point {
 public:
  Point( float x = 0.0, float y = 0.0, float z = 0.0 )
  : _x( x ), _y( y ), _z( z ) {}
  // no copy constructor, copy operator
  // or destructor defined ...
  // ...
 private:
  float _x, _y, _z;
};
```
我们也没有定义 copy constructor 或 copy operator，因为默认的位语义（default bitwise semantics）已经足够。也无需 destructor，默认的内存管理方法也够了
如果要对 class 中的所有成员都设定常量初值，那么使用 explicit initialization list 会比较高效，比如
```
void mumble()
{
  Point local1 = { 1.0, 1.0, 1.0 };
  Point local2;
  // equivalent to an inline expansion
  // the explicit initialization is slightly faster
  local2._x = 1.0;
  local2._y = 1.0;
  local2._z = 1.0;
}
```
local1 的初始化操作会比 local2 的高效。这是因为函数的 activation record 被放进程序堆栈时，上述 initialization list 中的常量就可以被放进 local1 内存中了。
这是因为函数的 activation record 被放进程序堆栈时，上述 initialization list 中的常量就可以被放进 local1 内存中了。
(Activation record is used to manage the information needed by a single execution of a procedure. An activation record is pushed into the stack when a procedure is called and it is popped when the control returns to the caller function.)

Explicit initialization list 带来三项缺点：
* 只有当 class members 都是 public 时，此法才奏效。
* 只能指定常量，因为它们再编译时期就恶意被评估求值。
* 由于编译器并没有自动施行之，所以初始化行为的失败可能性会比较高一些。

### 为继承做准备
```
class Point
 public:
  Point( float x = 0.0, float y = 0.0 )
  : _x( x ), _y( y ) {}
  // no destructor, copy constructor, or
  // copy operator defined ...
  virtual float z();
  // ...
 protected:
  float _x, _y;
};
```
这里并没有定义 copy constructor、copy operator、destructor。因为程序在默认语义之下表现良好.    
virtual function 的引入使得：
* object 拥有一个 virtual table pointer；
* constructor 被附加了一些代码，以便将 vptr 初始化。

### 继承体系下的对象构造
```
T obj;
```
constructor 可能会带有大量编译器为其扩展的代码：
* member initialization list 中的 data members 初始化操作会被放进 constructor 的函数本身
* 如果有 member 没有出现在 member initialization list 中，但它有一个 default constructor，那么该 default constructor 必须被调用
* 在那之前，所有上一层的 base class constructor 必须被调用
* 在那之前，所有 virtual base class constructors 必须被调用
* ... ...


### vptr初始化语意学
当我们定义一个 PVertex object 时，constructors 的调用顺序是:
```
Point(x, y);
Point3d(x, y, z);
Vertex(x, y, z);
Vertex3d(x, y, z);
PVertex(x, y, z);
```
假设：
1.每一个 class 都定义了一个 virtual function：size()，该函数负责传回 class 的大小。
2.每一个 constructor 内都带一个调用操作，比如：
```
Point3d::Point3d( float x, float y, float z )
: _x( x ), _y( y ), _z( z ) {
  if ( spyOn )
    cerr << "within Point3d::Point3d()"
         << " size: " << size() << endl;
}
```
问：每次对 size() 的调用都会被决议为 PVertex::size() 吗（毕竟我们正在构造的是 PVertex）
答：在 Point3d constructor 中调用的 size() 函数，必须被决议为 Point3d::size()

问：为什么必须决议为Point3d::size()
答：PVertex constructor 执行完毕之前，PVertex 并不是一个完整的对象.

此时我们的目标是：每一个 base class constructor 被调用时，编译器必须保证有适当的 size() 函数实体被调用。

问：如何保证这一点
答：constructor中的调用操作要以静态的方式决议，不使用虚拟机制

问：怎样保证以静态的方式决议
答：推迟size()函数成为虚函数的时间。这等同于推迟vptr的初始化时间

问：vptr什么时间初始化
答： base class constructors 调用操作之后;程序员供应的代码或是“member initialization list 中所列的 members 初始化操作”之前.

  这样，如果每个 constructor 都等到它的 base class 的 constructor 执行完毕之后才设定 vptr，那么每次就能够调用正确的 virtual function 实体。
  
  constructor 的执行算法通常如下：
1.在 derived class constructor 中，“所有 virtual base class”及“上一层 base class”的 constructor 会被调用。
2.上述完成之后，对象的 vptr 被初始化，指向相关的 virtual table。
3.如果有 member initialization list 的话，将在 constructor 体内扩展开来。这必须在 vptr 被设定之后才进行，以免有 virtual member function 被调用。
4.最后，执行程序员所提供的代码。

```
PVertex::PVertex( float x, float y, float z )
: _next( 0 ), Vertex3d( x, y, z ), Point( x, y ) {
  if ( spyOn )
    cerr << "within Point3d::Point3d()"
         << " size: " << size() << endl;
}
```
会扩展为：
```
// Pseudo C++ Code
// expansion of PVertex constructor
PVertex*
PVertex::PVertex( Pvertex* this, bool __most_derived,
                 float x, float y, float z ) {
  // conditionally invoke the virtual base constructor
  if ( __most_derived != false )
    this->Point::Point( x, y );
  
  // unconditional invocation of immediate base
  this->Vertex3d::Vertex3d( x, y, z );
  
  // initialize associated vptrs
  this->__vptr__PVertex = __vtbl__PVertex;
  this->__vptr__Point__PVertex = __vtbl__Point__PVertex;
  
  // explicit user code
  if ( spyOn )
    cerr << "within PVertex::PVertex()" << " size: "
         // invocation through virtual mechanism
         << (*this->__vptr__PVertex[ 3 ].faddr)(this)
         << endl;
  
  // return constructed object
  return this;
}
```

## 对象复制语义学
