---
layout: post
title:  "My first PR for the Rust compiler"
date:   2015-06-20 19:39:57
categories: rust compiler
author: "Paul Faria"
---

This is the first post for my new blog. I've somewhat recently started learning [Rust][rust] and contributing to the community (not just Rust, but [Servo][servo] as well). I'd like to document what went into making my first change to the compiler. It wasn't my first commit to the project, I managed to accidentally score a spot on the Rust 1.0 contributors list for fixing typos in the documentation. I've also helped out on issue [#22709][rust-22709].

[rust]:       http://www.rust-lang.org/
[servo]:      https://github.com/servo/servo/wiki
[rust-22709]: https://github.com/rust-lang/rust/issues/22709

It started off with me wanting to learn how the Rust compiler works. I came upon the conclusion that this would be easiest by fixing some of the issues tagged with E-Easy on their GitHub page. I found that a lot of the more recent ones had a lot of people commenting/working/mentoring already, so I decided to search for the oldest easy issues. I came across [#2392][rust-2392], which was over 3 years old at the time. It had even changed over time as changes to the compiler were made.

[rust-2392]: https://github.com/rust-lang/rust/issues/2392

The general idea at the time I started was to make this code:

{% highlight rust linenos %}
struct Cat {
    x: int
}

trait Meow {
    fn mew();
}

impl Cat: Meow {
    fn mew() {}
}

fn main() {
    let kitty = Cat { x:5 };
    assert_eq!(kitty.x(), 5);
}
{% endhighlight %}

output this error:

{% highlight text %}
<anon>:17:19: 17:22 error: type `Cat` does not implement any method in scope named `x`
<anon>:17     assert!(kitty.x() == 5);
                            ^~~
<anon>:17:19: 17:22 note: did you mean to use `kitty.x`?
<anon>:17     assert!(kitty.x() == 5);
                            ^~~
{% endhighlight %}

instead of this error:
{% highlight text %}
<anon>:17:19: 17:22 error: type `Cat` does not implement any method in scope named `x`
<anon>:17     assert!(kitty.x() == 5);
                            ^~~
<anon>:17:19: 17:22 note: use `(s.x)(...)` if you meant to call the function stored in the `x` field
<anon>:17     assert!(kitty.x() == 5);
                            ^~~
{% endhighlight %}

I also wanted to make the note ```use `(s.x)(...)` if you meant to call the function stored in the `x` field``` look more like ```use `(kitty.x)(...)` if you meant to call the function stored in the `x` field```. (Note the `s` => `kitty` change)

The code I needed to change was the below section of the `report_error` function from the file `src/librustc_typeck/check/method/suggest.rs`.

{% highlight rust %}
 pub fn report_error<'a, 'tcx>(fcx: &FnCtxt<'a, 'tcx>,
                              span: Span,
                              rcvr_ty: Ty<'tcx>,
                              item_name: ast::Name,
                              rcvr_expr: Option<&ast::Expr>,
                              error: MethodError)
 {
     // avoid suggestions when we don't know what's going on.
     if ty::type_is_error(rcvr_ty) {
         return
     }
 
     match error {
         MethodError::NoMatch(static_sources, out_of_scope_traits, mode) => {
             let cx = fcx.tcx();
 
             fcx.type_error_message(
                 span,
                 |actual| {
                     format!("no {} named `{}` found for type `{}` \
                              in the current scope",
                             if mode == Mode::MethodCall { "method" }
                             else { "associated item" },
                             item_name,
                             actual)
                 },
                 rcvr_ty,
                 None);
 
             // If the item has the name of a field, give a help note
             if let (&ty::TyStruct(did, _), Some(_)) = (&rcvr_ty.sty, rcvr_expr) {
                let fields = ty::lookup_struct_fields(cx, did);
                if fields.iter().any(|f| f.name == item_name) {
                    cx.sess.span_note(span,
                        &format!("use `(s.{0})(...)` if you meant to call the \
                                 function stored in the `{0}` field", item_name));
                }
            }
            
            ...
    }
}
{% endhighlight %}

Updates to come...