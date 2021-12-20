# Events

In the previously mentioned examples, you have seen callbacks mostly, and although that is one way of handling events, FLTK offers multiple ways to handle events:
- We can use the set_callback() method, which is automatically triggered with a click to our button.
- We can use the handle() method for fine-grained event handling.
- We can use the emit() method which takes a sender and a message, this allows us to handle events in the event loop.
- We can define our own event, which can be handled within another widget's handle method.

### Setting the callback
Part of the WidgetExt trait is the set_callback method:
```rust
use fltk::{prelude::*, *};

fn main() {
    let app = app::App::default();
    let mut my_window = window::Window::new(100, 100, 400, 300, "My Window");
    let mut but = button::Button::new(160, 200, 80, 40, "Click me!");
    my_window.end();
    my_window.show();
    but.set_callback(|_| println!("The button was clicked!"));
    app.run().unwrap();
}
```

```rust
use fltk::{prelude::*, *};

fn main() {
    let app = app::App::default();
    let mut my_window = window::Window::new(100, 100, 400, 300, "My Window");
    let mut but = button::Button::new(160, 200, 80, 40, "Click me!");
    my_window.end();
    my_window.show();
    but.set_callback(|b| b.set_label("Clicked!"));
    app.run().unwrap();
}
```
The set_callback() methods have default triggers varying by the type of the widget. For buttons it's clicking or pressing enter when the button has focus.
This can be changed using the set_trigger() method. For buttons this might not make much sense, however for input widgets, the trigger can be set to "CallbackTrigger::Changed" and this will cause changes in the input widget to trigger the callback.
```rust
use fltk::{prelude::*, *};

fn main() {
    let a = app::App::default();
    let mut win = window::Window::default().with_size(400, 300);
    let mut inp = input::Input::default()
        .with_size(160, 30)
        .center_of_parent();
    win.end();
    win.show();
    inp.set_trigger(enums::CallbackTrigger::Changed);
    inp.set_callback(|i| println!("{}", i.value()));
    a.run().unwrap();
}
```
This will print on every character input by the user.

### Using the handle method
The handle method takes a closure whose parameter is an Event, and returns a bool for handled events. The bool lets FLTK know whether the event was handled or not.
The call looks like this:
```rust
use fltk::{prelude::*, *};

fn main() {
    let app = app::App::default();
    let mut my_window = window::Window::new(100, 100, 400, 300, "My Window");
    let mut but = button::Button::new(160, 200, 80, 40, "Click me!");
    my_window.end();
    my_window.show();

    but.handle(|_, event| {
        println!("The event: {:?}", event);
        false
    });
    
    app.run().unwrap();
}
```
This prints the event, and doesn't handle it since we return false. Obviously we would like to do something useful, so change the handle call to:
```rust
    but.handle(|_, event| match event {
        Event::Push => {
            println!("I was pushed!");
            true
        },
        _ => false,
    });
```
Here we handle the Push event by doing something useful then  returning true, all other events are ignored and we return false.

Another example:
```rust
    but.handle(|b, event| match event {
        Event::Push => {
            b.set_label("Pushed");
            true
        },
        _ => false,
    });
```

### Using messages
This allows us to create channels and a Sender and Receiver structs, we can then emit messages (which have to be Send + Sync safe) to be handled in our event loop. The advantage is that we avoid having to wrap our types in smart pointers when we need to pass them into closures or into spawned threads.
```rust
use fltk::{prelude::*, *};

fn main() {
    let app = app::App::default();
    
    let mut my_window = window::Window::new(100, 100, 400, 300, "My Window");
    let mut but = button::Button::new(160, 200, 80, 40, "Click me!");
    my_window.end();
    my_window.show();

    let (s, r) = app::channel();
    
    but.emit(s, true);
    // This is equivalent to calling but.set_callback(move |_| s.send(true)); 

    while app.wait() {
        if let Some(msg) = r.recv() {
            match msg {
                true => println!("Clicked"),
                false => (), // Here we basically do nothing
            }
        }
    }
}
```
Messages can be received in the event loop like in the previous example, otherwise you can receive messages in a background thread or in app::add_idle()' s callback:
```rust
    app::add_idle(move || {
        if let Some(msg) = r.recv() {
            match msg {
                true => println!("Clicked"),
                false => (), // Here we basically do nothing
            }
        }
    });
```

You're also not limited to using fltk channels, you can use any channel. For example, this uses the std channel:
```rust
let (s, r) = std::sync::mpsc::channel::<Message>();
btn.set_callback(move |_| {
    s.send(Message::SomeMessage).unwrap();
});
```

You can also define a method which applies to all widgets, similar to the emit() method:
```rust
use std::sync::mpsc::Sender;

pub trait SenderWidget<W, T>
where
    W: WidgetExt,
    T: Send + Sync + Clone + 'static,
{
    fn send(&mut self, sender: Sender<T>, msg: T);
}

impl<W, T> SenderWidget<W, T> for W
where
    W: WidgetExt,
    T: Send + Sync + Clone + 'static,
{
    fn send(&mut self, sender: Sender<T>, msg: T) {
        self.set_callback(move |_| {
            sender.send(msg.clone()).unwrap();
        });
    }
}

fn main() {
    let btn = button::Button::default();
    let (s, r) = std::sync::mpsc::channel::<Message>();
    btn.send(s.clone(), Message::SomeMessage);
}
```

### Creating our own events
FLTK recognizes 29 events which are listed in enums::Event. However it allows us to create our own events using the app::handle(impl Into<i32>, window) call. The handle function takes an arbitrary i32 (> 30) value as a signal, ideally the values should be predefined, which can be handled within another widget's handle() method, the other widget needs to be within the window that was passed to app::handle.
In the following example, we create a window with a frame and a button. The button's callback sends a CHANGED Event through the app::handle_main function. The CHANGED signal is queried in the frame's handle method.
```rust
use fltk::{app, button::*, enums::*, frame::*, group::*, prelude::*, window::*};
use std::cell::RefCell;
use std::rc::Rc;

pub struct MyEvent;

impl MyEvent {
    const CHANGED: i32 = 40;
}

#[derive(Clone)]
pub struct Counter {
    count: Rc<RefCell<i32>>,
}

impl Counter {
    pub fn new(val: i32) -> Self {
        Counter {
            count: Rc::from(RefCell::from(val)),
        }
    }

    pub fn increment(&mut self) {
        *self.count.borrow_mut() += 1;
        app::handle_main(MyEvent::CHANGED).unwrap();
    }

    pub fn decrement(&mut self) {
        *self.count.borrow_mut() -= 1;
        app::handle_main(MyEvent::CHANGED).unwrap();
    }

    pub fn value(&self) -> i32 {
        *self.count.borrow()
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let app = app::App::default();
    let counter = Counter::new(0);
    let mut wind = Window::default().with_size(160, 200).with_label("Counter");
    let mut pack = Pack::default().with_size(120, 140).center_of(&wind);
    pack.set_spacing(10);
    let mut but_inc = Button::default().with_size(0, 40).with_label("+");
    let mut frame = Frame::default()
        .with_size(0, 40)
        .with_label(&counter.clone().value().to_string());
    let mut but_dec = Button::default().with_size(0, 40).with_label("-");
    pack.end();
    wind.end();
    wind.show();

    but_inc.set_callback({
        let mut c = counter.clone();
        move |_| c.increment()
    });

    but_dec.set_callback({
        let mut c = counter.clone();
        move |_| c.decrement()
    });
    
    frame.handle(move |f, ev| {
        if ev == MyEvent::CHANGED.into() {
            f.set_label(&counter.clone().value().to_string());
            true
        } else {
            false
        }
    });

    Ok(app.run()?)
}
``` 
The sent i32 signal can be created on the fly, or added to a const local or global, or within an enum. 

#### Advantages:
- No overhead.
- The signal is dealt with like any fltk event.
- the app::handle function returns a bool which indicates whether the event was handled or not.
- Allows handling of custom signals/events outside the event loop.
- Allows an MVC or SVU architecture to your application.

#### Disadvantages:
- The signal can only be handled in a widget's handle method.
- The signal is inaccessible within the event loop (for that, you might want to use WidgetExt::emit or channels described previously in this page). 

### fltk-evented crate
For an alternative `on_<event>` method of handling events, check the [fltk-evented crate](https://crates.io/crates/fltk-evented) whose Listener widget can handle events in the event loop, so you can avoid sending things into closures and worrying about lifetimes:
```rust
use fltk::{app, button::Button, enums::Color, frame::Frame, group::Flex, prelude::*, window::Window};
use fltk_evented::Listener;

fn main() {
    let a = app::App::default().with_scheme(app::Scheme::Gtk);
    app::set_font_size(20);

    let mut wind = Window::default()
        .with_size(160, 200)
        .center_screen()
        .with_label("Counter");
    let flex = Flex::default()
        .with_size(120, 160)
        .center_of_parent()
        .column();
    let mut but_inc: Listener<_> = Button::default().with_label("+").into();
    let mut frame = Frame::default();
    let mut but_dec: Listener<_> = Button::default().with_label("-").into();
    flex.end();
    wind.end();
    wind.show();

    but_inc.on_hover(|b| {
        b.set_color(Color::Red);
    });

    let mut val = 0;

    while a.wait() {
        if but_inc.triggered() {
            val += 1;
        }

        if but_inc.hovered() {
            but_inc.set_color(Color::White);
        }

        if but_inc.left() {
            but_inc.set_color(Color::BackGround);
        }

        if but_dec.triggered() {
            val -= 1;
        }

        if but_dec.hovered() {
            but_dec.set_color(Color::White);
        }

        if but_dec.left() {
            but_dec.set_color(Color::BackGround);
        }

        frame.set_label(&val.to_string());
    }
}
```