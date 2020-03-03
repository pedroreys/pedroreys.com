---
title: 'Creating a Linq Expression to represent a &ldquo;Right&rdquo; function in C#'
author: Pedro
type: post
date: 2011-07-27T00:57:20+00:00
draft: true
private: true
aliases:
 - /2011/07/27/creating-a-linq-expression-to-represent-a-right-function-in-c
categories:
  - .Net

---
I’m currently working on a project that have an external DSL that is than parsed into [Linq Expressions][1]. This DSL contains a “Right” function that behaves similar to the [T-SQL RIGHT function][2].

As C# doesn&#8217;t have a function like that, I had to create a expression tree that represent a code similar to this:

<noscript>
  <pre><code class="language-c# c#">public string Right(string value, int targetLenght)
{
  var startIndex = value.Length - targetLenght;
  return value.Substring(startIndex, targetLenght);
}</code></pre>
</noscript>

Given the snippet above, the Expression Tree needs to have:

  * 2 Constants (the value and the targetLength)
  * A Subtract Expression to represent the start index calculation
  * A Call Expression to represent the “Substring” method call.

Now that the expression is described, is just a matter of witting it using Linq Expressions:

<noscript>
  <pre><code class="language-c# c#">public Expression Right(string value, int targetLenght)
{
  var valueConstant = Expression.Constant(value);
  var targetLenghtConstant = Expression.Constant(targetLenght);
  var pi = typeof(String).GetProperty("Length");
  var mi = typeof(string).GetMethod("Substring", new[] { typeof(int), typeof(int) });

  var valueLenght = Expression.Property(valueConstant, pi);
  var startIndex = Expression.Subtract(valueLenght, targetLenghtConstant);
  var rightExpression =  Expression.Call(valueConstant, mi, startIndex, targetLenght);
  return rightExpression;
}
</code></pre>
</noscript>

A quick test to ensure the expression can be compiled into a executable code that works as expected:

<noscript>
  <pre><code class="language-c# c#">[Test]
public void Should_evaluate_a_Right_function()
{
  var rightExpression = ExpressionBuilder.Right();
  Func&lt;string,int,string&gt; right = Expression.Lambda&lt;Func&lt;string,int,string&gt;&gt;(rightExpression).Compile();
  right("Some String", 5).ShouldEqual("tring");
}
</code></pre>
</noscript>

I hope this gives you a glimpse of what can be accomplished using the Linq Expression API. Go check it out and start playing with it.

 [1]: http://msdn.microsoft.com/en-us/library/bb506649.aspx
 [2]: http://msdn.microsoft.com/en-us/library/ms177532.aspx
