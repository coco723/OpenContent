
### 引用References出现的背景

想要读取一个string2次 按ownership的方法就比较麻烦，需要调用的函数返回值 并接收这个值，所以有了引用References

## **reference** 是Non-Owning指针

**reference** 是一种没有所有权的指针 

下面这个函数会报错 因为m1 m2会被函数执行之后回收

```rust
fn main() {
    let m1 = String::from("Hello");
    let m2 = String::from("world");
    greet(m1, m2);
    let s = format!("{} {}", m1, m2); // Error: m1 and m2 are moved
}

fn greet(g1: String, g2: String) {
    println!("{} {}!", g1, g2);
}
```

可以通过函数返回值，或者引用的方式来解决

函数返回值：

```rust
fn main() {
    let m1 = String::from("Hello");
    let m2 = String::from("world");
    let (m1_again, m2_again) = greet(m1, m2);//这里增加了返回值
    let s = format!("{} {}", m1_again, m2_again);
}

fn greet(g1: String, g2: String) -> (String, String) {
    println!("{} {}!", g1, g2);
    (g1, g2)
}
```

引用

& 操作符可以创建一个对m1的引用（或叫借用’borrow’) 除了传参的时候，在函数签名里面，参数的类型注解也改变了 &String代表对String的引用

m1拥有‘hello’，g1既不拥有m1也不拥有’hello‘。greet函数执行完后，只有greet在栈中的frame没有了 其他都不变 这也遵循了盒子释放原则，因为g1不是’Hello’的拥有者

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled.png)

**reference** 是Non-Owning指针，因为它们不拥有它们所指向的数据

## 取消引用指针

**取消引用使用 * 操作符**

```rust
let mut x: Box<i32> = Box::new(1);
let a: i32 = *x;         // *x 直接读取 heap值，所以a等于1
*x += 1;                 // *x 改变了 heap 值,
                         //    所以x会指向2

let r2: &i32 = &*x;      // r2 直接指向heap的值;
```

注意：这里x是被赋值了Box::new(1);， 而不是1，所以会在heap里面创建一个Box，如果是1,就是在stack里面，像a一样

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%201.png)

再来看一种情况：

```rust
fn main() {
    let mut point = [0, 1];
    let mut x = point[0]; 
    let y = &mut point[1]; 
    x += 1; 
    *y += 1; 
    println!("{} {}", point[0], point[1]);
}
```

这里的point[0], point[1]分别会打印什么？😯

答案是0,2

```rust
fn main() {
    let mut point = [0, 1];
    let mut x = point[0];//x是对point[0]的复制
    let y = &mut point[1];//y是可变引用
    x += 1;//改变x不会影响指针 
    *y += 1;//改变y会改变指针
    println!("{} {}", point[0], point[1]);
}
```

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%202.png)

**取消引用有显式和隐式**

```rust
let x: Box<i32> = Box::new(-1);
let x_abs1 = i32::abs(*x); // explicit dereference i32::abs函数期望一个i32类型的输入 所以可以像这样显式取消引用
let x_abs2 = x.abs();      // implicit dereference 这种写法是上面那种函数调用写法的语法糖，不过是隐式取消引用
```

引用也有显式和隐式

```rust
let s = String::from("Hello");
let s_len1 = str::len(&s); // explicit reference 显式引用
let s_len2 = s.len();      // implicit reference调用在String上的len
assert_eq!(s_len1, s_len2);
```

## Rust会避免同时出现Aliasing和Mutation

指针会导致aliasing。

aliasing：通过不同的变量访问相同的数据

向量数据结构：Vec，长度是可变的

```rust
let mut v: Vec<i32> = vec![1, 2, 3];//<i32>表示vector中的元素都是i32类型
v.push(4);
```

v变量有一定的容量去分配heap数组，这里v变量长度为3 容量capacity也是3

在v.push(4);之后，v要扩容，创建一个新的内存，会释放掉原来的heap数组，就是【1,2,3】

```rust
let mut v: Vec<i32> = vec![1, 2, 3];
let num: &i32 = &v[2];
v.push(4);
println!("Third element is {}", *num);//报错 因为v.push(4);之后释放了num指向vec
```

抽象地讲，这种报错就是因为 v同时被别名（被num引用）和被改变mutated(v.push(4);) 

指针安全原则：数据不能**同时**被别名和改变

## 引用可以通过path改变权限

借用检查器的核心要点是变量对数据有3种权限

- 读取R  将数据复制到其他位置
- 写入W 数据可以就地改变
- 拥有O 数据可以被移动或删除

这些权限只在编译器中存在 在运行时不存在

默认，变量有读取和拥有的权利，如果变量是通过let mut声明的，就有写的权利。**引用可以暂时移除这些权利**

注意：这里&v[2]移走了v的W和O，只要是借用就会移走原来变量的拥有权和写入 

```rust
let mut v: Vec<i32> = vec![1, 2, 3];//v: R W O
let num: &i32 = &v[2];//v: R num: O R 因为num不是通过let mut来声明的 所以没有write
println!("Third element is {}", *num);//打印之后 num就没有了O和R v重新获得W O ，v:W R O
//println!("Third element is {}", *num);//如果后面还有使用*num 就在这里num再失去权利
v.push(4);//v没有了所有权限
```

为什么会同时看到num和 *num

通过引用访问数据和操作引用是不同的

```rust
let x =0;//x: R O
let mut x_ref = &x;//x:R,
//x_ref借用了x所以有了R O,又因为它是通过let mut定义的，所以有了W，x_ref:R W O , *x_ref:R
//这样再次给x_ref赋值就可以 但是不能改变*x_ref的值 
```

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%203.png)

x_ref有写入权限，所以可以x_ref=&y

*x_ref没有写入权限，所以不能被修改 *x_ref+=1是不行的

**path的定义**

权限是在path上定义的，而不仅仅是变量，一个path是任何可以放在表达式左侧的东西，它包含

1. 变量 a
2. 对路径取消引用 *a
3. 数组访问 a[0]
4. 路径的field 比如tuple类型的a.0或者struct类型的a.field
5. 上面提到的项的组合，比如 *((*a)[0].1)

## 借用检测器发现权限冲突

创建对象数据的引用&就是借用borrow。这里num对v的引用会导致v暂时为只读，直到不再使用num。

Rust使用借用检测器来检查潜在的不安全引用操作

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%204.png)

rust在使用path的时候，会根据对变量的操作显示变量应该要有的属性

这里&v[2]的时候需要v是可读的 所以R就显示了

而v.push(4)的时候 需要v是可读而且可以写入的 所以R和W都显示了 但是此时v没有W属性 所以W是镂空的，这样编译就会报错

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%205.png)

报错显示v不能被改变 因为reference:num还在使用，push之后num就无效了

## 可变引用提供了对数据单一且不拥有的权限

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%206.png)

num通过&mut变成了一个可变的引用。这和之前的区别：

1. num是不可变的时候,v还有R读取的权限，现在num是一个可变引用，只要num还在被用，v就什么权限都没有了，这就遵循了引用安全原则，v暂时是不可用的，所以就不构成别名
2. num是不可变的时候，*num只有R读取权限，现在num是一个可变引用，*num也获得了W写入权限，v[2]可以通过*num被改变，*num+=1就改变了v[2]。但是num没有W写入权限，因为num指向了这个可变引用本身，所以num不能被再次赋值

可变的引用也可以暂时被变成只读的引用

在这里是通过 &*num移走了*num的W权限

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%207.png)

## 权限在引用的生存周期结束后返回

生存周期：引用创建-引用最后一次被使用

在let z= *y 之后 y的W权限会返回给x 

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%208.png)

在不同的if分支里面 c有不同的生命周期，在then分支里面，*v得到c.to_ascii_uppercase()执行完之后才重新获得W权限，在else分支里面，*v一进入else分支就得到W权限

![Untitled](References%20and%20Borrowing%2053e01dc9ba774f7ab88202055ac484e7/Untitled%209.png)

```rust
fn get_first(v: &Vec<String>) -> &str {
    &v[0]
}

fn main() {
    let mut strs = vec![
        String::from("A"), String::from("B")
    ];
    let first = get_first(&strs);//通过get_first引用&strs 所以first借用了&strs
    if first.len() > 0 {
        strs.push(String::from("C"));
    }
}
```

## 数据必须比其所有引用都更持久

rust使用2种方式确保数据比其所有应用更持久：

1. 处理在一个作用域内创建和删除的引用

rust知道普通引用能存活多久 但是对于属于函数输入或者函数输出的引用 rust需要一个不同的机制 

1. 使用流权限

F是流权限(Flow permission) 当一个表达式使用了输入引用或者返回了一个输出引用，就需要流权限

```rust
fn first(strings: &Vec<String>) -> &String {
    let s_ref = &strings[0];//输入引用
    s_ref//返回输出引用
}
```

不像RWO，F在整个函数的执行过程中没有改变。一个引用如果允许被使用就有F权限：

这段代码不会编译 报错**missing lifetime specifier**

```rust
fn first_or(strings: &Vec<String>, default: &String) -> &String {
    if strings.len() > 0 {
        &strings[0]//没有流权限
    } else {
        default//没有流权限
    }
}
```

如果rust只看函数签名，就不知道输出的&String是对strings的引用还是default的引用 

如果像下面这样子使用first_or函数 代码不会被编译,因为rust需要确定default变量不会被流到返回值。因为如果default是返回值的一部分或者是返回值，drop会导致s无效 为了指定default能不能被返回，rust提供了lifetime parameters。（在第10章）现在只需知道：

1. input/output引用在函数体中和普通引用是不一样的
2. rust使用了流控制F来检查input/output引用的安全

```rust
fn main() {
    let strings = vec![];
    let default = String::from("default");
    let s = first_or(&strings, &default);
    drop(default);
    println!("{}", s);
}
```

测试题目：

这段代码会被编译吗？为什么

```rust

fn incr(n: &mut i32) {
  *n += 1;
}
fn main() {
  let mut n = 1;
  incr(&n);
  println!("{n}");
}
```

这段代码不会被编译 因为n是一个mut 所以对它的引用也得是mut incr(&mut n)

```rust

fn main() {
  let mut s = String::from("hello");
  let s2 = &s;
  let s3 = &mut s;
  s3.push_str(" world");
  println!("{s2}");//当不可变的s2还活跃的时候 就使用mut s给到s3是不被允许的
}
```

## Summary

引用提供了一种无需所有权的数据读取和写入方式 

引用是通过borrows(&和&mut)还有取消引用(*)创建的，通常是隐式创建

Rust使用借用检测器来确保引用被安全使用：

1. 所有的变量都可以读取 拥有 写入数据(optionally)
2. 创建一个引用会将权限从borrowed path转移到引用
3. 一旦引用的lifetime结束 权限就会返回
4. 数据的生命周期必须比所有指向它的引用的生命周期长

参考：
https://rust-book.cs.brown.edu/