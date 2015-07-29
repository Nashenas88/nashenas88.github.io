---
layout: post
title:  "My First PR for the Rust compiler"
date:   2015-07-28 19:39:57
categories: "rust"
tags: "rust compiler"
author: "Paul Faria"
---

This is the first post for my new blog. I've somewhat recently started learning [Rust][rust] and contributing to the community (not just Rust, but [Servo][servo] as well). I'd like to document what went into making my first change to the compiler. It wasn't my first commit to the project, I managed to accidentally score a spot on the Rust 1.0 contributors list for fixing typos in the documentation. I've also helped out on issue [#22709][rust-22709].

[rust]:       http://www.rust-lang.org/
[servo]:      https://github.com/servo/servo/wiki
[rust-22709]: https://github.com/rust-lang/rust/issues/22709

It started off with me wanting to learn how the Rust compiler works. I came upon the conclusion that this would be easiest by fixing some of the issues tagged with [E-Easy][rust-easy-issues] on their GitHub page. I found that a lot of the more recent ones had a lot of people commenting/working/mentoring already, so I decided to search for the oldest easy issues. I came across [#2392][rust-2392], which was over 3 years old at the time. It had even changed over time as changes to the compiler were made.

[rust-easy-issues]: https://github.com/rust-lang/rust/issues?q=is%3Aopen+is%3Aissue+label%3AE-easy
[rust-2392]: https://github.com/rust-lang/rust/issues/2392

The general idea at the time I started was to make this code:

{% highlight rust linenos %}
struct Dog {
    x: usize
}

trait Bark {
    fn woof();
}

impl Bark for Dog {
    fn woof() {}
}

fn main() {
    let dog = Dog { x:5 };
    assert_eq!(dog.x(), 5);
}
{% endhighlight %}

output this error:

{% highlight text %}
<anon>:15:17: 15:20 error: type `Dog` does not implement any method in scope named `x`
<anon>:15     assert!(dog.x() == 5);
                          ^~~
<anon>:15:17: 15:20 note: did you mean to use `dog.x`?
<anon>:15     assert!(dog.x() == 5);
                          ^~~
{% endhighlight %}

instead of this error:
{% highlight text %}
<anon>:15:17: 15:20 error: type `Dog` does not implement any method in scope named `x`
<anon>:15     assert!(dog.x() == 5);
                          ^~~
<anon>:15:17: 15:20 note: use `(s.x)(...)` if you meant to call the function stored in the `x` field
<anon>:15     assert!(dog.x() == 5);
                          ^~~
{% endhighlight %}

I also wanted to make the note ```use `(s.x)(...)` if you meant to call the function stored in the `x` field``` look more like ```use `(dog.x)(...)` if you meant to call the function stored in the `x` field```. (Note the `s` => `dog` change). You can play around with this example [here](http://is.gd/xgxsjW).

The code I needed to change was the below section of the `report_error` function from the file `src/librustc_typeck/check/method/suggest.rs`.

{% include hide_last_line_number.html %}
{% include line_number_offset.html offset=31 %}
{% highlight rust linenos %}
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
{% endhighlight %}

Let's break this down a little. We're in the `suggest.rs` file under the parent directory of `method`. The `report_error` function is used to output suggestions when the compiler comes across some kind of error related to methods. On this line:

{% include line_number_offset.html offset=39 %}
{% highlight rust linenos %}
if ty::type_is_error(rcvr_ty) {
    return
}
{% endhighlight %}

We check if the receiver type is an error (this means there was an issue parsing the `abc` in `abc.xyz()` or the `(a.x())` in `(a.x()).bar()`. There's no additional suggestions that can be made here since we can't lookup the type to give better hints.

Moving on, we're going to match on the error enum, which is of type [`MethodError`](method_error). For this issue, we're only interested in the `NoMatch` enumeration. We also don't care about the values that are pattern matched out (`static_sources`, `out_of_scope_traits`, and `mode`), since they're used for other purposes besides our issue at hand.

[method_error]: https://github.com/rust-lang/rust/blob/a5c12f4e39d32af3c951b66bd2839bc0b5a1125b/src/librustc_typeck/check/method/mod.rs#L35-L44

{% include line_number_offset.html offset=47 %}
{% highlight rust linenos %}
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
{% endhighlight %}
Here we're using the function context to output an error. There are some issues with multilingual translations with this chunk of code, but I'll get to that in another post. There's nothing here I want to focus on.

{% include line_number_offset.html offset=61 %}
{% highlight rust linenos %}
if let (&ty::TyStruct(did, _), Some(_)) = (&rcvr_ty.sty, rcvr_expr) {
    let fields = ty::lookup_struct_fields(cx, did);
    if fields.iter().any(|f| f.name == item_name) {
        cx.sess.span_note(span,
            &format!("use `(s.{0})(...)` if you meant to call the \
                     function stored in the `{0}` field", item_name));
    }
}
{% endhighlight %}
This is where our focus is really needed. This is where we're going to decide what suggestions to display after the error. We're pattern matching against the definition id (shortened to `did` in the code) of the receiver type (if the receiver type is a struct) and the existance of the receiver expression.

{% include line_number_offset.html offset=62 %}
{% highlight rust linenos %}
let fields = ty::lookup_struct_fields(cx, did);
{% endhighlight %}
The call here is using the current context, `cx`, along with the struct definition id to lookup all of the fields that belong to the struct with that definition id (The context stores a mapping of these ids to the different structs available from the current scope). Then, we're going to iterate over all of the fields and see if their name matches the name of the method we attempted to call. If the compiler was on a line like `(a.x()).y()`, and `y` is what didn't match, then `did` would be the definition id of the struct that's returned from the call to `a.x()`, `recvr_expr` would be `Some` expression `(a.x())`, and `item_name` would be the string `"y"`. If one such field is found, then a very generic suggestion is output as a note. It assumes that the user intended to call a function, without verifying if the field can be used as a function at all.

My fix for the issue (along with lots of help from eddyb on [irc][irc] (#rust)) is below (the difference in the initial line numbers is due to `use` statements, which I didn't feel the need to include).

[irc]: https://wiki.mozilla.org/IRC

{% include hide_last_line_number.html %}
{% include line_number_offset.html offset=34 %}
{% highlight rust linenos %}
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
            if let (&ty::TyStruct(did, substs), Some(expr)) = (&rcvr_ty.sty, rcvr_expr) {
                let fields = ty::lookup_struct_fields(cx, did);

                if let Some(field) = fields.iter().find(|f| f.name == item_name) {
                    let expr_string = match cx.sess.codemap().span_to_snippet(expr.span) {
                        Ok(expr_string) => expr_string,
                        _ => "s".into() // Default to a generic placeholder for the
                                        // expression when we can't generate a string
                                        // snippet
                    };

                    let span_stored_function = || {
                        cx.sess.span_note(span,
                                          &format!("use `({0}.{1})(...)` if you meant to call \
                                                    the function stored in the `{1}` field",
                                                   expr_string, item_name));
                    };

                    let span_did_you_mean = || {
                        cx.sess.span_note(span, &format!("did you mean to write `{0}.{1}`?",
                                                         expr_string, item_name));
                    };

                    // Determine if the field can be used as a function in some way
                    let field_ty = ty::lookup_field_type(cx, did, field.id, substs);
                    if let Ok(fn_once_trait_did) = cx.lang_items.require(FnOnceTraitLangItem) {
                        let infcx = fcx.infcx();
                        infcx.probe(|_| {
                            let fn_once_substs = Substs::new_trait(vec![infcx.next_ty_var()],
                                                                   Vec::new(),
                                                                   field_ty);
                            let trait_ref = ty::TraitRef::new(fn_once_trait_did,
                                                              cx.mk_substs(fn_once_substs));
                            let poly_trait_ref = trait_ref.to_poly_trait_ref();
                            let obligation = Obligation::misc(span,
                                                              fcx.body_id,
                                                              poly_trait_ref.as_predicate());
                            let mut selcx = SelectionContext::new(infcx, fcx);

                            if selcx.evaluate_obligation(&obligation) {
                                span_stored_function();
                            } else {
                                span_did_you_mean();
                            }
                        });
                    } else {
                        match field_ty.sty {
                            // fallback to matching a closure or function pointer
                            ty::TyClosure(..) | ty::TyBareFn(..) => span_stored_function(),
                            _ => span_did_you_mean(),
                        }
                    }
                }
            }
...
{% endhighlight %}

This is noticeably larger... the first change is here:
{% include line_number_offset.html offset=64 %}
{% highlight rust linenos %}
if let (&ty::TyStruct(did, substs), Some(expr)) = (&rcvr_ty.sty, rcvr_expr) {
    let fields = ty::lookup_struct_fields(cx, did);
{% endhighlight%}
I've pattern matched the `substs` and `expr`. The `expr` is explained previously (we're just unwrapping the `recvr_expr`), so we're only going to go into what this new `substs` thing is.

Let's say we have the following method definition:
{% highlight rust %}
<T as Trait<A, B>>::method::<X, Y>
{% endhighlight %}
The `Subst` for this corresponds to `[A, B], Self=T, [X, Y]`, which, at the lowest level, is a `vec![A, B, T, X, Y]` and a couple of indices splitting the vector into 3 pieces (again, thanks to [eddyb](https://botbot.me/mozilla/rust-internals/2015-06-15/?msg=41918422&page=9)).

So, now this makes it clear what a `TyStruct` contains. A definition id that can be used to lookup some (potentially generic) struct, along with a list of types that can fill in all of those generics slots.

Moving on to the next `if`:
{% include line_number_offset.html offset=67 %}
{% highlight rust linenos %}
if let Some(field) = fields.iter().find(|f| f.name == item_name) {
{% endhighlight %}
Rather than just checking if a field with a matching name exists, I'm going to find one that exists and store it in the `field` variable. If there's none, then there's no suggestion to be made, just like the code before the change.

{% include line_number_offset.html offset=67 %}
{% highlight rust linenos %}
let expr_string = match cx.sess.codemap().span_to_snippet(expr.span) {
    Ok(expr_string) => expr_string,
    _ => "s".into() // Default to a generic placeholder for the
                    // expression when we can't generate a string
                    // snippet
};
{% endhighlight %}
Here I'm generating a string to print out the `(a.x())` in `(a.x()).y())`. The span in `expr.span` represents a section of a string with two indices that represent the beginning and the end of the section. The string in question is the code that's being parsed by the compiler. This string lives in the codemap of the context's session. We can use the codemap's `span_to_snippet` method to convert the expression's span into the string that represents the expression. If this fails for any reason, we're going to default back to the old code's beloved `s`. (I really have no idea what the `s` signifies, it was there before, so I decided to keep it for backup).

{% include line_number_offset.html offset=75 %}
{% highlight rust linenos %}
let span_stored_function = || {
    cx.sess.span_note(span,
                      &format!("use `({0}.{1})(...)` if you meant to call \
                                the function stored in the `{1}` field",
                               expr_string, item_name));
};

let span_did_you_mean = || {
    cx.sess.span_note(span, &format!("did you mean to write `{0}.{1}`?",
                                     expr_string, item_name));
};
{% endhighlight %}
These are just closures to write the two span notes that can possible be output from here on. One of the two will be displayed at this point.

{% include line_number_offset.html offset=88 %}
{% highlight rust linenos %}
let field_ty = ty::lookup_field_type(cx, did, field.id, substs);
{% endhighlight %}
Here we're looking up the field type. This will be used later on. It represents the concrete struct of the field.

{% include line_number_offset.html offset=89 %}
{% highlight rust linenos %}
if let Ok(fn_once_trait_did) = cx.lang_items.require(FnOnceTraitLangItem) {
{% endhighlight %}
Here we try and require the Rust provided `FnOnce` trait language item. We're going to try and use this to determine if the field implements the `FnOnce` trait. If we're able to obtain the trait definition id, then we can move forward with the more solid approach (this should occur most of the time).

{% include line_number_offset.html offset=90 %}
{% highlight rust linenos %}
let infcx = fcx.infcx();
infcx.probe(|_| {
    let fn_once_substs = Substs::new_trait(vec![infcx.next_ty_var()],
                                           Vec::new(),
                                           field_ty);
{% endhighlight %}
The call to `probe`, is necessary because the call to `next_ty_var` on line 93 will modify the inferred context (`infctx`). The `probe` call will allow us to scope that change, and when the probe ends, the changes will be reverted.

{% include line_number_offset.html offset=92 %}
{% highlight rust linenos %}
let fn_once_substs = Substs::new_trait(vec![infcx.next_ty_var()],
                                       Vec::new(),
                                       field_ty);
let trait_ref = ty::TraitRef::new(fn_once_trait_did,
                                  cx.mk_substs(fn_once_substs));
let poly_trait_ref = trait_ref.to_poly_trait_ref();
let obligation = Obligation::misc(span,
                                  fcx.body_id,
                                  poly_trait_ref.as_predicate());
let mut selcx = SelectionContext::new(infcx, fcx);

if selcx.evaluate_obligation(&obligation) {
    span_stored_function();
} else {
    span_did_you_mean();
}
{% endhighlight %}
This block is determining if the field type implements `FnOnce`. If it does, then we're going to display the stored function suggestion to the user. If not, we're going to recommend removing the parenthesis.

{% include line_number_offset.html offset=109 %}
{% highlight rust linenos %}
} else {
    match field_ty.sty {
        // fallback to matching a closure or function pointer
        ty::TyClosure(..) | ty::TyBareFn(..) => span_stored_function(),
        _ => span_did_you_mean(),
    }
}
{% endhighlight %}
Let's roll back to line 90 and consider the rare case that we can't get the `FnOnce` trait definition id. We're going to try and match the field's type. If it's a closure or a bare function type, then we display the stored function message, otherwise we recommend removing the parenthesis.

{% highlight rust linenos %}
#![feature(unboxed_closures)]
struct Dog {
    x: usize,
	c: Barking,
}

struct Barking;
impl Barking {
    fn woof(&self) -> usize {
        3
    }
}

impl Fn<()> for Barking {
    extern "rust-call" fn call(&self, _: ()) -> usize {
        self.woof()
    }
}

impl FnMut<()> for Barking {   
    extern "rust-call" fn call_mut(&mut self, _: ()) -> usize {
        self.woof()
    }
}

impl FnOnce<()> for Barking {
    type Output = usize;
    extern "rust-call" fn call_once(self, _: ()) -> usize {
        self.woof()
    }
}

fn main() {
    let dog = Dog { x:5, c:Barking };
    let _ = dog.x();
    let _ = dog.c();
}
{% endhighlight %}

Now, after this change, attempting to compile the above code will produce the below error.

{% highlight text %}
<anon>:35:17: 35:20 error: no method named `x` found for type `Dog` in the current scope
<anon>:35     let _ = dog.x();
                          ^~~
<anon>:35:17: 35:20 note: did you mean to write `dog.x`?
<anon>:35     let _ = dog.x();
                          ^~~
<anon>:36:17: 36:20 error: no method named `c` found for type `Dog` in the current scope
<anon>:36     let _ = dog.c();
                          ^~~
<anon>:36:17: 36:20 note: use `(dog.c)(...)` if you meant to call the function stored in the `c` field
<anon>:36     let _ = dog.c();
                          ^~~
error: aborting due to 2 previous errors
{% endhighlight %}

You can play with this example [here](http://is.gd/EnHRWF).

**Note**: After I started writing this post, but before it was published, I was mentioned in issue [#26472](26472). After reviewing that issue, I realized there's a bug in my code, in that it will recommend private fields to a user. The fix for that is more complicated than you would think (I'd have to take the private scope into account, and currently, that's in another module in the compiler, in a completely separate pass).

[26472]: https://github.com/rust-lang/rust/issues/26472
