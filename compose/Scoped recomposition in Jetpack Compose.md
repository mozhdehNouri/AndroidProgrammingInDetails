
```kotlin
@Composable fun Foo() {
  var text by remember { mutableStateOf("") }

  Button(onClick = { text = "$text\n$text" }) {
    Text(text)
  }
}
```


**Recompose scopes:**

Recompose scopes are an important piece of the Compose puzzle. They do some bookkeeping and help reduce the amount of work Compose has to do to prepare a frame.

- They are the smallest unit of a composition that can be re-executed (recomposed) to update the underlying tree.
- They keep track of what snapshot-based State objects are read inside of them, and get invalidated when those states change.

For every non-inline composable function that returns Unit, the Compose compiler generates code that wraps the function’s body in a recompose scope. When a recompose scope is invalidated, the compose runtime will ensure the (entire) function body gets recomposed (reexecuted) before the next frame.
Foo’s body, Button’s body, the content lambda we pass to Button, Text’s body, all get their own recompose scopes.

**What about inline functions and functions that return a value?**
Inline functions are inline - their bodies are effectively copied into their caller, so they share their caller’s recompose scope.
Functions that have a non-unit return value don’t get their own scopes because they can’t be re-executed without also re-executing their caller, so their caller can “see” the new return value computed by the function.

Why isn’t just Text recomposed?
If we think of the text read as happening on its own line just above the Text call, then look for the nearest recompose scope enclosing that - it’s the button content lambda. That’s the recompose scope that gets invalidated when the text state value is changed.

So when text is changed, and the lambda’s recompose scope is invalidated, the entire lambda body gets recomposed for the next frame.

Why isn't all of Foo recomposed?
Now that we know how recompose scopes are delimited and invalidated, we can see that nothing directly in Foo reads any State, so it will never actually recompose.

This might be more apparent if we get rid of the delegate syntax for text, which doesn't change behavior but hides what's actually going on a bit:
```kotlin
@Composable fun Foo() {
  val text: MutableState<String> = remember { mutableStateOf("") }

  Button(onClick = { text.value = "${text.value}\n${text.value}" }) {
    Text(text.value)
  }
}
```

It's a lot more clear now that all Foo does with text is create the MutableState value holder. The actual object referenced by text doesn't change, it always points to the same MutableState instance. When we talk about state "reads", we mean reads of the State's value property – e.g. text.value.

Why isn't Button recomposed?
Because Foo isn't recomposed, nothing calls the Button function itself on recomposition.
However, this is a bit of a trick question: There may or may not be functions deep inside Button's implementation that recompose in certain cases, but that doesn't affect our code, so we can ignore those. Button does maintain some private state, so it can show its pressed status and the ripple, but they might not actually require recomposition, instead they can just request re-draws.

**What about the onClick lambda?**
Recompose scopes are only created around composable functions. Event handlers, like Button's onClick, are not composable, they're just regular functions. When the click handler is invoked by the framework, it is done so outside of any recompose scope.

**Caveat: Inline composable functions**
You might be surprised to find out (and I often forget) that common layouts like Column, Row, and Box are all inline functions. Even the core Layout composable itself is inline. Ignoring Compose for a second, remember that when a Kotlin function is marked inline, its body is always copied into its call sites by the compiler – it doesn't even exist as an entity in the bytecode. This is most often used in regular Kotlin code to avoid allocating objects to hold lambda functions for functions that are simple wrappers around other functions.

Compose uses inlining for a similar optimization. Because the body of inline composable functions are simply copied into their call sites, such functions do not get their own recompose scopes. In the following snippet, when the lambda passed to Wrapper gets recomposed, the App body will be recomposed as well, since Wrapper is inline and does not have its own recompose scope. In other words, any time this code prints "App recomposing", "Wrapper recomposing", or "Lambda recomposing", all three will be printed together.

```kotlin
@Composable fun App() {
  println("App recomposing")
  Wrapper {
    println("Lambda recomposing")
    // Read some state causing recomposition…
  }
}

@Composable inline fun Wrapper(content: @Composable () -> Unit) {
  println("Wrapper recomposing")
  content()
}
```
If you wrap the Text in the original code snippet in a Column and add trace statements, you’ll see that the invalidated scope is still the entire button content lambda, not just the Column content lambda.

**Recomposition Scope:**

In order to understand how Compose is able to optimize recompositions, it is important to take into account the scopes of the composable functions that we are using in our examples.
Compose keeps track of these composable scopes under-the-hood and divides a Composable function into these smaller units for more efficient recompositions. It then tries its best to only recompose the scopes that are reading the values that can change. In order to wrap our heads around what this means, let's use this lens and look at both our examples again. This time, we'll take into account the scopes that are available for the Compose runtime to do its book-keeping.
![[Pasted image 20240326141941.png]]

We see that there's a couple lambda scopes at play in the first example i.e the scope of the MyComponent function and the scope of CustomText function. Furthermore, CustomText is in the lambda scope of the MyComponent function. When the value of the the counter changes, we previously noticed that both these scopes were being reinvoked and here's why -

CustomText is recomposed because its text parameter changed as it includes the counter value. This makes sense and is probably what you want anyway.
MyComponent is recomposed because its lambda scope captures the counter state object and a smaller lambda scope wasn't available for any recomposition optimizations to kick in.
Now you might wonder what I meant when I said "a smaller lambda scope wasn't available". Hopefully the next example will make this clear!
![[Pasted image 20240326142037.png]]
What does a donut have to do with all this?
Let's get to why you opened this article in the first place - what do donuts have to do with all this? I'm about to say something that is going to blow your mind. Composable functions can be thought to be made up of donuts that are internally made up of smaller donuts 🍩. At least that's the metaphor the Compose team has been using to describe the optimizations related to recompositions. The composable function itself can be though to represent the donut, whereas its scope is the donut hole. Whenever possible, the Compose runtime skips running the "donut" if its not reading the value that changed (assuming its input didn't change either) and will only run the "donut-hole" (i.e its scope, assuming that's where the value is being read).

Let's visualize the composables in example 2 with this lens and see how they are being recomposed. Anything with the chequered pattern represents that it was recomposed.
![[Pasted image 20240326142325.png]]
![[Pasted image 20240326142357.png]]
**![[Pasted image 20240326142405.png]]