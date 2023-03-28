> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rusty-ferris.pages.dev](https://rusty-ferris.pages.dev/blog/asref-vs-borrow-trait/)

> This post is about my journey to grok the difference between the `AsRef` trait and
> 
> This post is about my journey to grok the difference between the `AsRef` trait and `Borrow` trait (and also compare it with `From` and `Into` trait).
> 
> It started by attempting to write a simple `fn` that takes a two-dimensional array of `char` values; and ended up using ChatGPT (with GPT-4 model) to deepen my understanding of the concepts involved.
> 
> THE STORY
> ---------
> 
> I was trying to write a function that takes a grid (read two-dimensional array) containing chars `W` (represents water) and `L`(represents land). The function returns the number of islands on the grid. [LeetCode](https://leetcode.com/problems/number-of-islands/) reference in case you're interested in the details.
> 
> So I wanted to figure out a way of writing the function so that it takes both an `array` and a `Vec`. I landed on this [forum discussion](https://users.rust-lang.org/t/how-do-you-pass-a-2d-array-to-a-function/2657/6) where the user '_ogeon_'suggests the following code that can"take anything array-ish that contains anything array-ish".
> 
> [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2015)
> 
> ```
> fn batch<Matrix: AsRef<[Row]>, Row: AsRef<[f32]>>(features: Matrix) {
>     for row in features.as_ref() {
>         for cell in row.as_ref() {
>             print!("{} ", cell);
>         }
>         println!("");
>     }
> }
> ```
> 
> `AsRef` ?!?
> -----------
> 
> This is great and it works(if I replaced `f32` above with `char`), but I had no idea how this piece of code worked, mainly because I was clueless about what `AsRef` actually did. So I opened the docs to enlighten myself, and came across [this definition](https://doc.rust-lang.org/std/convert/trait.AsRef.html).
> 
> ```
> pub trait AsRef<T>
> where
>     T: ?Sized,
> {
>     fn as_ref(&self) -> &T;
> }
> ```
> 
> > Used to do a cheap reference-to-reference conversion.
> > 
> > This trait is similar to `AsMut` which is used for converting between mutable references. If you need to do a costly conversion it is better to implement From with type &T or write a custom function.
> 
> I don't know about you but as an (advanced) beginner that was not enlightening at all. So I took out the [Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) book to see if they had a better explanation for what `AsRef` is supposed to do.
> 
> And they did. Copied verbatim from Chapter 12 ([2nd Edition](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) O'Reilly):
> 
> > When a type implements `AsRef<T>`, that means you can borrow a `&T` from it efficiently.
> > 
> > ...
> > 
> > So, for example, `Vec<T>` implements `AsRef<[T]>`, and `String` implements `AsRef<str>`. You can also borrow a `String`’s contents as an array of bytes, so `String` implements `AsRef<[u8]>` as well. `AsRef` is typically used to make functions more flexible in the argument types they accept.
> > 
> > For example, the `std::fs::File::open` function is declared like this:
> > 
> > `fn open<P: AsRef<Path>>(path: P) -> Result<File>`
> > 
> > What `open` really wants is a `&Path`, the type representing a filesystem path. But with this signature, open accepts anything it can borrow a `&Path` from—that is, anything that implements `AsRef<Path>`. Such types include String and str, the operating system interface string types `OsString` and `OsStr`, and of course `PathBuf` and `Path`; see the library documentation for the full list. This is what allows you to pass string literals to open:
> > 
> > `let dot_emacs = std::fs::File::open("/home/jimb/.emacs")?;`
> > 
> > ...
> > 
> > For callers, the effect resembles that of an overloaded function in C++, although Rust takes a different approach toward establishing which argument types are acceptable.
> 
> Armed with this knowledge, the `batch()` function starts to make sense:
> 
> ```
> fn batch<Matrix: AsRef<[Row]>, Row: AsRef<[f32]>>(features: Matrix) {
>        // -- snip --
>    }
> ```
> 
> So `Matrix` is your bog standard generic parameter with a trait bound that states that it could be any type that implements `AsRef<[Row]>`. And `Row` is another generic parameter that is bound by `AsRef<[f32]>` - basically any type that implements `AsRef<[f32]>` and therefore can return `&[f32]` when you call `as_ref` on the type. In my mental model, I started thinking of this as "function overloading" - closest thing to inheritence in Rust.
> 
> What about `Borrow`?
> --------------------
> 
> Awesome, but then I began to wonder how `AsRef` compares with `Borrow` which I (perhaps incorrectly) assumed does something similar. I opened the [docs](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) on `Borrow` and it says the following:
> 
> ```
> pub trait Borrow<Borrowed>
> where
>     Borrowed: ?Sized,
> {
>     fn borrow(&self) -> &Borrowed;
> }
> ```
> 
> > `Trait std::borrow::Borrow` A trait for borrowing data.
> > 
> > //- snip --
> > 
> > These types provide access to the underlying data through references to the type of that data. They are said to be ‘borrowed as’ that type. For instance, a `Box<T>` can be borrowed as `T` while a `String` can be borrowed as `str`.
> 
> Huh! So when should I use `Borrow` vs `AsRef`?
> 
> I opened my trusty "Programming Rust" book again and found a slightly better explanation. Excerpt from from Chapter 12 ([2nd Edition](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) O'Reilly):
> 
> > The `std::borrow::Borrow` trait is similar to `AsRef`: if a type implements `Borrow<T>`, then its borrow method efficiently borrows a `&T` from it. But `Borrow` imposes more restrictions: a type should implement `Borrow<T>` only when a `&T` hashes and compares the same way as the value it’s borrowed from. (Rust doesn’t enforce this; it’s just the documented intent of the trait.)
> > 
> > This makes `Borrow` valuable in dealing with keys in hash tables and trees or when dealing with values that will be hashed or compared for some other reason.
> > 
> > This distinction matters when borrowing from `String`s, for example: `String` implements `AsRef<str>`, `AsRef<[u8]>`, and `AsRef<Path>`, but those three target types will generally have different hash values. Only the `&str` slice is guaranteed to hash like the equivalent `String`, so String implements only `Borrow<str>`.
> 
> ..and finally:
> 
> > `Borrow` is designed to address a specific situation with generic hash tables and other associative collection types.
> 
> Basically, `Borrow` imposes additional restrictions on `T` so that it can be used as keys in hash tables, trees etc.
> 
> This made things slightly clearer but the docs and the stub definition leave out a very important piece of information for `Borrow` which I eventually found in [Jon Gjengset](https://twitter.com/jonhoo?lang=en)s book - [Rust for Rustacean](https://rust-for-rustaceans.com/):
> 
> Excerpt from Chapter 3:
> 
> > Specifically, `Borrow` is tailored for a much narrower use case: allowing the caller to supply any one of multiple essentially identical variants of the same type.
> > 
> > **It could, perhaps, have been called `Equivalent` instead**. For example, for a `HashSet<String>`, `Borrow` allows the caller to supply either a `&str` or a `&String`. While the same could have been achieved with `AsRef`, **that would not be safe without `Borrow`’s additional requirement that the target type implements `Hash`, `Eq`, and `Ord` exactly the same as the implementing type**.
> 
> This bit 👆 is key to understand the real difference between `Borrow` and `AsRef`.
> 
> > Borrow also has a blanket implementation of `Borrow<T>` for `T`, `&T`, and `&mut T`, which makes it convenient to use in trait bounds to accept either owned or referenced values of a given type. In general, `Borrow` is intended only for when your type is essentially equivalent to another type, whereas `Deref` and `AsRef` are intended to be implemented more widely for anything your type can “act as.”
> 
> And what about `From` and `Into`?
> ---------------------------------
> 
> If you've read so far and are now wondering -"Well, when do I implement `From` and `Into` traits because they also seem to do "conversion""? The important thing for `From` and `Into` is that they
> 
> > ..represent conversions that **consume** a value of one type and return a value of another. Whereas `AsRef` and `AsMut` traits borrow a reference of one type from another, `From` and `Into` take ownership of their argument, transform it, and then return ownership of the result back to the caller.
> > 
> > -- Programming Rust - Blandy, Jim; Orendorff, Jason; Tindall et al
> 
> Another difference is that `AsRef`(and `AsMut`) conversions are expected to be cheap - i.e. they don't require any data copying or allocation of new memory and in _most cases_ performed in constant time O(1), whereas `From` and `Into` conversions are not guaranteed to be cheap. It's not part of their contract.
> 
> One place where `From` and `Into` traits really shine is when we use the `?` operator. You should implement them (usually the `From<T>` trait) for your error types.
> 
> Enter ChatGPT
> -------------
> 
> I could end my post here, and honestly if you understand all of the above you can stop reading now (thanks for reading btw and make sure you buy the books linked here - I don't get any commission for referals).
> 
> I took the dog out for a walk after my "research" on this topic, and was mulling over the concepts when I decided that I had to look at the `String` implementation of `AsRef<str>` and `Borrow<str>` in order to solidify my understanding of `Borrow` and `AsRef`.
> 
> The implementations look like this:
> 
> ```
> // -- AsRef
> impl AsRef<str> for String {
>     #[inline]
>     fn as_ref(&self) -> &str {
>         self
>     }
> }
> 
> // -- Borrow
> impl Borrow<str> for String {
>     #[inline]
>     fn borrow(&self) -> &str {
>         &self[..]
>     }
> ```
> 
> Now instead of trying to reason why `as_ref()` returns `self` and `borrow()` returns `&self[..]` I chose to ask [ChatGPT](https://chat.openai.com/) because its 2023, and I was not disappointed by the outcome.
> 
> > First the `borrow` code: ![](https://rusty-ferris.pages.dev/img/borrow.png)
> > 
> > Then `as_ref`: ![](https://rusty-ferris.pages.dev/img/asref.png)
> > 
> > Explain why? ![](https://rusty-ferris.pages.dev/img/diff.png)
> 
> Looks reasonable to me. So +1 for Generative AI 🤖, unless I've missed the mistake(s) in ChatGPTs explanation. Perhaps a keen-eyed reader and an experienced Rustacean can let me know.
> 
> CONCLUSION
> ----------
> 
> That is all; I hope you found it as illuminating as I did. If you find any errors or mistakes, please reach out to me on [twitter](https://twitter.com/anup) or via email (anup[dot]jadhav[at]gmail[dot]com). Until next time.
> 
> * * *
> 
> > This work is licensed under [CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/).