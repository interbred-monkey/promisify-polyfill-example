# Promisify explanation

_Disclaimer_: I'm not sure how the guys at Node are doing it but if I had to do it, this is how I would go about it. This example leverages native promises and therefore will require **Node v0.12.18** and higher.

## How to run
This is a very simple example, you can see it in action by executing:

```
node main.js
```

## Using the library

Let's first take a look at `main.js` and try to understand what is going on there.

We include our library `const promisify = require('./promisify.js');` and define our asynchronous function that we wish to make into a promise.

```
function hasCallback(p1, p2, callback) {

  setTimeout(() => {

    console.log('Executing...');
    console.log(p1);
    console.log(p2);

    return callback(null, {'p1': p1, 'p2': p2});

  }, 1000);

}
```

In this instance all we are doing here is displaying messages after a timeout, depending on the values that have been supplied to the function. This could be anything that is asynchronous, like returning data from a database.

We then move onto initialising the asynchronous function into a a promise by passing it to our library.

`let asPromised = promisify(hasCallback);`

This will then return us the same function but as a promise, allowing us to use it as though it was already a promise function.

```
asPromised('foo', 'bar')
.then((data) => {

  console.log(JSON.stringify(data, null, 2));
  console.log(something);
  console.log(other);
  console.log('I kept my promise');

})
.catch(console.log);
```

As you can see I am passing it two values that will be passed through to the original function `'foo', 'bar'` and when running the example we will see the output:

```
Executing...
foo
bar
{
  "p1": "foo",
  "p2": "bar"
}
I kept my promise
```

With that being said we need to address, how to make it reject. Let's change up the `hasCallback` function so that it will return an error if the value of p1 is `foo`. This would look something like:

```
function hasCallback(p1, p2, callback) {

  setTimeout(() => {

    console.log('Executing...');
    console.log(p1);
    console.log(p2);

    if (p1 === 'foo') {

      return callback('I had my fingers crossed');

    }

    return callback(null, {'p1': p1, 'p2': p2});

  }, 1000);

}
```

Now when we execute the function with the same values it should reject right? Here is the output:

```
Executing...
foo
bar
I had my fingers crossed
```

You can see that we didn't even get to the `then` function, because it was rejected.

## How promisify works

This is the interesting part, let's look at the file as a whole first:

```
function promisify(fn) {

  return function() {

    return new Promise((resolve, reject) => {

      this.fn(...arguments, function(err, data) {

        if (!_.isNull(err)) {

          return reject(err);

        }

        return resolve(data);

      })

    })

  }.bind({fn: fn});

}

module.exports = promisify;
```

Yes, it does look a little scary, but it's actually very simple. To understand what is going on here, I will break it down step by step.

First we need to cross reference the two files. In `main.js` we pass the asynchronous function to the promisify function:

```
let asPromised = promisify(hasCallback);
```

In the promisify function this is then referenced by `fn`.

```
function promisify(fn) {
```

The next thing we do is to return a _"new"_ function and bind the _"old"_ asynchronous function to it. This means, when the _"new"_ function executes it will be able to access an instance of the original function.

```
return function() {

...

}.bind({fn: fn});
```

Put more simply, we are creating a new function that will execute the old one.

When we execute the function...

```
asPromised('foo', 'bar')
```

we are actually executing the one that returns the promise:

```
function() {

  return new Promise((resolve, reject) => {

```

The next thing we need to address is that we are passing parameters to the function, how are they accessed?

A function* that is executed has an arguments parameter, which is global to that function, a bit like `this`. For example:

```
function a(p1, p2) {

  console.log(arguments === [p1, p2]);

}

a('foo', 'bar');
```

The above would output `true`, because the parameters passed to the function are put into the arguments array as well as being assigned to the variables in the function declaration.

In our example we use the arguments array so that we can call the originally defined function (`fn`), supplying it with the same parameters it is expecting.

If you you remember we bound `fn` to the new function so we can access it by the `this` parameter. We can also de-structure the arguments array so that they are passed into the function declaration variables by using the spread operator.

```
this.fn(...arguments, function(err, data) {
```

Everything here on in is self explanatory...

If there is an error returned from the function, reject with the error, if there is no error resolve the promise.

Writing this normally would look something like this:

```
fn('foo', 'bar', function(err, data) {

  if (!_.isNull(err)) {

    // do something about the error

  }

  // wooooo bacon

})
```