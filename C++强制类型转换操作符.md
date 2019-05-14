# C++强制类型转换操作符

强制类型转换也称为显式转换，C++中强制类型转换操作符有static_cast、dynamic_cast、const_cast、reinterpert_cast四个。本节介绍static_cast操作符。

## static_cast
### 替代任何隐式类型转换:
如:int与float,int与char, double与char,int与enum等;
```
double a = 3.1415926;
int b = static_cast<int>(a);
```
把精度大的转换成精度小的类型,static_cast使用位截断进行处理.

### 将void&#42;类型转换成真实的类型
```
double a = 3.1415926;
void * ptr = &a;
double * dptr = static_cast<double *>(ptr)
```

### 用于基类与派生类之间指针或者引用转换
static_cast也可以用在于基类与派生类指针或引用类型之间的转换时不做运行时的检查,dynamic_cast会做运行时检查.static_cast仅仅是依靠类型转换语句中提供的信息来进行转换，而dynamic_cast则会遍历整个类继承体系进行类型检查,因此dynamic_cast在执行效率上比static_cast要差一些。
 1. 进行上行转换（把派生类的指针或引用转换成基类表示）是安全的；
 2. 进行下行转换（把基类指针或引用转换成派生类表示）时，由于没有动态类型检查，所以是不安全的。
```
class Boat
{
public:
    Car():_type("Boat"){};
    vitual void GetName(){cout << "Boat"};
private:
    string _type;
};

class FishingBoat: public Boat {
public:
    FishingBoat():_name("kaka"), _type("fishingBoat"){}
    void GetName(){cout << _name;};
    void GetType(){cout << _type;};
private:
    string _name;
    string _type;
};

int main() {
    //基类指针转换成派生类指针,且派生类指针指向基类对象
    Boat * bt = new Boat;
    FishingBoat *fbt = static_cast<FishingBoat *>(bt);
    fbt->GetType(); //error 基类对象中没有GetType方法, 该错误不会在编译时发现,只会在执行时报错.

    //基类指针转换成派生类指针,且该基类指针指向派生类对象
    Boat * bt = new FishingBoat;
    FishingBoat * fbt = static_cast<FishingBoat *>(bt);
    fbt->GetName();

    //子类指针转换成基类指针
    FishingBoat * fbt = new FishingBoat;
    Boat * bt = static_cast<Boat *>(fbt);
    bt->GetName();
}
```
### static_cast可以把任何类型的表达式转换成void类型
注: static_cast不能把换掉变量的const属性，也包括volitale或者______unaligned属性
