# Learning Rust: The Compiler is your Friend

The Rust programming language doesn't just aim to be practical, it also aims to be useful for the people working with it. Not only does this improve productivity, it also helps learning the language! One of the features I want to pick out: the borrow checker of the Rust compiler. This feature helps to avoid a great number of memory related bugs by enforcing correct memory allocation and use of pointers. At the same time, it also seems to be the source for a lot of frustration. This is reflected in in a phrase we often see: "fighting the compiler".

The reason why you can use the Rust compiler as your personal mentor when learning a core concept in Rust lies in the information, the error messages provide. I want to demonstrate this with a simple example about mutability and borrowing. You can try this yourself in this [playground](https://play.rust-lang.org/?version=stable&amp;mode=debug&amp;edition=2018&amp;gist=208af39ba7facc50154027834691ee74).

We define a variable `item` and bind a `String` containing the word *sausage* to it. A dog, Belka, comes along and wants to take bite of *sausage*. This is symbolized by the `String` being truncated.

```rust
fn main() {
    let item = String::from("sausage");
    println!("Belka finds a {} and takes a bite off of it.", item);
    belka_takes_bite_off(item);

}

fn belka_takes_bite_off(sausage: String) {
    sausage.truncate(5);
    println!("The left over {} lies in front of her.", sausage);
}
```

We run the code, but get an error message:

```
 Compiling playground v0.0.1 (/playground)
error[E0596]: cannot borrow `sausage` as mutable, as it is not
declared as mutable
 --> src/main.rs:9:5
  |
8 | fn belka_takes_bite_off(sausage: String) {
  |                       -------- help: consider changing this to be
mutable: `mut sausage`
9 |     sausage.truncate(5);
  |     ^^^^^^^^ cannot borrow as mutable

error: aborting due to previous error

For more information about this error, try `rustc --explain E0596`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

The compiler gives us an error number `E0596` with the corresponding verbal message: `cannot borrow 'sausage' as mutable, as it is not declared as mutable`. The verbal message does not only name the operation that broke Rust's rules, it also provides a reason why this operation is not possible. Additionally to being very explicit about the error, the compiler specifies the line, where the faulty operation takes place. Within the line, the affected variable is underlined.

It does not stop there. We're also offered help: We should consider changing the parameter to be mutable. The place in the code, where this needs to happen is also underlined.

From this error message we learn that variables need to be declared as mutable, if you want to change the data behind them.

Thanks to the specific instruction, this error is fixed very quickly by changing line 8 the following way:

```rust
fn belka_takes_bite_off(mut sausage: String) {
```
Running the program now yields this output:

```
Belka finds a sausage and takes a bite off of it.
The left over sausa lies in front of her.
```

The mutation of the data behind the variable was successful.

<hr>

The compiler does not always offer this kind of specific help. Often the course of action to resolve an error depends on the characteristics and the purpose of your data and the goals of your program.

Let's add another dog, Strelka, to the picture, who wants to eat the remains of the sausage. You can follow along in this [Playground](https://play.rust-lang.org/?version=stable&amp;mode=debug&amp;edition=2018&amp;gist=f8c6e55e62b90a8f3400198d3bd0a410).

```rust
fn main() {
    let item = String::from("sausage");
    println!("Belka finds a {} and takes a bite off of it.", item);
    belka_takes_bite_off(item);
    println!("Strelka steals the left overs.");
    strelka_takes_and_eats(item);

}

fn belka_takes_bite_off(mut sausage: String) {
    sausage.truncate(5);
    println!("The left over {} lies in front of her.", sausage);
}

fn strelka_takes_and_eats(mut sausage: String) {
    println!("Strelka swallows the {} in one bite.", sausage);
    sausage.clear();
    println!("Length of left over: {}", sausage.len());
}
```

When running this program, we get an error message: We tried to use a value that has already been moved.

```
 Compiling playground v0.0.1 (/playground)
error[E0382]: use of moved value: `item`
 --&gt; src/main.rs:6:29
  |
2 |     let item = String::from("sausage");
  |         ---- move occurs because `item` has type
`std::string::String`, which does not implement the `Copy` trait
3 |     println!("Belka finds a {} and takes a bite off of it.", item);
4 |     belka_takes_bite_off(item);
  |                        ---- value moved here
5 |     println!("Strelka steals the left overs.");
6 |     strelka_takes_and_eats(item);
  |                             ^^^^ value used here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
```

This time, we learn again what caused the error: the Type used does not implement `Copy` trait, but we're not given help. This situation is tempting to just implement the `Copy` trait as a fix, as it is obviously missing. The missing implementation is the reason why your intended operation is causing an error. This does **not** imply that implementing the missing trait is the best way to fix your error!

But the error message still can help us. Running `rustc --explain E0382` offers a more detailed explanation of this error and multiple approaches to solving the problem:

 - using a mutable reference with `&mut`
 - implementing the `Clone` trait and duplicating the data
 - having a shared ownership with a reference counter.

This is the point, where we reached a boundary of the compiler's guidance. We were provided with the information, but the decision on the course of action is on us and usually results from the program itself.

Looking at our story: There is one sausage, which is partially eaten by one dog and then taken and fully eaten by a second dog.
Cloning the data is out of the question because it would result in two identical sets of data, one for each dog. While it would be nice for the dogs to have a whole sausage each, this is not how the story goes. Speaking more abstract: Having copies of data in memory when it's not necessary is a waste of memory. If a set of data needs to be changed by several owners, making the changes to different copies of the data is not the same, as making changes to the original piece. Implementing the `Copy` trait would have also lead to cloning the data.

For this program, actually transferring the ownership is not necessary, the dogs don't care whom the sausage belongs to, as long as they can have a piece of it, which makes a shared ownership unnecessary. So a mutable reference `&amp;mut` is the way to go:

We change lines 10 and 15:

<div class="highlight"><pre class="highlight rust">`<span class="k">fn</span> <span class="nf">belka_takes_bite_off</span><span class="p">(</span><span class="n">sausage</span><span class="p">:</span> <span class="o">&amp;</span><span class="k">mut</span> <span class="nb">String</span><span class="p">)</span> <span class="p">{</span>

`</pre></div><div class="highlight"><pre class="highlight rust">`<span class="k">fn</span> <span class="nf">strelka_takes_and_eats</span><span class="p">(</span><span class="n">sausage</span><span class="p">:</span> <span class="o">&amp;</span><span class="k">mut</span> <span class="nb">String</span><span class="p">)</span> <span class="p">{</span>

`</pre></div>
Running the program fails:

<div class="highlight"><pre class="highlight plaintext">`   Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --&gt; src/main.rs:4:24
  |
4 |     belka_takes_bite_off(item);
  |                        ^^^^
  |                        |
  |                        expected `&amp;mut std::string::String`, found
                           struct `std::string::String`
  |                        help: consider mutably borrowing here:
                           `&amp;mut item`

error[E0308]: mismatched types
 --&gt; src/main.rs:6:29
  |
6 |     strelka_takes_and_eats(item);
  |                             ^^^^
  |                             |
  |                             expected `&amp;mut std::string::String`,
                                found struct `std::string::String`
  |                             help: consider mutably borrowing here:
                                `&amp;mut item`

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
`</pre></div>When making the necessary changes to have mutable references to the item instead of transferring its ownership, beginners often stumble over the fact that `T` and `&amp;mut T` are two different types. But don't worry, the compiler is still helpful, and guides you through to getting the types right: We learn, that if a function expects a certain type, the call of the function needs to happen with that same type.

So we change the variable `item` to `&amp;mut item` in the function calls:

<div class="highlight"><pre class="highlight rust">`    <span class="nf">belka_takes_bite_off</span><span class="p">(</span><span class="o">&amp;</span><span class="k">mut</span> <span class="n">item</span><span class="p">);</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"Strelka steals the left overs."</span><span class="p">);</span>
    <span class="nf">strelka_takes_and_eats</span><span class="p">(</span><span class="o">&amp;</span><span class="k">mut</span> <span class="n">item</span><span class="p">);</span>
`</pre></div>
When learning Rust, it often is a back and forth, making changes, running the code, getting an error message, repeat. So the last changes we made are still not enough. Running the code yields us the next error message:

<div class="highlight"><pre class="highlight plaintext">`   Compiling playground v0.0.1 (/playground)
error[E0596]: cannot borrow `item` as mutable, as it is not declared
as mutable
 --&gt; src/main.rs:4:24
  |
2 |     let item = String::from("sausage");
  |         ---- help: consider changing this to be mutable: `mut item`
3 |     println!("Belka finds a {} and takes a bite off of it.", item);
4 |     belka_takes_bite_off(&amp;mut item);
  |                        ^^^^^^^^^ cannot borrow as mutable

error[E0596]: cannot borrow `item` as mutable, as it is not declared
as mutable
 --&gt; src/main.rs:6:29
  |
2 |     let item = String::from("sausage");
  |         ---- help: consider changing this to be mutable:
                 `mut item`
...
6 |     strelka_takes_and_eats(&amp;mut item);
  |                             ^^^^^^^^^ cannot borrow as mutable

error: aborting due to 2 previous errors

For more information about this error, try `rustc --explain E0596`.
error: could not compile `playground`.

To learn more, run the command again with --verbose.
`</pre></div>We're already familiar with this error and the required mutability declaration is made quickly:

<div class="highlight"><pre class="highlight rust">`<span class="k">fn</span> <span class="nf">main</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">let</span> <span class="k">mut</span> <span class="n">item</span> <span class="o">=</span> <span class="nn">String</span><span class="p">::</span><span class="nf">from</span><span class="p">(</span><span class="s">"sausage"</span><span class="p">);</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"Belka finds a {} and takes a bite off of it."</span><span class="p">,</span> <span class="n">item</span><span class="p">);</span>
    <span class="nf">belka_takes_bite_off</span><span class="p">(</span><span class="o">&amp;</span><span class="k">mut</span> <span class="n">item</span><span class="p">);</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"Strelka steals the left overs."</span><span class="p">);</span>
    <span class="nf">strelka_takes_and_eats</span><span class="p">(</span><span class="o">&amp;</span><span class="k">mut</span> <span class="n">item</span><span class="p">);</span>

<span class="p">}</span>

<span class="k">fn</span> <span class="nf">belka_takes_bite_off</span><span class="p">(</span><span class="n">sausage</span><span class="p">:</span> <span class="o">&amp;</span><span class="k">mut</span> <span class="nb">String</span><span class="p">)</span> <span class="p">{</span>
    <span class="n">sausage</span><span class="nf">.truncate</span><span class="p">(</span><span class="mi">5</span><span class="p">);</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"The left over {} lies in front of her."</span><span class="p">,</span> <span class="n">sausage</span><span class="p">);</span>
<span class="p">}</span>

<span class="k">fn</span> <span class="nf">strelka_takes_and_eats</span><span class="p">(</span><span class="n">sausage</span><span class="p">:</span> <span class="o">&amp;</span><span class="k">mut</span> <span class="nb">String</span><span class="p">)</span> <span class="p">{</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"Strelka swallows the {} in one bite."</span><span class="p">,</span> <span class="n">sausage</span><span class="p">);</span>
    <span class="n">sausage</span><span class="nf">.clear</span><span class="p">();</span>
    <span class="nd">println!</span><span class="p">(</span><span class="s">"Length of left over: {}"</span><span class="p">,</span> <span class="n">sausage</span><span class="nf">.len</span><span class="p">());</span>
<span class="p">}</span>

`</pre></div>Running the code finally gives us this output:

<div class="highlight"><pre class="highlight plaintext">`Belka finds a sausage and takes a bite off of it.
The left over sausa lies in front of her.
Strelka steals the left overs.
Strelka swallows the sausa in one bite.
Length of left over: 0
`</pre></div>
This above mentioned going in circles of making minor changes, compiling and getting an error message can be frustrating. But even if it seems like the compiler is working against you, it provides you a railing: You're either given help directly or provided with the information you need to make a decision for your course of action. While this can feel like a fight at first, the compiler is actually your sparring partner that holds a focus mitt for you to practice your punches. You will get better with time, the frequency of getting error messages will get lower, because you learn, where you have to make references, where you have to declare mutability, when a memcopy is the better idea over referencing. With time, your punches will land.

The compiler is a strict training partner. But in the end, even beginners have written a memory safe program, which is quite an achievement!

Takeaways from this blog post:

<ul>
 - Follow the help provided in the error message.
 - If no help is provided, run `rustc --explain` and read the output, before removing causes of errors to solve them.
 - The Rust compiler is your friendly sparring partner.
</ul>

<h2 id="stay-tuned-">Stay tuned …</h2>
In the next part of this series, we will talk about the journey of learning Rust from a C/C++ background. We will speak about the rewards but also about the pitfalls to look out for.

In the meantime, please consider signing up for our training <a href="http://eepurl.com/gYWZKr">newsletter</a> or enlisting us as <a href="https://ferrous-systems.com/training">trainers</a> for your team.
</div>
      <ul class="article__nav">
        <li class="article__navitem article__navitem--previous">
            <a href="/blog/probe-run/">Last Article</a>

        <li class="article__navitem article__navitem--next">
            <a href="/blog/run-rust-on-your-embedded-device-from-vscode/">Next Article</a>

      </ul>
