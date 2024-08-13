---
layout: post
title: Understanding the Rust Iced GUI Application Architecture and Implementing a Sandbox Example
tags: [Rust, Iced]
---

Over the past few weeks, I've been exploring GUI app development in Rust and discovered that Iced is a sleek and minimalistic framework ideal for creating lightweight GUI applications. In this post, I'll dive into the architecture of Iced and build a minimal GUI app using this framework.

## Architecture

In a GUI, some widgets, like buttons, are interactive, while others, like labels, are not. When a button is pressed, it triggers an interaction that can drive certain actions in the backend, such as updating the text displayed in a label. This interaction is stateful because it alters the widget's state.

There are three elements in our user interface:

- **Widgets** — the visual elements of an interface.
- **Interactions** — the actions that may be triggered by some widgets.
- **State** — the underlying information of an interface.

The elements connect with each other which forms a close feedback loop.

> **Widgets** produce **interactions** when the user interacts with them. These **interactions** then change the **state** of the application. The changed **state** propagates and dictates the new widgets that must be displayed. 

![The GUI Trinity]({{ "/assets/img/posts/understanding-the-iced-gui-application-architecture-and-implementing-a-sandbox-example/the-gui-trinity.svg" | relative_url}})

 This concept is heavily inspired by The Elm Architecture. The structure of an Iced application borrows terminology from The Elm Architecture to describe its core components:

* **Model** — the state of the application.
* **Messages** — the interactions of the application.
* **Update logic** — how the messages change the state.
* **View logic** — how the state dictates the widgets.

These ideas will become clearer as we walk through the implementation of our "Hello World" GUI app.

You should read the Iced author's explanation of the architecture [here](https://book.iced.rs/architecture.html). The previous section is my own interpretation.

## Implementation

Let's create a minimal iced app.

```bash
$ cargo new --bin iced-simple-sandbox-app
```

Add iced dependency the cargo toml file.

```conf
[package]
name = "iced-simple-sandbox-app"
version = "0.1.0"
edition = "2021"

[dependencies]
iced = {version = "0.12.1"}
```

The app features a label that displays the value of a variable (state) and includes two buttons: 'Increment' and 'Decrement.' When the user presses the 'Increment' button, the variable's value increases by 1. Similarly, pressing the 'Decrement' button decreases the value by 1. 

In Iced terminology, our model encapsulates the application's state, which is the variable that is incremented or decremented.

```rust
struct AppState {
    index: i32,
}
```

The messages correspond to user interactions with the application, such as pressing the "Increment" or "Decrement" buttons.

```rust
#[derive(Debug, Clone)]
enum Message {
    IncrementButtonPressed,
    DecrementButtonPressed,
}
```

Iced provides a trait called `Sandbox`, which our application will implement. The Iced runtime uses this trait to call the appropriate functions and run the application. Below is the `Sandbox` trait.

Note that the `Sandbox` trait supports minimal features. There's also a trait called `Application`, which offers more flexible capabilities, such as running asynchronous operations. We'll cover `Application` in a future article.

```rust
pub trait Sandbox {
    type Message: Debug + Send;

    // Required methods
    fn new() -> Self;
    fn title(&self) -> String;
    fn update(&mut self, message: Self::Message);
    fn view(&self) -> Element<'_, Self::Message>;

    // Provided methods
    fn theme(&self) -> Theme { ... }
    fn style(&self) -> Application { ... }
    fn scale_factor(&self) -> f64 { ... }
    fn run(settings: Settings<()>) -> Result<(), Error>
       where Self: 'static + Sized { ... }
}
```
- The `new()` method is called when the application is initialized.
- The `title()` method is invoked every time the UI is updated. It returns a `String` that serves as the application's title.
- The `update()` method is triggered whenever a user interaction occurs, with the `Message` parameter indicating the type of interaction. This updates the state of the application.
- The `view()` method is called when the application first runs or when the user interacts with it. This returns a generic type `Element`. `Element` is composable. We will talk about this in some other article. 

The other methods in the `Sandbox` trait have default implementations, so we don't need to provide them. However, I've included an implementation of the `theme()` method because a dark theme adds a sleek, modern touch.

The state diagram is as follows
![Iced state diagram]({{ "/assets/img/posts/understanding-the-iced-gui-application-architecture-and-implementing-a-sandbox-example/Iced-sandbox.png" | relative_url}})

Let's implement the application.

```rust
impl Sandbox for AppState {
    type Message = Message;

    fn new() -> AppState {
        println!("{:?} Sandbox::new()", std::thread::current().id());
        Self {
            index: 0,
        }
    }

    fn title(&self) -> String {
        println!("{:?} Sandbox::title()", std::thread::current().id());
        String::from("AppState")
    }

    fn update(&mut self, message: Self::Message) {
        println!("{:?} Sandbox::update({:?})", std::thread::current().id(), message);
        self.index = match message {
            Message::IncrementButtonPressed => {self.index + 1},
            Message::DecrementButtonPressed => {self.index - 1},
        };
    }

    fn view(&self) -> Element<Self::Message> {
        println!("{:?} Sandbox::view()", std::thread::current().id());
        let incerement_btn = button(
            "Increment"
        )
        .width(150)
        .on_press(Message::IncrementButtonPressed);

        let label_text = text("Value:").width(75);
        let index_text = text(self.index.to_string()).width(75);

        let decrement_btn = button(
            "Decrement"
        )
        .width(150)
        .on_press(Message::DecrementButtonPressed);

        let content = column![
            incerement_btn,
            row![label_text, index_text,],
            decrement_btn,
        ]
        .width(Length::Fill)
        .align_items(Alignment::Center)
        .spacing(10);

        container(scrollable(content))
            .width(Length::Fill)
            .height(Length::Fill)
            .center_x()
            .center_y()
            .into()
    }
}
```

The application looks like this
![Sandbox app screenshot]({{ "/assets/img/posts/understanding-the-iced-gui-application-architecture-and-implementing-a-sandbox-example/sandbox-app-screenshot.jpg" | relative_url}})

The code has been committed to the repository: [https://github.com/asit-dhal/iced-simple-sandbox-app](https://github.com/asit-dhal/iced-simple-sandbox-app).

Thanks for reading!