---
layout: post
title:  "My First PR for the Rust compiler"
date:   2015-06-20 19:39:57
categories: "rust"
tags: "rust compiler"
author: "Paul Faria"
---

**This is a work in progress. It is not yet complete.**

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
<anon>:15:19: 15:22 error: type `Cat` does not implement any method in scope named `x`
<anon>:15     assert!(kitty.x() == 5);
                            ^~~
<anon>:15:19: 15:22 note: did you mean to use `kitty.x`?
<anon>:15     assert!(kitty.x() == 5);
                            ^~~
{% endhighlight %}

instead of this error:
{% highlight text %}
<anon>:15:19: 15:22 error: type `Cat` does not implement any method in scope named `x`
<anon>:15     assert!(kitty.x() == 5);
                            ^~~
<anon>:15:19: 15:22 note: use `(s.x)(...)` if you meant to call the function stored in the `x` field
<anon>:15     assert!(kitty.x() == 5);
                            ^~~
{% endhighlight %}

I also wanted to make the note ```use `(s.x)(...)` if you meant to call the function stored in the `x` field``` look more like ```use `(kitty.x)(...)` if you meant to call the function stored in the `x` field```. (Note the `s` => `kitty` change)

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

we check if the receiver type is an error (this means there was an issue parsing the `abc` in `abc.xyz` or the `(a.x())` in `(a.x()).bar()`. There's no additional suggestions that can be made here since we can't lookup the type to give better hints.

Moving on, we're going to match on the error enum, which is of type `MethodError`. For this issue, we're only interested in the `NoMatch` enumeration. We also don't care about the values that are pattern matched out (`static_sources`, `out_of_scope_traits`, and `mode`), since they're used for other purposes besides our issue at hand.

In the following code:
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
we're going to use the function context to output an error. There are some issues with multilingual translations with this chunk of code, but I'll get to that in another post.

The `if` expression here:
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
is where the meat of our issue lies. This is where we're going to decide what suggestions to display after the error. We're pattern matching the id of the receiver type's struct, and the existance of the receiver expression.

The `ty::lookup_struct_fields` call on line 63 is using the current context, along with the struct definition id to lookup all of the fields that belong to the struct with that definition id. Then, we're going to iterate over all of the fields and see if their name matches the name of the method we attempted to call. If the compiler was on a line like `(a.x()).y()`, and `y` is what didn't match, then `did` would be the definition id of the struct that's returned from the call to `a.x()`, `recvr_expr` would be `Some` expression `(a.x())`, and `item_name` would be the string `"y"`. If one such field is found, then a very generic suggestion is output as a note. It assumes that the user intended to call a function, without verifying if the field is a function type at all. My fix for the issue (along with lots of help from eddyb on [irc](irc) (#rust)) is below (the difference in line numbers is due to `use` statements, which I didn't feel the need to include).

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

This is noticeably larger... the first change comes on line 65. I've pattern matched the `substs` and `expr`. The `expr` is explained previously (we're just unwrapping the `recvr_expr`), so we're only going to go into what this new `substs` thing is.

Let's say we have the following method definition:
{% highlight rust %}
<T as Trait<A, B>>::method::<X, Y>
{% endhighlight %}
The `Subst` for this corresponds to `[A, B], Self=T, [X, Y]`, which, at the lowest level, is a `vec![A, B, T, X, Y]` and a couple of indices splitting the vector into 3 pieces (again, thanks to [eddyb](https://botbot.me/mozilla/rust-internals/2015-06-15/?msg=41918422&page=9)).

So, now this makes it clear what a `TyStruct` contains. A definition id that can be used to lookup some (potentially generic) struct, along with a list of types that can fill in all of those generics slots.

Moving on to line 68, rather than just checking if a field with a matching name exists, I'm going to find one that exists and store it in the `field` variable. If there's none, then there's no suggestion to be made, just like the code before the change.

On line 69, I'm generating a string to print out the `(a.x())` in `(a.x()).y())`. The span in `expr.span` represents a section of a string with two indices, the beginning, and the end of the section. The string in question is the code that's being parsed by the compiler. This string lives in the codemap of the context's session. We can use the codemap's `span_to_snippet` method to convert the expression's indices into the string that represents the expression. If this fails for any reason, we're going to default back to the old code's beloved `s`. (I really have no idea what the `s` signifies, it was there before, so I decided to keep it for backup).

Lines 76 through 86 are just closures to the two span notes that can possible be output from here on. One of the two will be displayed at this point.

On line 89, we're looking up the field type. This will be used later on. It represents the concrete struct of the field.

On line 90, we try and require the Rust provided `FnOnce` trait language item. We're going to try and use this to determine if the field implements the `FnOnce` trait. If we're able to obtain the trait definition id, when we can move forward with the more solid approach (this should occur most of the time).

Line 92, the call to `probe`, is necessary because the call to `next_ty_var` on line 93 will modify the inferred context (`infctx`). The `probe` call will allow us to scope that change. When the probe ends, the changes will be reverted.

Lines 93 through 104 are determining if the field type implements `FnOnce`. If it does, then we're going to display the stored function suggestion to the user. If not, we're going to recommend removing the parenthesis.

Let's roll back to line 90 and consider the rare case that we can't get the `FnOnce` trait definition id. We're going to try and match the field's type. If it's a closure or a bare function type, then we display the stored function message, otherwise we recommend removing the parenthesis. 