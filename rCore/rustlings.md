### Good examples:

```rust
fn main() {
    let cat = ("Furry McFurson", 3.5);
    let (name, age)/* your pattern here */ = cat;
    println!("{} is {} years old.", name, age);
}

fn indexing_tuple() {
    let numbers = (1, 2, 3);
    let second = numbers.1; // counting from 0.
    assert_eq!(2, second,
    "This is not the 2nd number in the tuple!")
}

fn main() {
    let mut vec0 = Vec::new();
    let mut vec1 = fill_vec(&mut vec0);
    println!("{} has length {} content `{:?}`", "vec1", vec1.len(), vec1);
    vec1.push(88)
    println!("{} has length {} content `{:?}`", "vec1", vec1.len(), vec1);
}
// return a mutable reference, but can simply transfer to a vector/mutable vector/etc.
fn fill_vec(vec:&mut Vec<i32>) -> &mut Vec<i32> {
    vec.push(22);
    vec.push(44);
    vec.push(66);
    vec
}

// pay more attention to string_uppercase.
  5 fn main() {
  6     let mut data = "Rust is great!".to_string();
  7
  8     get_char(&mut data);
  9
 10     string_uppercase(&mut data);
 11 }
 12
 13 // Should not take ownership
 14 fn get_char(data: &mut String) -> char {
 15     data.chars().last().unwrap()
 16 }
 17
 18 // Should take ownership
 19 fn string_uppercase(data: &mut String) {
 20     *data = data.to_uppercase();
 21
 22     println!("{}", data);
 23 }
```

I found that my opinion on this has changed significantly over the past 12 months as I have grown more experienced with Rust.

I now strongly prefer to_owned() for string literals over either of to_string() or into().

What is the difference between String and &str? An unsatisfactory answer is "one is a string and the other is not a string" because obviously both are strings. Taking something that is a string and converting it to a string using to_string() seems like it misses the point of why we are doing this in the first place, and more importantly misses the opportunity to document this to our readers.

The difference between String and &str is that one is owned and one is not owned. Using to_owned() fully captures the reason that a conversion is required at a particular spot in our code.
```rust
struct Wrapper {
    s: String
}

// I have a string and I need a string. Why am I doing this again?
Wrapper { s: "s".to_string() }

// I have a borrowed string but I need it to be owned.
Wrapper { s: "s".to_owned() }
```

The cheat sheet is very useful!

https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html#modules-cheat-sheet

#### special case: 
for 'delicious_snacks', the 'fruits' can be cited without using 'pub' keyword. But for things inside the 'fruits' mod, the 'pub' keyword will be a must.

'pub use': for external usage. (external to mod delicious_snacks).
```rust
  5 mod delicious_snacks {                                                                              │favorite snacks: Pear and Cucumber
  4     // TODO: Fix these use statements                                                               │
  3     // use 'pub use' to be used by external things.                                                 │====================
  2     // Or                                                                                           │
  1     // pub use self::fruits::PEAR as fruit;                                                         │You can keep working on this exercise,
13      // pub use self::veggies::CUCUMBER as veggie;                                                   │or jump into the next one by removing the `I AM NOT DONE` comment:
  1                                                                                                     │
  2     pub mod fruits {                                                                                │ 4 |  // Execute `rustlings hint modules2` or use the `hint` watch subcommand for a hint.
  3         pub const PEAR: &'static str = "Pear";                                                      │ 5 |
  4         pub const APPLE: &'static str = "Apple";                                                    │ 6 |  // I AM NOT DONE
  5     }                                                                                               │ 7 |
  6                                                                                                     │
  7     pub mod veggies {                                                                               │
  8         pub const CUCUMBER: &'static str = "Cucumber";                                              │
  9         pub const CARROT: &'static str = "Carrot";                                                  │
 10     }                                                                                               │
 11 }
 ```

#### ref:
Bind by reference during pattern matching.

match values can always consume the value.

https://doc.rust-lang.org/std/keyword.ref.html
#### result
话说这里的`Result<String, String>`倒是有点出乎我的意料。
```rust
  pub fn generate_nametag_text(name: String) -> Result<String, String> {                              │   |        Result<i32, ParseIntError>
  1     if name.is_empty() {                                                                            │
  2         // Empty names aren't allowed.                                                              │error: aborting due to previous error
  3         Err("`name` was empty; it must be nonempty.".into())                                        │
  4     } else {                                                                                        │For more information about this error, try `rustc --explain E0369`.
  5         Ok(format!("Hi! My name is {}", name))                                                      │
  6     }                                                                                               │
  7 }


   12 // This is required so that `CreationError` can implement `error::Error`.
 11 // Note this!!
    // errors5.rs. 
    // To return multiple types of errors in ONLY ONE FUNCTION.
   9 // TODO: update the return type of `main()` to make this compile.
 10 fn main() -> Result<(), Box<dyn error::Error>> {
 11     let pretend_user_input = "42";
 12     let x: i64 = pretend_user_input.parse()?;
 13     println!("output={:?}", PositiveNonzeroInteger::new(x)?);
 14     Ok(())
 15 }
 10 impl fmt::Display for CreationError {
  9     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
  8         let description = match *self {
  7             CreationError::Negative => "number is negative",
  6             CreationError::Zero => "number is zero",
  5         };
  4         f.write_str(description)
  3     }
  2 }
  1
66  impl error::Error for CreationError {}
```

我明白您的困扰了。😊

根据搜索结果，您可以尝试以下方法来在vim的:term中上下翻页：

按Ctrl + w，然后按Shift + n，暂停终端，然后可以使用Ctrl-b或Ctrl-f来翻页，使用gg或G来跳到顶部或底部。按i返回到终端1。
按Ctrl + \，然后按Ctrl + n，进入普通模式（vim 8.1），然后可以使用Ctrl-u或Ctrl-d来翻半屏，使用Ctrl-b或Ctrl-f来翻全屏。按i或a从任何页面输入命令2。
映射PageUp和PageDown键到其他功能键，比如F1和F2，然后在vim的:term中使用它们来翻页3。
这些方法对您有帮助吗？

#### 生命周期
下面这个函数对切片进行了引用，但是引用的时候并没有声明相关的切片的生命周期。由于返回的值可能是x也可能是y，而返回的情况将会影响到引用的生命周期，因此下面这个函数没法编译成功。
```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
需要对上面引用变量的生命周期做明确的规定，因为引用并非数据本身，在函数行为与分支不明确的时候，很难在运行之前将自身的生命周期搞清楚。

#### 智能指针
其中`Arc::new`表示可以有多个所有者，`Mutex`则是可以共享的空间。为什么需要`Arc`来修饰？原因在于`Mutex`一般情况下只能有一个所有者。整个思维脉络在`book`的`16.3`中有详细的介绍。
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```
`RC`指针的特色：共享与克隆。

`COW`指针的特色：慵懒的拷贝，一般情况下默认包裹住的东东的值不改变，在需要改变时加上`to_mut()`函数来辅助。
### 线程3：
thread 3这道题是我完全想不到的，建议之后把材料看清楚了。