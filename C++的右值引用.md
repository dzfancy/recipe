## C++11的右值引用

### 要解决的问题
* c++11中引入右值引用是为了解决之前0x标准中的三个问题
	* 对象作为函数的返回值的额外拷贝开销
	* 临时对象作为拷贝构造函数入参额外的拷贝开销
	* 函数模板只能根据形参决定函数调用

#### 对象作为返回值的额外拷贝开销
* 如下面的代码所示,在**没有任何优化措施**的情况下，这段代码将会执行一次默认构造函数，两次拷贝构造函数；一次构造是Foo()调用的开销，Foo()调用产生的是栈上对象，在函数执行完成之后就会被释放，因此，需要一个临时值存放栈上对象，以供赋值动作使用；赋值动作本身会调用第二次拷贝构造；当然，编译器会做返回值优化，减少不必要的两次拷贝构造，但这是编译器做的事情，不是C++语法的提供的便利

~~~c++
class Foo {
public:

    Foo() {
        cout << "Foo default constructor" << endl;
    }

    Foo(const Foo &theFoo) {
        cout << "Foo copy construct called";
    }

};

Foo getFoo(){
    return Foo();
}
int main() {

    cout << "bind with object" << endl;
    Foo foo1 = getFoo();
    cout << "bind with const reference" << endl;
    const Foo &foo2 = getFoo();
    cout << "bind with rvalue reference" << endl;
    Foo &&foo3 = getFoo();
    return 0;
}
~~~

* 由于返回的对象是右值，如果对数据没有写的需求，可以右值引用绑定;在***没有任何优化***的条件下，使用右值引用可以减少一次拷贝构造的调用;在C++0X标准中，可以使用常量引用来绑定函数返回的对象，也可以减少一次拷贝构造函数的调用
* 值得注意的是，使用右值引用或者常量引用绑定临时变量可以延长变量的生命周期，变量的生命周期变的与绑定的引用一样长


### 临时对象拷贝的额外开销
* 我们都知道，如果一个对象持有堆内存，在执行拷贝构造的时候需要进行深拷贝，防止由于重复释放内存导致的进程崩溃

~~~c++
LargeHeap(){
    length = 10000;
    largeBuffer = new int(length);
}


    LargeHeap(const LargeHeap& theOther){
        largeBuffer= std::copy(theOther.largeBuffer,theOther.largeBuffer+theOther.length,largeBuffer);
        length = theOther.length;
    }
	
    ~LargeHeap(){
        delete largeBuffer;

    }
private:
    int * largeBuffer;
    int length;

};

LargeHeap getLargeHeap(){
    return LargeHeap();
}
int main() {
    LargeHeap largeHeap = getLargeHeap();
    return 0;
}
~~~

* 上面的代码中，我们通过`getLargeHeap`函数获取一个带有大堆内存的临时对象，然后用这个临时对象初始化另外一个具名对象，这回调用类的拷贝构造函数，拷贝完成之后，临时对象就会自动销毁，在这过程中，堆内存会重复的创建销毁，如果内存很大会降低程序的运行效率
* 如果一个对象在构造完成另一个对象之后生命周期就会结束(不限于临时对象)，堆内存可以直接`移交`给新的对象，不必执行拷贝的动作；


~~~c++
	LargeHeap(LargeHeap &&theOther) noexcept {

        largeBuffer = theOther.largeBuffer;
        length = theOther.length;
        theOther.largeBuffer = nullptr;
        theOther.length = 0;
    }
~~~

* 上面的代码展示了‘移交’的动作，又用到了右值引用，这里有一个问题，临时对象既然可以被常量引用绑定，又可以被右值引用绑定，那绑定的时候的优先级是怎么样的呢？
* 如果一个对象是左值，如何将数据移交给另外一个对象呢？c++11中提供了`std::move`方法，此方法可以把左值转换成右值

### 函数模板只能根据形参决定函数调用

### C++的值类型

#### 左值与右值
* 在C++11标准之前，一个表达式只有两种值类型(value category),左值(lvalue)与右值(rvalue)。左值，顾名思义，可以出现在复制表达式的左侧。左值不是临时变量，而且可以出现在多个表达式当中;右值，可以理解为只能出现在表达式的右侧，右值通常是临时值，声明周期与所在的表达式一致；
* 在C++11标准之前，右值是只能被引用，最多只能绑定常量引用

~~~c++
int &i=0; /错误，非常量左值引用不能绑定临时值(右值)
const int &j =0; /正确，使用右值绑定常量左值
~~~

#### 将亡值
c++11增加了新的值类型--将亡值，顾名思义，将亡值就是生命周期快要结束的表达式的值类型，为什么要添加将亡值，需要从右值引用说起

### C++98之痛

* 右值引用的出现，主要是为了解决两个问题，一个是临时对象作为参数需要额外的拷贝开销，另一个是函数模板中的参数类型的精确转发

#### 临时对象作为参数的额外开销
* 将函数模板与右值引用组合在一起的时候会发生比较奇妙的作用：当函数模板的模板类参数是右值引用，在函数模板实例化时，如果传递的模板类参数是左值，则实例化的模板函数入参也是左值，否则，得到的将是平凡值；
* 这一点听起来不应该是理所当然的吗？实际上在C++11之后之前不是的，看下面的代码片段

~~~c++
void g(int &i){
    cout << "this is lvalue reference" << endl;
}

void g(const int &i){
    cout << "this is rvalue reference" << endl;
}

template <typename T>


void func1(T & t){
   g(t);
}

template <typename T>
void func2(const T& t){
    g(t);
}
~~~

* 在上面的代码中，有两个g函数，区别是入参分别是左值引用与右值引用：对于函数模板func1，由于其声明的入参类型是左值，在编译的时候会调用参是左值的g函数；对于fun2，声明的形参是右值，无论实参是左值还是右值，都只能调用右值版本的g函数；总之，模板函数实例化时，只能根据形参的类型，而不是实参的类型，决定调用哪个版本的函数

* c++ 11的完美转发解决了这个问题,如下面的函数func3，当传递的实参是左值时,`std::forward<T>`没有做任何事情，只是把左值传递给g函数；如果实参是右值，`std::forward<T>`就等价于`std::static_cast<T&&>`,将形参转化成右值，调用右值版本的g函数；

~~~c++
template <typename T>
void func3(T && t){
    g(std::forward<T>(t));
}
~~~

* 完美转发只有在函数模板的场景下才有用，普通的函数没有之一功能

~~~c++
func4(int &&t){
	g(t);
}
~~~
* 上面的代码只能调用右值版本的g函数


### 将亡值(xvalue)

#### 左值
* 顾名思义，左值可以出现在赋值表达式的左侧的表达式；左值不是临时变量，并且左值可以用在做个表达式中，
* 任何的C++表达式都是左值或者右值之一，不会存在例外.左值是可以独立于表达式存在的值，有名字的值都是左值，任何的变量都是左值，右值是一个临时值，声明周期不能超越使用它的表达式；


~~~c++

int i = 0;
int &j =i;

~~~
 int &绑定了一个左值，