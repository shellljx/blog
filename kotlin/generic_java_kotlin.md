### Java 和 kotlin 泛型

**相同之处**

1. Kotlin 泛型和 java 泛型相当相近：他们使用同样的方式声明泛型函数和泛型类

2. 和 java 一样，泛型类型的类型实参只在编译期存在运行期擦除

3. kotin 不能把带类型实参的类型和 `is` 运算符一起使用，因为类型实参在运行时将被删除，java 则不能和 `instanceOf` 运算符一起使用

4. 变形都是说明两种拥有相同基础类型和不同类型参数的泛型类型之间子类型化关系的方式，都说明了如果其中一个泛型类型参数是另一个类型参数的子类型，这个泛型类型就是另一个泛型类型的子类型或者超类型

   ```java
   List<? super Fruit> fruits = new ArrayList<>();
   List<? super Apple> apples = new ArrayList<>();
   
   apples = fruits; //可以，Apple 某个超类型 list 是 Fruit 某个超类型 list 的父类型
   fruits = apples; //不可以
   
   List<? extends Fruit> fruits = new ArrayList<>();
   List<? extends Apple> apples = new ArrayList<>();
   fruits = apples;	//可以，Apple 某个子类型的list 是 Fruit 某个子类型的 list 的子类型
   apples = fruits;	//不可以
   
   ```

   

5. 都可以声明一个类在某个类型参数上是**协变**的，`out` 和 `? extends` 对应，在运行时泛型会被擦除保留上界类型，都表明只能向外提供数据，无法向列表中添加数据
6. 都可以声明一个类在某个类型参数上是**逆变**的，`in` 和 `? super` 对应，在运行时泛型会被擦除保留下界类型，只能往里插入下界类型和下界的子类型，向外提供数据也只能得到 object
7. 当确切的类型实参是未知或者不重要时，kotlin 可以使用`星号投影` java 可以使用 `?`

**不同之处**

1. java 可以声明原始类型(因为 java 最开始没有泛型) 而 kotlin 不可以声明原始类型