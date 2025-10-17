---
title: Tailwind-style syntax for Zed’s gpui framework
description: Tailwind-style syntax and composable styling for Zed’s gpui framework
date: 2025-10-17
tags: Rust, gpui, zed, Tailwind
---

Lately I’ve been exploring Zed’s [`gpui`](https://github.com/zed-industries/zed/tree/main/crates/gpui) framework, it uses builder-style APIs for building UI elements in Rust and feels like writing React in Rust.

In the Webdev world, [Tailwind CSS](https://tailwindcss.com/) has gained popularity for its declarative, composable styling using classes like — `size-10`, `text-blue-500`, `p-2`, `m-4` so on.
In gpui each of these become functions like `div().size_10().color(rgba::Blue).padding_2().margin_4()`. When your UI grows in complexity, managing such long chains can quickly become noisy and repetitive.

Tailwind is used in frameworks to have "drop-in" components, for example [Shadcn UI](https://ui.shadcn.com/). The framework provides you components as independent files that you drop into your project, and customize to your needs. The index file will quite often have a bunch of variants and tailwind classes defined for each of them, e.g [`shadcn-vue/button`](https://github.com/unovue/shadcn-vue/blob/dev/apps/v4/registry/new-york-v4/ui/button/index.ts#L6-L35)

```Typescript
export const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground shadow-xs hover:bg-primary/90",
        destructive: "bg-destructive text-white",
        outline: "border bg-background hover:bg-accent hover:text-accent-foreground",
      },
      size: {
        default: "h-9 px-4 py-2 has-[>svg]:px-3",
        sm: "h-8 rounded-md gap-1.5 px-3 has-[>svg]:px-2.5",
        lg: "h-10 rounded-md px-6 has-[>svg]:px-4",
        icon: "size-9",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  },
);
```

I attempt to have a similar macro for tailwind classes in `gpui`, such that the styles can be concisely specified yet compile-time checked. I will use the [`comptime`](https://docs.rs/comptime/latest/comptime/) crate and procedural macros to achieve this.


## Tailwind-style syntax in gpui

In `gpui`, you normally style elements using method chains:

```Rust
div()
    .size_10()
    .color(rgba::Blue)
    .padding_2()
    .margin_4()
```

With `gpui_twind`, you can instead write:

```Rust
div().twist(cx, twind!("size-10 color@blue padding-2 margin-4"))
```


## The idea: twind! and twist()

The main concept revolves around two pieces - the `twind!` macro which allows us to create a "reusable-style" that can be applied to any element using the `twist` trait.

The code I want:

```Rust
fn render(self, _window: &mut Window, cx: &mut App) -> impl IntoElement {
    // STYLES
    let button_style_primary = twind!("bg@primary-bg border-1 border-color@g-white");
    let button_style_destructive = twind!("bg@destructive-bg");
    let btn = div().twist(cx, twind!("flex flex-col gap-3 size-20"));

    // Apply style based on prop
    let btn = if self.style == ButtonStyle::Primary {
        btn.twist(cx, button_style_primary)
    } else {
        btn.twist(cx, button_style_destructive)
    };
}
```

Assume a `Twist` trait that can be implemented for any styled element:

```Rust
pub trait Twist<'a>: gpui::Styled {
    fn twist(self, cx: &App, style: impl TwistGen<Self>) -> Self {
        style.apply(cx, self)
    }
}
```

Note that this means we need to have another trait `TwistGen<Self>` that can be used to borrow the context for applying styles. We can generate anonymous structs implementing this trait from the `twind!` macro.

For example:

```Rust
let button_style_primary = twind!("bg@primary-bg border-1 border-color@g-white");
```

expands (via cargo expand) into:

```Rust
let button_style_primary = {
    #[derive(Clone, Copy)]
    struct TwistGenInner;

    impl<T: gpui::Styled> gpui_twind::TwistGen<T> for TwistGenInner {
        fn apply(&self, app: &App, element: T) -> T {
            use gpui::Styled;
            let theme = app.theme();
            element.bg(theme.primary_bg)
                .border_1()
                .border_color(gpui::white())
        }
    }

    TwistGenInner {}
};
```

`TwistGenInner` is an anonymous struct here and what stays important is that this struct implements the `TwistGen` trait.
This means when we pass it to the `twist` method, it can call the `apply` method to apply styles to the element.


## Themes and flexibility

Notice the use of `theme = app.theme()` inside the macro, which can be used to resolve colors or styles dynamically. We can support it by just relying on the fact that a trait adding `.theme()` to the `App` is implemented and in context.
This gives us a very "loose-contract" theme which isn't super great, but it should allow mixing and matching components from different boilerplate repos and then defining the required theme variables.


## Under the hood: comptime + procedural macros

Implementing the `twind!` macro is pretty simple using `comptime` as it basically reads like templating code. The full implementation can be found in the [repo](https://github.com/meetparikh7/gpui_twind), but the interesting bits are:

```Rust
#[crabtime::expression]
#[macro_export]
fn twind(input: String) {
    // `flex flex-col gap-3` becomes `.flex().flex_col().gap_3()`
    let input = input.replace("-", "_").map(|e| format!(".{}()", e));

    crabtime::output! {
        #[derive(Clone,Copy)]
        struct TwistGenInner;
        impl<T: gpui::Styled> gpui_twind::TwistGen<T> for TwistGenInner
        {
            fn apply(&self, app: &App, element: T) -> T {
                use gpui::Styled;
                let theme = app.theme();
                // This line becomes `element.flex().flex_col().gap_3()`
                element{{ input }}
            }
        }
        TwistGenInner {}
    }
}
```

## Conclusion
The broader takeaway is that this pattern can be used to create "reusable initializers" for "builder-style" patterns.
`gpui_twind` was an experiment of bringing the ergonomics of Tailwind into the safety of Rust.
