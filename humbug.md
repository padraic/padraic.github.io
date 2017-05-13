Lies, Damned Lies and Code Coverage: Towards Mutation Testing
=============================================================

About Humbug
------------

Humbug is a Mutation Testing framework for PHP. Essentially, it injects deliberate defects into your source code, designed to emulate programmer errors, and then checks whether your unit tests notice. If they notice, good. If they don’t notice, bad. All quite straightforward. Humbug will log which defects were not noticed by your unit tests, complete with diffs, and provide some basic metric scores so that you can fuel your Github badge mania someday.

You can try out Humbug today, though it remains a work in progress (PHPUnit only) and certain combinations of code, tests and moon phase may result in “issues”. Do try it though, it’s polishing up very nicely and I’m looking forward to a stable release. The readme has more information.

This article however is mostly reserved to explain why I wrote Humbug, and why Mutation Testing is so badly needed in PHP. There’s a few closing words on Mutation Testing performance which has traditionally been a concern impeding its adoption.

Code Coverage
-------------

Code Coverage is a measure of how many lines of code a unit test suite has executed. Not tested. Executed. If lines of code are not executed then, it logically follows, they were not executed by the test suite. This might be bad. There are probably tests missing. Off to the editor with you!

This distinction between testing and executing is all important, and something I feel that we’ve lost sight of in PHP when we’re busy decorating our Github pages with nice green 100% badges and talking about imposing 100% Code Coverage.

Let’s imagine that you have 100% Code Coverage. That’s actually a lie. More specifically, you actually have 100% Line Coverage. PHP_CodeCoverage and XDebug are incapable, at this time, of measuring Statement Coverage, Branch Coverage, and Condition Coverage. Your 100% score is only 25% of the story. Let’s call it 10% because I’m mean and there are other forms of Coverage that I have not mentioned.

Your Code Coverage is now 10%.

You know, I think I was too generous, and I’ve arbitrarily assigned a score. This simply will not do and its unfair to those who, in reality, might have 11% Code Coverage. We’ll have to take a more scientific approach.

Your Code Coverage is now 0% pending scientific research and peer review.

We can rephrase all of the above as follows: Line Coverage is an indicator of where source code was definitely not executed by any test. It does not indicate that a line was tested, or even fully exercised, merely that something on that line was executed at least once.

Taking this at face value, we can invent a problem to provide more illumination on how Line Code Coverage can mislead us:
```
if ($i >= 5) {
    // do something
}
```
The above is a condition where there are three possible outcomes. $i will either be greater than 5, equal to 5 or less than 5. Two of these possibilities will evaluate the expression to true, the other to false. This suggests that we need 3 tests – one for each of the outcomes. We also need to be very specific. What if an error changes the 5 to a 6? Testing if 10 passes would be a bad test. What if it were changed to a 4? Then not testing with values of 4 or 5 would make for bad tests. It’s not all random integers we want in such tests – their selection should be deliberately targeting the boundary of the condition so as to avoid writing overly positive tests that are unlikely to ever fail.

Writing just one test that executes the above line will still leave us two tests short of where we should be. How do we know when those two tests are missing? Line Code Coverage will give us a 100% percent score for writing between 0 and 33% of the expected effective tests.

Dave Marshall recently wrote about Code Coverage with [another real life example](http://davedevelopment.co.uk/2015/01/07/probing-test-suite-quality-with-mutation-testing.html).

Line Code Coverage in PHP is simply not fit for our purposes. Being the sole possible Code Coverage type in PHP at present does not excuse it from being a misleading, inflated, and overly trusted metric that is easily fooled by writing bad tests and relying on coincidental execution.

The more insidious problem is that relying solely on Code Coverage as a measure of test quality, which is what we often end up doing, is attempting to automate an intellectual task. You cant simply run a magic report and leave your brain at home. Your brain is very much required when assessing test suite effectiveness.

Measuring Unit Test Quality
---------------------------

Above, I made a distinction between code that was executed and code that was tested. Code coverage is an assertion that code is executed. It’s entirely possible to attain 100% Code Coverage, yet test absolutely nothing at all. The probable methods of achieving this are through tests which make no assertions, positive tests with long odds of failure, and coincidental execution by tests not specifically targeting the line (see PHPUnit’s @covers annotation).

This needn’t be intentional! It’s quite easy to overlook tests and that’s why we use Code Coverage to help us identify missing tests. We just can’t rely it as our sole means of ensuring test existence. Better Code Coverage would help us find a lot more missing tests, but it’s still solely a measure of execution.

So, given a test suite with 100% Line Coverage, how can we examine the test suite and arrive at any conclusion as to its quality and effectiveness in preventing regressions?

This is where Mutation Testing shines.

Mutation Testing
Imagine our original example:
```
if ($i >= 5) {
    // do something
}
```
During Mutation Testing, Humbug would introduce three subtle defects, i.e. mutations. It would mutate the “>=” to each of “>” and “<”. It would also mutate the “5” to “6”. Depending on the nature of the code block, this should result in unexpected behaviour that your unit tests, if written well, should have assertions against. Occasionally, a mutation is equivalent to the original statement (e.g. perhaps $i is hardcoded to >5 and it’s not actually settable from a test) but we would expect the false positive rate to be minimal.

For each mutation, noting that only one is applied at a time, we run the relevant unit tests. If a defect causes a test failure, error or a timeout (infinite loops may occur infrequently with a mutation) then we can assert that this particular defect is tested. If the tests all pass, we can assert that this defect was not tested and we can log it for investigation. A new test would now be needed to cover that defect unless, of course, it’s a provable false positive.

We are no longer playing games with execution statistics. We’re actually measuring the effectiveness of a test suite, and improving its effectiveness over time. The provided scores, taken with a double pinch of salt, assist in gauging how bad or good defect detection is by calculating the ratio of detected mutations to the total generated and the total covered by tests (yes, we contrast to Code Coverage). The logs are the more valuable output, offering diffs for each undetected mutation. These can be examined (by an actual living entity) to see where new tests might be needed.

Your Code Coverage metrics essentially tell me nothing about the effectiveness of your unit test suite. They only tell me that your unit tests executed stuff. Your Mutation Testing scores, on the other hand, give me some ballpark estimates on the real effectiveness of those same tests.

Performance
-----------

I can’t sign off without mentioning Mutation Testing performance.

Traditionally, Mutation Testing has been ridiculously slow, often running the entire test suite for every single mutation. On one library this morning, I generated close to 1000 mutations. The test suite typically took 5 seconds to run. Doing the math is close to crazy. The solution implemented by Humbug was to take something I criticised (ahem, Code Coverage) and use its data to only run tests which execute the mutated line. It takes around 2 minutes for Mutation Testing of that library. In another example, a library with ~5000 tests running in 3 minutes took around 12 minutes to mutation test (~1.5k mutations were generated).

I expect to improve on that even more and enable specific class targeting as a future feature. It would be even faster if we had improved Code Coverage in PHP. And, as always, your mileage will most definitely vary – performance is influenced by the mutation count and the performance of both code and tests. Slow tests, in particular, while ordered to run last may have a significant impact.

Tools like Humbug are no longer restricted to academic papers.

I bring this up, because performance is clearly one huge reason why Mutation Testing hasn’t already become commonplace despite its very obvious benefits. You won’t be mutation testing all the time, but running it occasionally for your entire test suite, or at least a few times for each new testable class, is quite reasonable and within current reach. Implementing filters and other focus aids, would allow for even more dynamic and regular usage alongside your testing framework to keep feedback regular and fast.