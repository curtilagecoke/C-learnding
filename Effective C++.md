## 条款2:尽量使用const，enum，inline替换#define

- 对于单纯常量，尽可能地使用const对象或enums替换#define
- 对于形似函数的宏，最好改用inline函数替换#define

## 条款3:尽可能使用const

const针对成员函数:

有个很好玩地事实，关于const的看法分为两派，第一派则是认为const不应改变成员内的任何一个bit，但是这存在一个问题，例程如下:

```c++
class string{

public:

	char& operator [](std::size_t position) const

	{return ptext[position];}

private:

	char* ptext;

};
```

这里确实很正常的使用[]访问const string类型的字符串的下标所在位置的元素，但是请注意，const ctext ctb("hello"); char *pc=ctext[0];\*pc='J'; 如果是这样操作，这时候的ctext就会变成"Jello"了，这与我们给ctext上的const属性相违背。

针对另一派对const的看法，则是可以修改const对象中的某些bit，在客户端检测不出的情况下，例如类中的size和bool_length，他们也许是随时可变的，但如果是const对象，他们就无法修改，应对方法就是对这两个变量加上mutable(可变的)，这样他们就能在const成员函数中去修改他们的值了。

对于上述这个增加mutable关键字的做法，还不足以应对所有情况，对于non-const成员函数与const成员函数同时存在，并且实际上是实现一个功能的时候，我们可以为了避免代码膨胀，而是在使用non-const成员函数时，先使用static_cast对*this转型为const类型，去调用其const成员函数的版本，在最后返回值时，做const_cast去掉const属性。

## 条款4:确定对象在使用前已被初始化

实际上构造函数是有2个动作的:1.初始化 2.函数体内动作，所以构造函数的初始化列表往往比较高效，我们不应该放弃初始化的动作而直接做赋值动作，这是浪费了一部分机会，如果成员对象时一个类的话，那么更是如此，在初始化列表中只调用一次默认构造函数，要比进行一次默认构造再做一次拷贝构造要快得多。并且对于成员变量时const或者reference，他们就一定要初值，不能被赋值。

对于不同源文件中，互相会用到对方的对象，那么这时候需要保证被使用到的这个对象要先于当前使用他的对象之前就初始化了，但是这个能怎么确定呢？这里提供了一个手法，将这些对象都包在各自对应的函数之中，作为一个local-static对象，那么在使用他只是，只需要调用这个对应函数，返回一个reference，那么就可以调用这个对象，并且多个成员都可以共享这一个副本，并且生成一个static对象，之后直接返回他，在第一次需要调用构造函数，之后的每一次就不再需要了，并且代码量小，很容易作为inline函数来进行使用，在频繁使用的情况下效率会有提升。

## 条款5:了解C++默默编写并调用了哪些函数

系统在一个空类中，会根据编译器需要，自动生成构造函数，拷贝构造函数，拷贝赋值函数，析构函数。但是要特别注意，对于内含指针或引用的类，我们需要自定义这些函数，因为有关于指针的操作他并不会帮我们自动生成于释放，有可能出现内存泄露的危险。

## 条款6:不想使用编译器自带的默认函数，就要明确拒绝

例如我们需要禁止A的对象做拷贝动作，那么我们就要对拷贝构造函数与拷贝赋值函数做一些小动作，但我们如果不声明，他自己也会默认生成两个，所以我们的解决方案可以是在借助隔离等级private内来声明拷贝函数但不加以定义。但是这存在一个问题，即是这样，我们依旧有可能访问到这两个拷贝函数，借助成员函数以及友元函数即可做到访问。为了解决这个问题，我们可以衍生出一个仅有函数定义的普通基类，其可以使用的函数定义在protected部分，拒绝操作的定义在private部分，并且在基类使用他的时候，使用private继承，保证隔离等级如上，在运行期间，尝试做赋值动作的，都会跳转到基类，会发现不可访问而被拒绝。

这里解决了个人对继承与隔离等级的一个理解，对于private继承，这里的对于派生类的影响是，无法通过派生类对象直接访问基类的成员，但是基类中，public，protected等级的成员函数，在派生类中，依旧可以通过派生类成员函数来进行访问，private成员则不行，需要派生类先调用基类的public或protected的成员函数，再通过其去调用基类本身的private成员。

其实这里在C++有提出两个关键字应对这个问题，=default，=delete。=default为应用所对应函数的默认版本，而=delete是禁用这类函数的默认版本。

## 条款7:为多态基类声明virtual析构函数

带有多态性的继承关系，都是要为析构函数带上virtual关键字的，因为如果这是由一个继承关系，A<-B<-C，我们实例化了一个A指针，指向C对象，这时候他的内存布局为A，B，C三个成员的数据大小组合在一起，那么出现的问题也很简单，如果没有这个虚析构函数，那么程序结束，析构的结果只有A会发生析构，而其他的内存完全不受我们控制，丢失了，但我们也无法使用他，这就是内存泄漏，大量的内存泄漏最终会使得程序无内存可用，这是一件灾难性的事情，所以针对于多态性这种需要创建指针指向的情况，我们需要做虚析构函数，那么如果是普通继承无多态性的类呢？那实际上根本没有必要，实例化对象已经可以满足他的功能了，而实例化对象的内存是连在一起的，析构时会一并回收。

## 条款8:别让异常逃离析构函数

C++本身并不禁止析构函数抛出异常，但往往会有如此场景，当我们使用一个容器，例如vector容器时，其内部元素类型是widge，那么在容器使用全部结束之后，容器本应该把内部的这些元素全部析构掉，但这时候出现了第一个危险场景，他的第一个元素在析构的时候抛出了异常，那么这时候程序也许会无视他，又或者有自己的动作来处理异常，若没有处理，那么跳转到第二个元素，如果第二个元素也在析构的时候抛出了异常，那么这时候C++会认为程序的异常太多了，只有两种情况，1.过早结束程序 2.出现未明确行为，这两种行为都是危险的。针对C++析构函数时抛出的异常，提供了两种处理方案:

1. 直接吞下这个异常，不做处理。这并不是一个很好的做法，因为他依旧没法对错误有任何动作，但他依旧可以保证程序的执行。
2. 碰到异常就直接杀死程序，做abort操作，规避掉之后可能出现的不明确行为。

这里有一个更佳的做法即是自己重新设计析构函数内的动作，通过析构函数内调用外部自定义函数去进行操作，在自定义函数没法处理时，再交还由析构函数来进行处理。

综合上述说法，如果某个操作可能在失败时抛出异常，而又存在某种需要必须处理该异常，那么这个异常必须来自析构函数以外的某个函数，因为析构函数吐出就是危险。

- 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何一场，然后吞下他们或者终止程序。
- 如果客户端需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数执行该操作。

## 条款9:绝不在构造和析构过程中调用虚函数

首先，明确一个问题，虚函数是针对类的，但虚表指针是给对象的，所以在构造函数中，如果是有一个继承体系，基类是先于派生类进行构造的，如果这时候基类调用虚函数，想要使用派生类的信息的话，是没法成功的，他调用的必定是自己的版本。因为这时候内层基类先进行构造，生成了一张虚表，而虚函数时通过访问虚表来决定调用的，所以这时候他调用的是基类的虚表内的那个虚函数，而之后派生类生成一张新的虚表，会在原虚表上进行覆盖。所以在构造过程中，base class的成员函数，不会下降到derived class阶层。在构造期间，虚函数就不是虚函数。

同理，析构函数也是类似的，当derived class的析构函数开始调用的时候，编译器会视derived class中的成员都不存在，由此析构至base class的时候，调用虚函数，同样没法起到作用。

如果确实在base class中用到derived class中的信息，那么可以选择在base class的构造函数内，调用一个非虚函数，而这个非虚函数内接受的参数，是构造函数的参数，这个参数可以由derived class在其构造函数的初始化列表时提供。

## 条款10:令operator =返回一个reference to *this

由于C++语法上兼容连=，连+=这样的操作，例如a=b=c，这样如果是返回值是by value的形式，那么就失败了，为了应对这种情况，我们的赋值操作返回值就应该是by reference的形式，其返回的引用作用在左边对象上。

- 令赋值操作符返回一个reference to *this

## 条款11:在operator =中处理自赋值操作

自我赋值给人感觉是一种很蠢的操作，但实际上有的时候会间接的写出这样的代码，例如\*pb=*pd，也行他们指向的就是同一块区域，那么这时候如果这是某个class中的一个point的话，他的operator =如果写成delete pb;pb=pd，那么这时候就出现了一个问题，原来他们两个指向的地址都是同一个，而pb操作就直接将那块地址给释放了，那么相应的pd指向的也是一个空的区域，那么这个操作即使可以执行，但也失去了意义。所以这时候做自赋值检测是一件非常有必要的事情。首先传进来的一定需要是引用或是指针，因为地址相同才意味着他们是相同的。然后这时候在进入运算符重载之时，就要先判断当前\*this是否与传入引用相同，相同即无需处理。但是这样的处理只能保证自赋值的安全行，但不具备异常安全性，所以这时候得手法可以变化为，传进来时，为传入的引用参数做一份数据副本，然后做swap操作，这样可以相对满足这两个条件。

- 确保当对象赋值时operator =有良好的行为，其中技术包括比较“来源对象”和“目标对象”的地址、语句顺序、以及拷贝交换
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

## 条款12:复制对象时勿忘其每一个部分

考虑继承中的情况，派生类的拷贝函数只拷贝了派生类的数据成员，而并没有对其所带有的基类部分做操作，很明显，这是不对的，因为这拷贝并没有完全，不是我们所理想的效果，所以我们需要在拷贝函数中去显式的调用基类中的拷贝函数，这样才能实现数据的完全拷贝。

- 拷贝函数应确保复制“对象内的所有成员变量”及“所有base class成分”。
- 不要尝试以某个拷贝函数实现另一个拷贝函数，应该将共同部分放入第三个独立函数供2者调用。

## 条款13:以对象管理资源

我们往往会使用一个指针去做各种操作，但要注意，指针分配完之后，这个内存是开辟在堆上的，除非我们去显式的释放他，否则他会一直都在堆上，并不会自己释放。考虑一个问题，当前我有一个指针对象，在他函数末尾有一个delete函数，这原本应该是一个正确的行为，但在代码维护中，会出现一个问题，也许在delete之前，就已经return或者continue掉了，那就轮不到这个delete方法来进行执行，那么出现的问题就是刚才的指针对象的内存没有办法再控制了，他占着那块内存，但我们并没有办法找到它。这时候我们就会考虑用类的一个特性:生命周期结束时自动调用析构函数这个特点来处理他，那么在析构函数中增加delete操作，可以保证在脱离其生命周期时，自动销毁指针成员。那么这时候我们就可以封装一个像指针的类来进行资源管理，这就是智能指针。那么智能指针的管理形式其实是RAII(当获取到资源时，便会对资源立刻进行初始化)，我的理解是初始化指的就是类内的指针成员这时候接收这个资源并进行了初始化。然后在之后的使用中，使用其重载的*与->符号，同样可以实现类似指针的各种操作。但是要注意，如果是auto_ptr这种第一版智能指针，也许他会缺少考虑，因为也许会有多个智能指针对象指向的是同一块内存，这时候如果存在一个已经析构了，那么就会引发其他几个智能指针对象会对一块空地址去再进行delete，这是个错误的行为，所以在之后的改进中，增加了资源计数或者独占指针等行为来解决这类问题。

- 为防止资源泄漏，请使用RAII对象，他们在构造函数中获得资源并在析构函数中释放资源。

## 条款14:在资源管理类中小心copying行为

当一个RAII对象在被复制的时候，我们并不能确保他之后的动作，也许这个副本先于我们原始使用的这个原本先释放了，那么原本所占有的资源就被释放掉了，这是一个很危险的行为，他影响到了别人的情况。那么这时候，我们可以考虑两种可行方案:

1. 禁止这个RAII对象的拷贝，例如条款6的操作，将这个拷贝函数放入private等级区内，那就禁止了拷贝行为，可是这并不合理，资源本身就是可以被共享的，独占式的资源式偏少的，不符合实际上的应用。
2. 采取资源计数器来进行管理，为了避免数据copying出去之后不受自身管控，那么我们可以设置一个计数器，每有一个新的对象来共享这个资源时，分配出对象，计数器+1，他们不使用时，就计数器-1，直到计数器为0时，才释放这个资源，那么这时候资源可以由大家共同控制，满足了所有人的需求之后再给他进行释放，这是合理的。

- 复制一个RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。

## 条款15:在资源管理类中提供对原始资源的访问

例如说有一个成员函数，接受的是指针类型，但这时候我们有的是一个RAII对象，一个smart point，那么传进去做参数的话，是不被认可的，所以smart point的底层实际上由提供一个get()方法直接访问它的指针资源，但这似乎有些太麻烦了，所以我们也可以考虑隐式的类型转换，即operator 类型名称()这样来实现隐式的，类C里面的类型转换，方便用户操作，当然，显式转换与隐式转换的选择，要针对具体的使用情况。

- APIs往往要求访问局部资源，所以每一个RAII对象应该提供一个“取得其管理的资源”的方法。
- 对原始资源的访问可能经由显式转换或隐式转换，一般而言显式转换更为安全，隐式转换更为方便。

## 条款16:成对使用new和delete时要采用相同的形式

new分为array new与单个资源的new，array new的行为是在底层开辟一块连续空间，在这块空间内的头部会有一个地址存放这个数组的大小，对应的array delete，即delete[]，会明确要释放的是数组，会去探查这个数组的大小，然后进行释放。如果他使用的是delete，那么他只会释放一个位置，即我们要释放的数组的首位置，这就出现了大量的内存泄露了。

- 如果在new表达式中使用了[]，那么对应的delete也需要使用[]。

## 条款17:以独立语句将newed对象置入智能指针

假设如此场景:int priority();void process(shared_ptr\<widge> pw,int priority)，这时候如果函数调用是process(new widge,priority())，他是不能通过编译的，因为shared_ptr的构造函数是一个explcit函数，无法进行隐式转换，所以我们可以写成process(shared_ptr\<widge>(new widge),priority())，这样通过编译了，因为这是使用了一个已有对象来对shared_ptr进行的初始化。然而我们需要考虑函数中的参数的一个调用问题，很明显的，申请空间一定是先于构造shared_ptr对象的，而priority()则不能确定，所以这样的一种情况产生了:①new widge ②调用priority() ③调用shared_ptr构造函数。那么问题出现了，如果在priority的调用过程中抛出了异常，那么就无法执行shared_ptr的构造函数了，那么new widge这块内存就无人管理，一直遗留在内存之中，出现了内存泄漏。如果想要避免这类问题，可以采取以下办法:shared_ptr\<widge> pw(new widge);process(pw,priority());这种手法，将构造语句单独分离出来，编译器对于跨越语句的各项操作没有重新排列的自由，所以这时候分配内存与priority()调用是完全隔离开的动作。

- 以独立语句将newed对象存储于智能指针内，如果不这样做，一旦有异常抛出，有可能导致难以察觉的内存泄漏。

## 条款20:宁以pass-by-reference-to-const替换pass-by-value

 想像这个场景:people类中，拥有2个string成员与1个拷贝构造函数，student类继承自people类，其内部也有2个string成员与1个拷贝构造函数，这时候我们调用一个函数use_people(student test)，调用行为为student test1;use_people(test1)，这样的传值调用将会以test1为蓝本，拷贝生成一个test，这只是其中一个拷贝构造函数，在这之内还有2个string成员的拷贝构造函数，1次people的拷贝构造函数，2次people::string对象的2次拷贝构造函数，总计6次，这是一个很大的开销。析构同理也是6次。但如果是pass-by-reference呢？他就没有新的对象会生成，所以就省去了这些动作。但是如果是pass-by-reference，并且不对其进行修改的话，要注意将其定义为const，因为如果没有const，编译器会疑虑这个元素究竟会不会进行修改，若是修改了则会对原来的那些数据带来改变。其次，传入引用，可以避免对象切割的问题。传值情况，如果use_people的形参变量的类型是people，那么我们传入一个student类型的变量时，他会发生切割，只能调用people所有的那些函数与成员，而原来的student中特有的成员函数，成员变量，都遭到了切割，无法使用。使用传引用会发生隐式的类型转换，使得传进来的变量是什么类型，在函数内表现的就是什么类型。当然，我们也可以合理假设，除了内置类型与STL迭代器和函数对象这种“pass-by-value并不昂贵”的类型之外，其他任何东西都建议pass-by-reference-to-const。

- 尽量以pass-by-reference-to-const替换pass-by-value，前者比较高效，并可避免切割问题。

## 条款24:若所有参数皆需类型转换，请为此采用non-member函数

令C++支持隐式类型转换其实是个不好的建议，但有的时候我们还是需要这样做，因为需可以提供一定的便利。想像这样一个场景，分数类型，其乘法运算符重载，应该是分子*分子，分母\*分母(简化问题就没有约分操作)。但如果这时候是一个整数类型\*一个分数类型的话，基于非explict的构造函数，这个整数会隐式转化为一个分数类型，编译器会采取这样的动作:Rational(整数，1)，第二参数其实是个default值，那么这时候就支持了整数\*分数。但是，如果是这样的情况，将会编译错误:Rational oneself(1,2);Rational result=2\*oneself;(编译错误)，出现这样的情况，原因是编译器在执行这个语句的时候，会如此编译:result=2.operator\*(oneself)，但作为2，是不存在这样的重载运算符的，所以问题就已经出现了，他的重载运算符，是针对Rational类的，对于global区域并没有任何影响，对于整数的转型，其实他发生的情况是只有他作为函数参数的时候才会默认发生，所以我们需要将这个重载运算符的操作定义在类外部，作为一个non-member函数，接收的就是2个Rational的reference类型。这里解决了我个人的一个困惑，为什么这个重载不会影响到本身的\*操作呢？这是这是一个重载类型，针对的是两个特定类型Rational &，所以常规的操作并不会带来变化。那么他定义在类外部，想要调用类的成员变量，回时一个friend函数吗？答案是否定的，因为我们的操作是基于两个reference对象来调用内部的成员数据来实现的，不需要友元这个特性来实现。

## 条款25:考虑写出一个不抛异常的swap函数

swap是支持两个对象相互交换数据的函数，对于模板swap来说，只要类型T支持copying操作，缺省的swap就会帮助我们置换类型为T的对象。swap的交换时普通的，参考条款10，11，12，这样的copying操作在涉及大数据量时，他的copying操作时非常慢的，所以针对这种类，我们可以在类内部做swap的重载，直接交换他的数据成员的指针即可，但是类外部的swap函数也要做重载，针对这个类型的数据swap，即是调用这个引用成员的成员函数。这时候考虑如果是作为类模板的情况下，他如果是以这个形式来进行特化的:template\<typename> void swap< widge\<T> > (widge\<T>& a,widge\<T> &b){}，这样的情况是不合理的，因为我们企图偏特化一个函数模板，是无法通过编译的，所以我们打算偏特化一个函数模板的时候，常用做法应该是template\<typename T>

void swap(widge\<T> &a,widge\<T> &b){a.swap(b);}这样才是可行的。但是如果这是在std命名空间里，编译器会组织这个可能会引起std命名空间发生代码膨胀的行为，所以这时候另一种可行的做法则是在另一个命名空间内执行这个操作。但是对于这个交换的情况，如果说在这个全新命名空间中的特化版本无法实现，那么编译器就不知道需要调用哪一个swap函数了，所以这时候我们需要在其所在类的命名空间内using std::swap来满足swap的最差行为。

- 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
- 如果你提供一个member swap，也该提供一个non-member swap用来调用前者，对于classes，也请特化std::swap。
- 调用swap时应针对std::swap使用using表达式，然后调用swap并且不带任何“命名空间资格修饰”。
- 为“用户定义类型”进行std template全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。

## 条款26:尽可能延后变量定义式的出现时间

如果类定义了一个变量，它具有析构函数与构造函数，那我们需要承担他构造与析构的成本，即使他没有使用。所以例如说我们给一个密码操作，string encpy(string &password){string enctyed;if(password.length()<min){return ...;} return enctyed;}从这个例子可以看出，实际上这个enctyed的string变量，在发生密码不够长的时候，是不会用到的，所以我们可以延后他出现的时间，放在if的后面，并用传入的参数直接做值初始化他，这样就能省去一次可能多余的构造与析构的过程。如果是循环呢？考虑这样的情况:

```c++
①
widge w;
for(int i=0;i<n;++i){
    w=...;
    ....
}
②
for(int i=0;i<n;++i){
    widge w;
    ...;
}
```

这两种赋值方法各自的特点是:①写法需要承担1个构造函数+1个析构函数+n个赋值操作，②做法则是n个构造函数+n个析构函数，当赋值的成本低于一组析构+构造的时候，做法A时更为优秀的，但是做法A容易带来w作用域更大的问题，有的是会带来意想不到的清空问题。个人亲自碰到过的问题有:vector\<int>a,\<vector\<int>>ans; for(int i=0;i<n;++i){a.push_back(i);ans.push_back(a);a.clear()}，这样的动作会导致ans内插入的那个vector也会被清空。现在找到了原因，是因为clear操作清理的是一整块内存区域，它是根据begin指针和end指针来进行操作的。

## 条款27:尽量少做转型动作

转型动作相对来说较为危险，大部分情况下应该去尽量回避掉他，但如果确实需要，可以将转型动作封装在某些函数中，而非显式的出现在主函数代码内。参照接下来的这个例子:class base{}; class derived:public base{};derived d;base* pd=&d;(在这个动作中，就将derived隐喻的转换为了base*)。有时候上述两个指针值并不相同，那么这时候会有个偏移量在运行期间倍适性与derived指针身上，用以取得正确的base\*指针值。这里也间接说明了为什么多态是运行期才会发生的特性了。但这个偏移量也是危险的，因为他在不同平台上，基于内存布局的不同，他们的偏移量也不同，所以这就缺乏了可移植性。

此时有一个特殊场景window base class与special window derived class，假设二者都定义了virtual函数onsize，进一步的，special window的onsize函数被要求首先调用window的onsize函数，那么实现方式有可能是(虽然是错误的，但需要理解为什么错误):class window{public:virtual void onbase(){}};class specialwindow:public window{public:virtual void onsize(){static_cast\<window>(*this).onsize()}};那么这段代码逻辑上也许是认为正确的，因为我们要先执行window的onsize函数，但是实际上并不是，他执行的\*this对象的base class部分的暂时副本的onsize，然后在当前对象上执行special window专属动作，如果window::onsize修改了对象内容，当前对象其实没被修改，改动的是副本，但如果specialwindow::onsize改动了对象，那么对象的内容会被真实的改动。那么这个对象的改动实际上是一个残缺的状态，他只收到了specialwindow改动的信号，但是window改动的信息，丢失了，所以specialwindow中的改动，实际上可以写为window::onsize，显式的指定他由哪部分来进行操作。

涉及到一个最特殊的新型转型动作:dynamic_cast，他是只能用于类指针或引用的上下转型，对于向上转型，不做任何类型检查，对于向下转型，其实就是一个向上转型，能否成功基于他的一个当前指向的静态类型在继承链中与期望类型的上下位置。对于一个继承深度很深的类型，其内部还有string类型这样的变量的时候，那么他的转型将会做大量的strcmp操作，这样是很浪费时间的。当然，解决方法提供了两种:

1. 使用容器并在其中存储直接指向derived class的指针，通常是智能指针。对于这个问题的实现手法通常为申请一个vector，内部起初存放的是base class的指针，在放入一个新的元素时，使用base class的指针去指向他。但这里有一个问题就是，如果只有derived class才支持这样的功能，那务必要将容器内存储的类型改为derived class的类型。
2. 通过base class接口处理所有可能之各种window派生类，那就是在base class内提供virtual函数做你想对各个derived class做的事。例如上述所说的如果只有derived class才支持这样的功能，那么这时候实际上可以参照上面的手法，通常为申请一个vector，内部起初存放的是base class的指针，在放入一个新的元素时，使用base class的指针去指向他。这样即可通过虚机制来解决这个问题。

- 如果可以，避免转型。
- 如果转型是必要的，试着将他隐藏于某个函数背后。
- 最好是使用新式转型。

## 条款28:避免返回handles指向对象内部成分

例如有这样的一个点类，点类的数据成员是x，y，而在矩形内部，则是指向这样一个封装了左上角，右下角的矩形标识点类型的智能指针。在这个类内，定义了形如这样的两个返回reference的查询函数，point& upperleft() const {return pData->ulhc;}，编译是不存在问题的，但是真实使用情况会带来问题，考虑这样一种情况:const rectangle rec(point1,point2); rec.upperleft().setx(50);可以注意到，本身rec是一个const类型，是不允许进行修改的，但是在实际应用中，他能被查询点返回的引用来进行修改，这就引申出了一些问题:第一，成员变量的封装性最多值等于“返回其reference”的函数的访问级别。第二，如果const函数传出了一个reference，后者所指数据于对象自身有关联，而它又被存储在对象之外，那么这个函数调用者可以修改那笔数据。对于返回handle(引用，指针，迭代器这一类标识身份的东西)，即是返回值，也要格外小心。偶尔也会出现这样的错误情况:const point* p=&(bound(&pgo).upperleft())；这一次upperleft的返回值为一个value，但是这段代码还是存在问题，bound(&pgo).upperleft()是一个临时对象，脱离了这句语句他就会消亡，而此时的p指向了这块区域，在脱离了这句语句之后，p将指向一块垃圾内存。

- 避免返回handle指向对象内部，遵守这个条款可增加封装性。

## 条款29:为“异常安全”而努力是值得的。

异常安全有两个条件，当异常被抛出时，带有异常安全性的函数会:

1. 不泄露任何资源:例如说我们在函数内申请了一个堆空间，在return前我们会将它进行delete，但这时候在delete语句之前发生了异常，异常安全机制需要保证这个堆空间依旧能被正确释放。
2. 不允许数据败坏:如若我们申请一个堆空间之前，先对申请操作的记录次数进行+1，但这时候申请操作发生了异常，那么实际上空间并没有申请成功，但是记录数已经增加了，这就是数据败坏。

解决内存泄露的问题相对简单，我们可以引入有一个class来管理这个资源，他在脱离作用域时会自动析构。

对于数据败坏，首先异常安全函数提供三个保证之一:

1. 基本承诺:如果异常被抛出，程序内的任何对象仍然保持在有效状态下，没有任何对象或数据结构会因此遭到破坏，所有对象都处于一种内部前后一致的状态。
2. 强烈保证:如果异常抛出，程序状态不改变，调用这种函数，如果函数成功，就是完全成功，如果失败，程序会恢复到调用函数之前的状态。
3. 不抛掷保证:承诺绝对不抛掷异常，因为他们可以完成他们原先承诺的功能。假设函数带着“空白的异常明细”者必为nothrow函数，这种说法不尽然，例如 int do() throw(); 这里就不是说do绝不会抛异常，而是do如果抛出异常，将会是严重错误，会有意想不到的函数调用。

对于异常安全码，一般不抛掷保证是难以是实现的，所以我们会考虑强烈保证。想要提供强烈保证，我们可以设计这样的一个函数体，第一，针对内存泄露问题，类成员接收的指针存放到智能指针类型；第二，我们可以重新排列函数体内的语句顺序，避免一些操作提前发生，例如计数器需要放再申请空间的语句之后。

当前，借用智能指针对象，有可能在这个对象构造传入new时抛出异常，所以上述方案还只是最简单的异常保证。我们可以考虑一个更为巧妙的手法，copy and swap，我们先将需要操作的资源存放到一个临时对象之中，然后对他进行操作，如果他抛出异常，不影响函数本身内部资源的状态，但没有抛出异常，我们可以做资源的swap，交换指针，将这块临时变量置空删除即可。

但是存在这样的一种函数，他调用了其他的子函数，例如void somefunc(){a();b();....}，针对这种情况，它并不能做为有异常保证的函数，因为即使a是强烈异常保证，执行成功，但不能说b就一定能够执行成功，一旦b执行抛出异常，somefuc依旧需要有对应的异常处理机制，所以异常保证是具有连带性的，不可简单认为。当强烈保证都无法达成的时候，就需要考虑基本承诺来解决异常问题。

- 异常安全函数即使发生异常也不会发生内存泄漏与数据败坏，这样的函数区分为三种可能的保证:1.基本保证2.强烈保证3.不抛掷承诺。
- “强烈保证”往往能够以copy-and-swap来表现出来，但强烈保证并非对所有函数都可实现或具备意义。
- 函数提供的异常安全保证通常最高只等于其所调用的各个函数的异常安全保证的最弱者。

## 条款30:透彻了解inlining的里里外外

inline(内联)函数，作用在编译时期将函数体与函数调用处以代码形式展开，省去函数调用开销，在一定程度上可以提高效率。但是相对的，此类行为也会膨胀代码体积，过大的代码体积会导致额外的换页行为(虚拟内存缺页置换)，降低指令高速缓存装置的命中率。

inline只是一个对编译器的建议，编译器会考量是否需要将这个函数转化为inline，可以先生在函数名前声明inline，也可以在class内定义函数隐喻的向编译器建议inline。大部分的inline是在编译器就完成动作的，但模板具现化与inline无关。当我们写一个模板函数时，如果认为他的动作应该inline，就要把这个模板声明为inline，如果没有，就不要加上inline。

有时候编译器有意愿inline某个函数，还是可能为该函数生成一个函数本体。如果程序要取某个inline函数的地址，编译器通常必须为此函数生成一个outline函数本体，毕竟编译器没有能力提出一个指针指向并不存在的函数。当然，要注意，编译器通常不对通过函数指针调用的函数实施inlining。

事实上构造函数和析构函数往往不适合做inline函数，因为例如一个例子class base{private: string a,b;}; class derived:public base{pubilc:derived(){} private:string str1,str2,str3;};如果这个derived构造函数确实为inline，并且string的构造函数也为inline的话，其构造将会展开5个代码模块，膨胀大小无法估量。

- 将大多数inlining限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升最大化。

##  条款32:确定public继承塑膜出is-a关系

is-a关系往往使用public继承来实现，因为is-a的关系表示derived类是一个base类，那么可以说明derived可以继承base的一切动作。但是这时候考虑一个情形:例如企鹅是鸟类，但是鸟类是会飞的。企鹅public继承鸟类，那么是意味着企鹅会飞吗？答案应该是否定的，但在继承中就体现出了这样的特点。我们可以想象的方案就是分离这两个特征，一个基础鸟类，一个可以飞的鸟类，还有一种不会飞的鸟类。可是这样的问题存在许多，我们无法设计那么多的类去解决这些问题。

- public继承意味着is-a，适用于base class身上的动作也一定是用与derived class身上，所以在设计继承体系的时候需要考虑到父类子类的相同动作。

## 条款33:避免遮掩继承而来的名称

本条款主要说明的就是局部作用域同名函数或变量冲突的问题，当我们在函数内定义一个double x，而global区域中已经存在了一个int x，这时候我们如果在函数内cin>>x的话，cin读取到的其实是double x，这里发生了变量遮掩的情况。在继承体系中，也常常发生这种情况。例如derived class中由func1，func2，而base class中也有这两个函数，甚至于还有这两个函数的重载，那么实际上，由derived class 对象上调用的func1，只会是derived class上的这个func1(non-virtual函数的情况下)，甚至于重载版本都被屏蔽了，无法使用。如果我们想要在derived class强行使用base class的fun1版本，要在derived class中声明using base::func1。

因为使用using会使得func1的重载版本同样可见，所以对于derived class如果只想使用base class的func1的基础版本，这时候就可以构建一个转交函数，形式为virtual void func1{base::func1();}如果是想要有参版本则内部修改为base::func1(int x)即可。

- derived class内的名称会遮掩base class内的名称，在public继承中没有人会希望如此。
- 为了使用被遮掩的接口，可使用using声明式或转交函数。

## 条款34:区分接口继承和实现继承

继承体系中的函数一般可以分为3种:

1. 一个pure virtual(纯虚)函数
2. 一个impure virtual函数(朴素虚)函数。
3. 一个非虚函数。

这样规定的情况主要是为了区分派生类的成员函数是要怎么继承基类的成员函数，有的时候我们仅仅想要继承函数接口，又或者同时继承接口和实现，又或者希望能够重写他，又或是不允许重写，这些都是不同的做法。成员函数的接口总是被继承。

对于pure virtual(纯虚)函数来说，他仅继承函数接口，但不继承函数定义。但如果我们需要调用基类的纯虚函数定义，我们需要使用时需要明确指出其class名称。

一个impure virtual函数(朴素虚)函数是为了让派生类继承该函数接口和缺省实现。当然，这也存在一定的风险，假设这样一个情形:A，B继承了基类的fly函数，并都是以缺省形式存在的，但这时候存在一个C类，他也继承了基类，但他fly方式应该是与基类的缺省方式不同，但他忘记重写了，这时候就会带来问题。所以我们可以对fly定义为一个纯虚函数，而A，B的调用则调用另一个non-virtual函数。这个non-virtual在派生类完全不需要重新定义，所以定义为non-virtual。那么对于C来说，他必须重写fly函数才能实现自己特殊的飞行功能。

对于上述方案的优化版本可以完全抛开defalut fly函数，因为我们可以在基类的fly函数中提供函数定义，我们需要调用则是直接明确指出其class名称，就能实现调用基类中的纯虚函数中的定义。声明non-virtual函数的目的是为了令派生类继承函数的接口及一份强制性实现，说明的是任何派生类都不应该改变其行为，由于non-virtual函数代表的意义是不变性凌驾特异性，所以他不被派生类所改变。

- 接口继承和实现继承不同，在public继承之下，派生类总是继承基类的接口。
- 纯虚函数只继承具体指定接口函数。
- 普通虚函数具体指定接口继承及缺省实现继承。
- non-virtual函数具体指定接口继承以及强制性实现继承。

## 条款35:考虑virtual函数以外的其他选择

让virtual函数的定义不再那么明显是本条款的目标，所以有以下3种方式可以解决:

1. 由 base class的non-virtual函数去调用private之下的virtual函数。这样的实现，是为了可以在调用这个虚函数之前，先布置好对应场景，然后调用虚函数，再清楚场景，这样可以让虚函数的调用，保持在一个合适的场景之中，当然，derived class也可以重新定义虚函数的内容，但base class保留函数何时调用的权利。这里注意，我们是不让外部看到这个virtual函数的定义的，所以其一般是private访问级的，但是事实使用并非如此，往往会需要将其设置为public或protect的场景，这就违背了NVI的手法(通过non-virtual函数间接调用private virtual函数)了。
2. 更为特别的做法，自然是可以主张我们想要实现这个虚函数的效果，与类本身并无关系，那么这时候可以借由函数指针来进行实现，即类内typedef 一个函数指针，在构造函数时传入外部的一个非member function，来作为函数指针的指向，当我们需要调用这个函数的时候，直接使用函数指针进行调用即可。这样做的方式是可以使得同一类型的不同实体可以有不同的计算函数，也可以使得某个对象的这个计算函数在运行期可以变更。
3. 函数指针的方式带来的问题是，他的计算函数，只支持访问public成员来进行计算，没有办法调用更低隔离级的成员，并且函数指针只接受来自函数本身的定义，因此第三种方式则是通过C++11中的tr1::function来进行操作，使得计算函数更具有弹性。在成员内的typedef可以变更为std::tr1::function<int (cosnt class_name&)>这样的形式来实现，这样我们接收外部函数，仿函数以及绑定(bind)到成员函数身上时，都会自动转换为int (class_name&)的形式，形成函数调用，拓展了弹性，以及加强了函数的可访问性。
4. 最后一种设计模式，即外部有一个计算函数的类，它可以被继承用以修改，而需要这个计算函数的类，可以内嵌一个他的指针类型的成员，这样也可以实现类似的效果。

- virtual函数的替代方案包括NVI手法及Strategy设计模式的多种形式
- 将机能从成员函数移到class外部函数，带来的问题就是非成员函数无法访问class的non-public成员
- tr1::function对象的行为就像一般的函数指针，这样的对象可接纳“与给定目标签名式兼容”的所有可调用物。

## 条款36:绝不重新定义继承而来的non-virtual函数

当我们使用derived class的成员函数时，这两种情况要加以考虑

```c++
①
D x;
B* pB=&x;
pB->mf();
②
D* pD=&x;
pD->mf();
```

如果他们调用的是non-virtual函数，那么他们调用的，是各自静态类型的版本，即pB调用的是B中的mf，而pD调用的是D中的mf，如果是virtual函数，则调用的都是D类中的mf，因为虚函数是动态绑定的，调用的是指针指向的具体类型。当然，本条款着重是为了解释，对于派生类中重新定义non-virtual函数，会发生函数二义性问题，你会根据你的静态类型去做函数调用，这样就违背了继承的is-a原则，因为B能做的，D也都能做，那么出现的问题就是，这时候D做的，与B就完全不一样了，这是有悖于原则的。

- 绝对不要重新定义继承而来的non-virtual函数。

## 条款37:绝不重新定义继承而来的函数参数缺省值

动态绑定机制会改变程序运行时变量的类型，从而实现虚机制，因此在他进行动态绑定时，其实调用的虚函数的缺省值，还是base class的缺省值，会是这样的原因是:运行期效率的问题，如果缺省值是动态绑定的，编译器就必须有某种办法在运行期为virtual函数决定适当的参数缺省值，这比在编译期决定的机制更慢并且更为复杂，因为考虑3层继承的关系，你的函数缺省值，应该是第一层继承，还是第二层继承，这都是有待讨论的。为了执行速度和编译简化，故做base class的缺省值也为动态绑定的缺省值。但如果derived class中的缺省值与base class相同，那更不是一个好的设计，代码重复又带着相依性，如果基类中的改变了，会连带着派生类一起变化，因此我们又可以是行NVI手法，将这个缺省值交由non-virtual函数来进行管理，在其内部调用virtual函数。

- 绝对不要重新定义一个继承而来的缺省参数值，因为缺省参数值都是静态绑定的，而virtual函数则是动态绑定

## 条款38:通过复合塑膜出has-a或'根据某物实现出'

复合是一个包含关系，表示在B内存在一个A对象，他与B内的其他成员变量处于同一等级，而非是B的核心的意思。假设这时候我们想要重新实现一个set，那我们可以基于list来进行实现，但是存在一个问题，我们如果将class  set:public list\<T>，这样并不能达到我们的目标。因为list可以有重复元素，而set不行，但是set又可以使用list来实现，这时候我们就能将list设置为set的成员变量来进行实现。

- 复合的意义与public继承完全不同

## 条款39:明智而审慎地使用private继承

public继承会使得函数参数在必要时刻将derived class转化为base class使得函数调用可以成功，而private继承是做不到这一点的。private继承的意义是只有实现部分被继承，接口被略去。其意义其实与复合很相近，所以能使用复合的情况下尽量使用复合，而避免private继承。但是存在这样一个情况，当一个对象在不同阶段拥有不同的行为轮廓，那么我们应该设计一个计时器类timer，而这个timer应该被这个变化对象的类所继承，这样会带来一个问题，这个对象并不是一个timer，所以继承这个is-a的关系被打破，我们只能考虑复合联系。在这个变化类的内部，我们可以封装一个widgetimer类，private继承自timer，并在内部对timer类的方法进行重写，并且在这个变化类内部，添上一个这样的一个继承timer类，这样可以使得这个变化类在之后出现继承情况时，派生类无法去重写这个继承timer类中的计时方法，并且可以将变化类与timer类的编译依存性降到最低。当然，private最常用的地方，还是在继承空类的时候可以加以使用。例如说base类中仅有typedef与member function，而derived class恰巧都需要这些东西，这时候如果复合，那么derived class是要多占额外的空间，但如果是private继承，则可以不用消耗额外的空间，并且继承这些typedef和函数接口。

- private继承意味着根据某物实现出某些东西，他通常比复合的级别低，但是当derived class需要访问保护型基类成员，或重新定义继承而来的virtual函数时，这么设计时合理的。
- 和复合不同的是，private继承可以优化empty base带来的开销。

## 条款40:明智而审慎地使用多重继承

多重继承最长出现的问题即是菱形继承的情况，这种情况并不是一种好的选择，原因如下:1.会造成数据副本拷贝的浪费，需要复制多份副本2.最关键的，在出现两个类中具有同名函数时，会发生函数二义性的问题。为了解决菱形继承，可以使用虚基类的形式，当继承前加上virtual关键字，表示将当前继承基类作为一个虚基类，其派生类成员共享其成员变量，函数副本，仅有一份。但是存在一个问题，如果使用虚基类，那么会导致类体积会略微变大，访问虚基类成员的速度要慢于非虚基类成员，总之，就是要付出一定的代价。对于virtual继承，非必要时不要使用，但如果一定要使用时，避免在内部放入数据。

当然，多重继承的应用场景并不是没有。例如当前我们具有一个抽象类person，其内部是多个纯虚函数，这时候我们只能操作其指针或引用来对这个类进行使用，但如果这时候我希望能够去具象化的使用他，那我们就需要有一个承载他的媒介。所以这时候我们可以引申出一个新的派生类cperson来public继承自person，之后需要对其虚函数进行改写。但如果这时候有一个库，恰巧可以提供person内所有纯虚函数的动作，那我们就不需要自己写了，这是一件非常方便的事。但如何使得二者产生联系呢？就可以让cperson public继承person，private继承personinfo这个方法类，那么我们可以通过在cperson中重写纯虚函数，在纯虚函数内部调用personinfo内的动作，就可以实现一个全新的包装，使得cperson可以具现化person进行使用，并且person的指针或引用在指向cperson对象时，也能够具有他的动作。

- 多重继承会导致歧义性，并且需要多承担virtual继承的开销。
- virtual继承会增加大小、速度、初始化复杂度等成本，如果virtual base class内不含数据，是最好情况
- 多重继承也有正确用途，设计public继承一个接口，private继承某个协助实现的class

## 条款41:了解隐式接口和编译期多态

```c++
template<typename T>
void doprocessing(T& w){
    if(w.size()>10&&w!=someNastyWiget){
        T temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```

- 此时的w必须支持某一些接口，系由template中的w身上的操作来决定，从上述例子来说，它需要支持size，normaliz，swap，拷贝构造函数，不等比较操作符，显然，这一组表达式便是T必须支持的一组隐式接口。
- 只要是涉及到w的操作，，例如operator>和operator !=，都有可能造成template具现化，才能使得这些操作符调用成功，这种具现化行为出现在编译期，以不同参数具现化会导致调用不同的函数(这里的个人理解是，如果说这个比较函数也是个模板，并且有特化情况，那么调用的函数自然不同)，这就是编译期多态。

显式接口往往就是函数的签名式(函数名称、参数类型、返回类型)构成。隐式接口则是由有效表达式组成。例如例子中的if(w.size()>10&&w!=someNastyWiget),size()返回值不一定需要是一个int类型，也可以是非explicit构造函数对10进行隐式转化，调用operator>来进行使用，它具有无限的可能性。

加诸于template参数身上的隐式接口，与显示接口一样真实存在，并且二者都在编译期完成检查，你无法以一种与显示接口矛盾的方式来使用对象，也无法在template中使用不支持template所要求之隐式接口的对象。

- classes和template都支持接口和多态。
- 对classes而言接口是显示的，以函数签名为中心，多态则是通过virtual函数发生在运行期。
- 对template而言，接口是隐式的，基于有效表达式，多态则是通过template具现化与函数重载解析发生于编译期。

## 条款42:了解typename的双重含义

在template声明式中，typename和class并无什么区别，都是根据个人习惯来进行编写，但是typename具有其他的意义。

假设当前我们具有一个template function，接受一个STL兼容容器为参数，容器内持有的对象可以赋值为ints，给出以下例子

```c++
template<typename C>
void print2nd(const C& container){
    if(container.size()>2){
        //C::const_iterator* x;
        C::const_iterator iter(container.begin());
        ++iter;
        int value=*iter;
        std::cout<<value;
    }
}
```

这段例子其实是一段错误的代码，在模板内部，出现的某个变量名称如果依附于某个template参数，称之为从属名称，如果从属名称在class内呈嵌套状，则成为嵌套从属名称，C::const_iterator就是一个典型例子，嵌套从属名称带来的问题就是会导致解析困难，例如说C::const_iterator* x;，我们想要声明的意思其实是一个指针类型的x，但如果C::const_iterator并不是一个类型，那这个表达式的解析可能就会变成相乘的动作。我们在知道C是什么之前，是没有任何办法去知道C::container是否是一个类型，因此为了避免这种情况，编译器给出了自己的解决方案:如果解析器在template遭遇了一个嵌套从属名称，它便假设这不是一个类型，除非我们告诉他是。所以缺省情况下嵌套从属名称不是类型。代码可以修改为

```c++
template<typename C>
void print2nd(const C& container){
    if(container.size()>2){
        //C::const_iterator* x;
        typename C::const_iterator iter(container.begin());
        ++iter;
        int value=*iter;
        std::cout<<value;
    }
}
template<typename C>//可以换成class
void f(C& container,typename C::iterator iter)//第一参数不允许使用typename，第二参数必须使用typename
```

当然，typename必须作为嵌套从属名称的前缀词这个规则也存在例外，就是typename不能出现在base classes list内的嵌套从属名称之前，也不可以在成员初值列中作为base class修饰符。例如

```c++
template<typename T>
class Derived:public Base<T>::Nested{//base class list中不允许出现typename
public:
    explicit Derived(int x):Base<T>::Nested(x){//成员初值列不允许出现
        typename Base<T>::Nested temp;//都不在的情况下，需要typename标名为嵌套从属名称
    }
}
```

当然，还有一个更为新颖的例子，说明了stl源码中那么多特别的编程习惯

```c++
template<typename iterT>
void workwithiterator(iterT iter){
    ①typename std::iterator_traits<iterT>::value_type temp(*iter);
    ②typedef typename std::iterator_traits<iterT>::value_type value_type;
    value_type temp(*iter);
}
```

这段代码的意思就是要取出这个iter的类型指定为temp的类型，并将temp将iter作为参数进行拷贝构造。很明显，这个写法十分冗余，所谓我们可以写成②的形式，可以在之后也形成复用。

- 声明template参数时，typename与class等价
- 请使用typename标识嵌套从属类型名称，但不得在base class lists或成员初值列内以它作为base class修饰符

## 条款43:学习处理模板化基类内的名称

参照这样一个例子

```c++
template<typename company>
class msgsender{
public:
    ...
    void sendclear(const msginfo& info){
        std::string msg;
        company c;
        c.sendcleartext(msg);
    }
    void sendsecret(const company& info){
        ...
    }
};
template<typename company>
class loggingmsgsender:public msgsender<company>{
public:
    ...
    void sendclearmsg(const msginfo& info){
        //搭建环境
        sendclear(info);//调用base class函数，但是无法通过编译
        //撤销环境
    }
};
```

当编译器遭遇loggingmsgsender时，并不知道他继承的什么样的class，因为company这是不可知。假设这时有一个模板全特化的类:

```c++
class conmanyz{
public:
    ...
    void sendencrypted(const string &msg){
        
    }
};
template<>
class msgmsender<companyz>{//这个全特化特别之处在于删除了shaderclear
public:
    ...
    void sendsecret(const msginfo& info){
        
    }
    ...
};
```

针对这个全特化情况，那么loggingmsgsender这个模板类内的sendclear(info)就是一个错误的调用了，故它编译是不通过的。因为它知道base class templates有可能被特化，而那个特化版本可能不提供和一般性template相同的接口。因此它往往拒绝在templatized base class内寻找继承而来的名称。

但是存在三种办法令c++“不进入templatized base class观察”的行为失效。

1. 在base class函数调用行为之前加上this->
2. 使用using关键字，表明他在哪里出现过
3. 显式指出被调用函数在base class中，但这个做法会出现问题，如果调用的函数是虚函数，上述明确资格修饰会关闭“virtual绑定行为”。

面对涉及到基类成员引用的时候，编译器的诊断时间可能发生在早期，也可能在晚期，我们希望实现的，则是尽早诊断，所以c++类的template derived class会对template base class的内容一无所知。

- 可在derived class template内通过“this->”指涉base class templates内的成员名称，或由一个明白写出的“base class资格修饰符”完成。

## 条款44:将于参数无关的代码抽离templates

使用templates可能会导致代码膨胀，所以我们要尽可能的去减小这种情况带来的开销。当我们编写某个函数的时候，如果他们具有相同的功能，并且目标码相同，自然的我们会想着去把他单独抽离出来作为一个独立函数进行调用，当我们编写templates时参考同样的方法避免重复

```c++
template<typename T,std::size_t n>
class SquarMatrix{
public:
    ...
    void invert();
};
```

假设以这个作为一个模板基类，生成SquarMatrix<double,5>与SquarMatrix<double,10>对象，都调用invert函数，实际上我们可以发现，invert除了操作的矩阵与尺寸n关联之外，其余操作并没有任何区别。所以这时候我们建立一个新类

```c++
template<typename T>
class SquareMatrixBase{
protected:
    ...
    void invert(std::size_t n);
    ...
private:
    std::size_t size;
    T* data;
};
template<typename T,std::size_t n>
class SquarMatrix:private SquareMatrixBase<T>{//私有继承表示单单继承接口
private:
    using SquareMatrixBase<T>::invert;//保证基类的invert函数在派生类中可访问，而不是函数屏蔽
public:
    ...
    void invert(){this->invert(n)};
private:
    T data[n*n];
};
//又或是
template<typename T,std::size_t n>
class SquarMatrix:private SquareMatrixBase<T>{//私有继承表示单单继承接口
private:
    using SquareMatrixBase<T>::invert;//保证基类的invert函数在派生类中可访问，而不是函数屏蔽
public:
    SquarMatrix:SquarMatrixBase<T>(n,0),pData(new T[n*n]){this->setDataPtr(pData.get());}
    void invert(){this->invert(n)};
    ...
private:
    boost::scoped_array<T> pData;
};   
```

这样独立化一个类之后，调用的所有invert只会剩下一个版本，即SquareMatrixBase的invert版本，并且调用基类的invert的成本几乎为0，因为他是inline调用。

这时候又留下了一个问题，我们怎么知道要操作的是哪些数据呢？这些数据应该只有derived class会知道吧。所以这时候，我们需要对SquareMatrixBase安插一个指向数据的指针。

不同大小的矩阵只拥有单一版本的invert，可减少执行文件大小，也就因此降低程序的进程内存页大小，并强化指令高速缓冲区的引用集中化，这些都可能使程序运行得更快速，超越“尺寸专属版”invert的最优化效果。当然，也存在一些问题，我们往基类引入这个指针之后，其数据还是贮存在derived class上，那么指针该不该删除也成了一个困扰的问题。

- templates生成多个classes和多个函数，所以任何template代码都不该与某个造成膨胀的template参数产生相依关系
- 因非类型模板参数而造成的代码膨胀，往往可消除，做法是以函数参数或class成员变量替换成template参数
- 因类型参数而造成的代码膨胀，往往可降低，做法是让带有完全相同二进制表述的具现类型共享实现码

## 条款45:运用成员函数模板接受所有兼容类型

对于一个template来说，我们无法确定他的一个转型动作，例如说samrtpoint，他是一个像指针的类，当然指针支持相互的转型，例如派生类能够转化为基类，而在template中，并不是这样的一个形式，同一个template的不同具现体之间并不存在什么固有关系，所以编译器会视基类与派生类为不同的class，二者没有任何联系，自然没有转型的行为，因此我们如果想要将他进行转型，可以在其模板内部，加入模板函数来实现，接下来的这段代码就是模板与泛型编程之间的关系:

```c++
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
};
```

以上代码的意思是任意类型的T与U，可以根据SmartPtr\<U>生成一个SmartPtr\<T>，因为SmartPtr\<T>有个构造函数接收SmartPtr\<U>对象，这一类构造函数可以基于对象u生成对象t，而u与v的类型是同一个template的具现体，我们称其为泛化拷贝函数。

当然，对应的也有一些问题，即是转型的有效性问题，例如说int*不能转化为double\*，基类指针不能直接转化为派生类指针，所以这时候我们可以采取以下动作

```c++
template<typename T>
class SmartPtr{
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other)
    :heldPtr(other.get()){...}
    T* get() const {return heldPtr;}
    ...
private:
    T* heldPtr;
};
```

这样的写法，能保证存在某个隐式转换可将U*指针转化为T\*指针时才能通过编译。这是我们拥有了一个泛化的构造函数，只在其获得的实参隶属适当类型时才通过编译。

成员函数模板并不仅限于构造函数，很多时候也用于赋值，拷贝操作。例如tr1中的shared_ptr

```c++
template<typename T>
class shared_ptr{
public:
    template<class Y>
        explicit shared_ptr(Y* p);
    template<class Y>
        shared_ptr(shared_ptr<Y> const& r);
    template<class Y>
        explicit shared_ptr(weak_ptr<Y> const& r);
    template<class Y>
        explicit shared_ptr(auto_ptr<Y>& r);
};
```

上述构造函数均是explicit，唯有泛化构造函数除外，意味着从一个shared_ptr指针转化到另一个shared_ptr指针式允许的，但从内置指针或从其他智能指针类型进行隐式转换并不被认可。

成员模板函数虽然奇妙，但他不改变语法规则，在条款5说过，编译器会为我们生成默认的4个成员函数，所以考虑T和Y相同的时候，泛化拷贝函数会被具现化为“正常的”拷贝构造函数。因为语法规则不变，所以编译器会在你没有声明我们需要的构造函数的时候，会暗自生成一个。所以如果我们想要把控拷贝构造函数的方方面面，那就必须

同时声明泛化拷贝函数和“正常的”拷贝构造函数

- 请使用成员函数模板生成“可接受所有兼容类型”的函数
- 声明成员模板用于“泛化拷贝构造”或“泛化赋值操作”，还需要声明正常的拷贝构造函数与拷贝赋值函数

## 条款46:需要类型转换时请为模板定义非成员函数

例如在条款24中出现的Rational类的operator*操作，它需要在外部定义这个opertor\*操作，那么在模板中，他还能做此体现吗？我们知道函数是需要提供类型的，而这个类型推导在外部是不得而知的，并且根据条款24所说，若一个整性变量\*这个模板类，我们想要得到的结果应该是模板类，但是实际上整型变量并没有重载\*这个符号，所以在无法得知类型，又要保证类型转换正确且可支持转换的情况下，我们需要将这个operator\*定义为友元，定义在模板内部，让他能够支持\*操作能够自动转化\*前后的两个变量为模板内的类型，并且在外部可以被识别到并进行使用，friend关键字修饰的函数即在外部被当作普通函数，就可以进行使用。特别的，咱模板内部，就需要给出函数定义，这是必要的。

- 当我们编写一个类模板，当它所提供之“与此template相关的”函数支持“所有参数之隐式类型转换”时，请将那些函数定义为“class template内部的friend函数“

## 条款47:请使用traits classes表现类型信息

本条款我认为是一条设计理念，只是其广泛应用在STL库中。考虑STL中的5中迭代器:

1. input迭代器，也称为读迭代器，只能向前移动，一次一步
2. ouput迭代器，也成为写迭代器，只能向前移动，一次一步
3. forward迭代器，也称为单向迭代器，可读可写，但只能向前移动，一次一步，继承至input迭代器
4. bidirectional迭代器，也称为双向迭代器，支持双向移动，但一次也只能走一步，继承至单向迭代器
5. random_access迭代器，也成为随机访问迭代器，支持随机访问，指定步数移动，继承至双向迭代器

所以在STL中，为了应对各种迭代器，需要使用traits技术来进行迭代器分类的筛选，来决定对应的操作，很明显，取类型并不是一件简单的事，所以在迭代器内，通常要求这样写

```c++
template<...>
class deque{
public:
    class iterator{
      typedef random_access_iterator_tag iterator_category;
    	...
    };
    ...
};
template<typename IterT>
struct iterator_traits{
  typedef typename IterT::iterator_category iterator_category;  
};
```

这样的写法针对于自定义类型，内部的iterator_category是作为嵌套从属名称，在iterator_traits我们传入这个容器，我们就能去访问其嵌套从属的类别

当然，这也存在一个问题，原生指针并不带有这个参数，因此我们需要对原生指针进行偏特化，特别针对指针类型迭代器提出一种trains的方法，这样问题得以解决。这仍旧并非是我们想要的，IterT的类型在编译期就可得知，所以iterator_traits\<IterT>::iterator_category在编译期也能确定，但是if核定迭代器需要在运行期才能确定，这样不仅浪费时间，也可能会导致可执行文件膨胀。那么我们可以考虑将核定工作一起融入到编译期去做

如果我们想要尽早去核验编译期核定成功之类型，有一个取得这种行为的办法，就是重载。所以这时候编写多个模板，名字相同，但是函数内部的默认参数不同，为各自的迭代器类型，并且函数体内的动作根据迭代器类型来决定。这时候继承的好处就来了，random_access迭代器在没有找到自己的专属重载的时候，会查找父类bidirectional迭代器的专属版本，所以他最后一定是会落在最佳匹配的函数上。有了这些函数的重载版本，它需要做的只是调用它们并额外传递一个对象，后者必须带有适当的迭代器分类

```c++
template<typename IterT,typename DistT>
void doAdvance(Iter &iter,DistT d,std::random_access_iterator_tag){
    iter+=d;
}
template<typename IterT,typename DistT>
void advance(Iter &iter,DistT d){
    doAdvance(iter,d,typename::std::iterator_traits<IterT>::iterator_category());
}
```

可以总结一下如何使用traits class了:

- 建立一组重载函数或函数模板，彼此间的差异只在各自的traits参数，令每个函数实现码与其接受之traits信息相应和
- 建立一个控制函数或函数模板，他调用上述函数并传递traits class所提供的信息

需要记住的:

- Traits classes使得“类型相关信息”在编译期可用，他们以templates和“templates特化”完成实现
- 整合重载技术后，traits class技术有可能在编译器对类型执行if...else测试

## 条款49:了解new-handler的行为

## 条款50:了解new和delete的合理替换时机

想要替换new和delete的原因主要有3类:

1. 用来检测运用上的错误:如果将new所得内存delete掉却不幸失败，会导致内存泄漏，如果对new所得内存身上多次delete则会导致不确定行为，如果operator new持有一串动态分配所得地址，而operator delete将地址从中移走，倒很容易检测出上述错误用法。此外编程错误可能会导致数据写入点在分配区块尾端之后或在分配块起点之前。自行定义的operator new可以超额分配内存，以额外空间放置特定的签名。operator delete得以检查上述签名是否原封不动，若不是就表示在分配区的某个生命时间点发生了数据写入点错误的问题，那么operator delete可以记录这个事情以及那个出问题的指针
2. 为了强化效能:编译器自带的operator new和operator delete主要用于一般性功能，必须接受各种分配形态，这会导致程序最终无法满足大内存块内存要求，即是彼时发现总量足够但分散为许多小区块的自由内存。所以应对这种中庸的分配情况，很多时候可以针对程序特定的功能对operator new与delete进行定制操作，重新设计内存分配方案，可以大量减少时间及内存开销
3. 为了收集使用上的统计数据:我们可以收集你的软件如何使用动态内存，分配区块的大小分布如何？寿命分布如何？他们倾向于先进先出次序还是后进先出的次序还是随机次序？他们的运用形态是否随时间改变，也就是说你的软件在不同的执行阶段有不同的分配/归还形态？任何时刻所使用的最大动态分配量是多少？自行定义operator new我们就能很容易得知这些信息。

关于operator new的制定，涉及到一个比较微妙的问题，就是齐位问题。许多计算机体系结构要求特定的类型必须在特定的内存地址上，例如它可能会要求指针的地址必须是4的倍数，或double必须是8的倍数，如果没有按照这个约束条件，可能会出现运行期异常

齐位意义重大，因为C++要求所有operator new返回的指针都有适当的对齐，malloc就是这样工作的，所以令operator new返回一个malloc的指针是安全的，然而自己编写的operator new并不一定返回一个来自malloc的指针，而是返回一个得自malloc且偏移int大小的指针，没人能保证安全。如果客户端调用这个operator new企图获取一个足够给一个double所用的内存，而我们在一部int为4byte且double为8byte的机器上，我们可能会获得一个未有适当齐位的指针，那可能回导致程序崩溃或者变慢。

所以这时候的operator new的重写目的，可以归类为:

1. 为了检测应用错误
2. 为了收集动态分配内存之使用统计信息
3. 为了增加分配和归还的速度:专属class可以针对特定对象，分配特定的内存块来进行内存管理
4. 为了降低缺省内存管理器带来的内存额外开销:因为malloc每次分配都要带上下8字节的cookie，这时候我们可以一次分配一大块内存，用以节省cookie开销。
5. 为了弥补缺省分配器中的非最佳齐位:因为自定义数据类型的字节大小不一定与内置类型相同，所以我们可以做齐位管理
6. 为了将相关对象成簇集中:同类型，关联性的对象，往往放在相关联的内存位置可以加速访问，而非间接寻址
7. 为了获得非传统行为:可以针对个人数据做安全性保障。

## 条款51:编写new和delete时需固守常规

operator new必须返回正确的值，内存不足时必得调用new-handling函数，必须有对付零内存的准备，还需避免不慎掩盖正常形式的new(这点会出现在global函数中)，operator new的返回值并不简单，不仅仅是分配内存，成功就返回指针，失败就抛出bad_alloc异常，实际上他会不止一次尝试分配内存，因为new-handling函数有可能会释放掉一些空间。所以只有当new-handling函数的指针式null，operator new才会抛出异常

并且奇怪的是，operator new应对客户提出0byte这种奇怪的分配，也得返回一个合法指针。例如这段operator new源码

```c++
void* operator new(std::size_t size){
    using namespace std;
    if(size==0){
        size=1;
    }
    while(true){
        if(分配成功){
            return (一个指向这块内存空间的指针);
        }
        new_handler globalHander=set_new_handler(0);
        set_new_handler(globalHander);
        if(globalHander){
            (*globalHander)();
        }else{
            throw std::bad_alloc();
        }
    }
}
```

要求分配0字节，但是操作系统本身需要标识对象本身式独一无二的，所以需要1的占位内存，所以这时候会把size调整至1，然后调整new_handler的参数，进行递归查找，若返回是null，则抛出异常

当然对于每个类本身的operator new都可以进行专门的设计，但是一旦在base class处自定义了一个operator new，但是如果他被继承了，derived class自然也会继承这个operator new的行为，那么这时候分配的字节数，就有可能对不上了，所以应对这种情况，我们需要对这个operator new进行字节大小检查，即size!=sizeof(Base)，如果是这样就将分配内存的动作转交给global的operator new，operator array new也是同理，只要是无法处理的非正确情况一律转交给::operator new

- operator new应该内涵一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用new-handler，它也应该有能力处理0byte申请。class专属版本则还应该处理比正确大小更大的错误请求
- operator delete应该在收到null指针时不做任何事情，class专属版本则还应该处理比正确大小更大的错误申请

## 条款52:写了placement new也要写placement delete

placement new是一种比较特别的new，他是在一块已有的内存上调用构造函数，不会引发内存分配的动作。考虑这个动作Widge *pw=new Widge;这样是有可能发生隐藏极深的内存泄漏，因为这个动作执行了内存分配的operator new，第二是执行了构造函数，所以当构造函数抛出异常的时候，那么那块分配的内存就不会调用operator delete这个动作，自然的就内存泄露了。取消步骤一并恢复旧观的责任是C++运行期系统的责任，所以他会去寻找所调用的operator new的相应operator delete版本，前提当然是它必须知道哪一个operator delete被调用。如果面对的是拥有正常签名的new和delete，并不是问题，但是如果存在这种情况

```c++
void* operator new(std::size_t) throw(std::bad_alloc);
void operator delete(void* rawMemory) throw();
void operator delete(void* rawMemory,std::size_t size) throw();
```

因此当只使用正常形式的new和delete，运行期系统都能找到对应的delete，然而非正常形式的operator new，是哪一个delete与之对应，就出现了很大的问题

如果一个operator new接受的参数处size_t之外还有其他，这就是placement new，placement new中最有用的那个就是接受一个指针指向对象该被构造指出，以这样的形式展示

```c++
class Wiget{
public:
    ...
    static void* operator new(std::size_t size,std::ostream& logStream) throw(std::bad_alloc);
    static void operator delete(void* pMemory,std::size_t size) throw();
    ...
};
```

void* operator new(std::size_t,void* pMemory) throw();

这个placement new的定义即是在一个已经分配好的未使用空间上直接创建对象，那么不会引发内存分配的问题，那么这时候就能方便我们对内存进行管理了。例如说这个动作 Widge *pw=new (std::cerr) Widge;如果内存分配成功，构造函数异常，那么我们应该恢复旧观，那么运行期系统会选择寻找参数个数和类型斗鱼operator new相同的某个operator delete，如果找到，那就调用它，选择补上这种delete形式，就能防止这种难以察觉的内存泄漏。当然，placement delete只有在伴随placement new调用而触发的构造函数抛出异常才会被调用，因为这是他的对应delete，其他的new形式都与其对应不上

要特别注意的是，由于成员函数的名称会掩盖其外围作用域中的所有同名函数，必须小心避免让class专属的new掩盖客户期望的其他new。所以对于撰写内存分配函数，需要记住，缺省情况下C++在global作用域内提供以下形式的operator:

```c++
void* operator new(std::size_t) throw(std::bad_alloc);//常规new
void* operator new(std::size_t,void*) throw();//placement new
void* operator new(std::size_t,const std::nothrow_t&) throw();//不抛异常的new
```

如果在class内声明任何operator new，它会遮掩上述标准形式，如果希望这些函数有着平常的行为，只要令class专属版本呢调用global版本即可

```c++
class StandardNewDeleteForms{
public:
    //常规new
    static void* operator new(std::size_t) throw(std::bad_alloc){
        return ::operator new(size);
    }
    static void operator delete(void* pMemory) throw(){
        ::operator delete(pMemory);
    }
    //placement new
    static void* operator new(std::size_t size,void *ptr) throw(){
        return ::operator new(size,ptr);
    }
    static void operator delete(void* pMemory,void *ptr) throw(){
        ::operator delete(pMemory,ptr);
    }
    //不抛异常的new
    static void* operator new(std::size_t size,const std::nothrow_t& nt) throw(){
        return ::operator new(size,nt);
    }
    static void operator delete(void* pMemory,const std::nothrow_t&) throw(){
        ::operator delete(pMemory);
    }
};
```

只要是以自定形式扩充标准形式的行为，可利用继承机制及using声明式取得标准形式然后再进行添加

- 写一个placement operator new，请确定也写了对应的operator delete，如果缺少，可能会发生隐微而时断时续的内存泄漏。
- 当声明了placement new与delete的时候，不要无意识的覆盖了他们的正常版本。

## 条款54:让自己熟悉包括TR1在内的标准程序库

TR1组件实例:

- 智能指针

    应用最广泛的TR1组件，有tr1::shared_ptr和tr1::weak_ptr，前者的作用犹如内置指针，但会记录有多少个shared_ptr共同指向同一个对象，这就是引用计数，一旦最后一个这样的指针被销毁，也就是引用计数变为0，这个对象就被自动删除。这在非环形数据结构中可以防止资源泄漏，但是如果出现环形指向，会不断等待对方资源的解放，导致都无法得到内存释放，转而变成内存泄漏。提出weak_ptr，其设计使其表现像是非环形shared_ptr-based数据结构中的环形感生指针，其并不参与引用计数的计算，当最后一个对象的shared_ptr被销毁，即是有个weak_ptr对象继续指向同一对象，该对象仍会被删除，weak_ptr会被自动指向无效。

- function

    function可以表示任何可调用物(任何函数或函数对象)，只要签名符合该目标。例如我们需要注册一个callback函数，该函数声明:void callback(std::string func(int));这里的参数类型是函数，该函数接受一个int并返回一个string，其中参数名称func可有可无，所以声明也可以如下:void callback(std::string (int));这里的std::string (int)就是一个函数签名。tr1::function是上述的callback有可能更富有弹性地接收任何可调用物，只要这里接受一个int或任何可转换为int的东西，并返回string或任何可被转换为string的东西，tr1::function是个template，以其目标函数的签名为参数，可以声明如下:void callback(std::tr1::function<std::string (int)>func);

- bind

    他能做STL绑定器bind1st和bind2nd所做的每一件事，而又更多，tr1::bind可以和const和非const成员函数协作运行，可以和by-reference参数协同运行，而且它不需要特殊协助就可以处理函数指针

- hash tables

    用来实现unordered容器的组件，形成的内部数据是无序的

- 正则表达式

    包括以正则表达式为基础的字符串查找和替换，或是从某个匹配字符串到另一个匹配字符串的逐一迭代等等

- Tuples

    标准库中的pair template的新一代制品，pair只能持有2个对象，tr1::tuple可持有任意个数的对象

- array

    本质是一个STL化数组，即支持STL中的迭代器与算法模块，提供更强的交互性，但是其大小固定，并不适用动态内存

- reference_wrapper

    一个让references的行为更像对象的设施，它可以造成容器犹如持有references，而实际上容器只能持有对象或指针

- 随机数生成工具

    超越了rand的一个随机数生成工具

- 数学特殊函数

    包括Laguerre多项式、Bessel函数、完全椭圆即分，以及更多数学函数

- C99兼容扩充

- Type traits

    一组traits classes，用以提供类型的编译期信息。基于一个类型T，TR1的type traits可以指出T是否是个内置类型，是否提供virtual析构函数，是否是个空类，可隐式转换为其它类型U吗.....等等，TR1的type traits也可以显现该给定类型之适当齐位

- result_of

    这是个template，用来推导函数调用的返回类型。当我们编写templates时，能够指涉函数(函数模板)调用动作所返回的对象的类型往往很重要，但是该类型有可能以复杂的方式取决于函数的参数类型，tr1::result_of使得指涉函数返回类型变得十分容易
