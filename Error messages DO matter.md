# Bad error messages

The term UX is also applicable to the process of software development.
And error messages are quite important.

Search for this text:

> Module not found: Error: Can't resolve ...

and discover how many people have hit this problem, which probably won't occur, if message was augmented with this:

> Maybe you meant the <em>relative</em> path rather than the module.

----

Another example is [here](https://github.com/dotnet/fsharp/issues/9490#issue-640921496): Let's name function `Seq.take`
similar to `Enumerable.Take`, but change its semantics, so it throws, if sequence has fewer than `n` elements. 
 
Again, error message contains no hints, but it could do better by adding:
 
> Consider 'Seq.truncate' function to get no more than `n` elements.
