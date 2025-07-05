# Kyle d'Oliveira - Fibonacci Funhouse: Exploring Ruby Algorithms for Fibonacci Numbers

Mathematics is often very powerful and elegant. Do you know what else is powerful and elegant? Ruby.

This article is a deep dive into the world of Fibonacci numbers to show some really neat tools that Ruby provides. Some may not be used very often but it is always good to have them in your toolbox.

So using Ruby, what is the largest Fibonacci number we can calculate quickly?

## Understanding the Fibonacci Sequence

In case you are not familiar, the Fibonacci sequence is a classic mathematical progression where each number is the sum of the previous two. Starting with 0 and 1, the sequence looks like this:

$$0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...$$

## Basic Recursive Fibonacci Algorithm and Its Limitations

The most straightforward method to compute Fibonacci numbers is a simple recursive function:

```ruby
def fibonacci(n)
 return 0 if n == 0
 return 1 if n == 1


 fibonacci(n - 1) + fibonacci(n - 2)
end
```

This algorithm is simple and clear. However, if we look at the performance of this, we can see that it starts out fine, but once we cross the 30th Fibonacci number, the performances starts to get exponentially worse.

![Graph 1](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%201.png)

This is happens because we are doing a lot of redundant calculations. Imagine calculating the 5th Fibonacci number. In order to calculate that, we need the 4th and the 3rd numbers. To calculate the 4th number, we need the 3rd and the second. This pattern repeats until we hit the 0th or the 1st number. The larger the number is, the more computations we will need to perform.

## Using Binet’s Formula for Constant Time Calculation

It turns out there is a close-form expression for calculating the nth Fibonacci number.
To overcome recursive inefficiency, we turn to Binet’s formula, a closed-form expression for the nth Fibonacci number:

$$F_n = \frac{\phi^n - \psi^n}{\sqrt{5}}$$

Where:

- $\phi =  \frac{1 + \sqrt{5}}{2}$, (phi) the golden ratio
- $\psi = \frac{1 - \sqrt{5}}{2}$, (psi) its conjugate

This formula allows computation in constant time, a significant performance leap. Implementing it in Ruby involves calculating powers of irrational numbers and rounding the result.

In fact, since $\psi^n$ is so small, we can further simplify this formula to:

$$F_n = \Bigl\lfloor\frac{\phi^n}{\sqrt{5}}\Bigr\rceil$$

### Closed form algorithm

```ruby
SQRT_FIVE = Math.sqrt(5)
PHI = (1 + SQRT_FIVE) / 2


def fibonacci(n)
 ((PHI**n)/SQRT_FIVE).round
end
```

This is a constant time algorithm according to our performance graphs.

![Graph 2](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%202.png)

However, this algorithm faces two major challenges:

1. **Precision limits:** At the 71st Fibonacci number this algorithm will return a value of `608061521170130` however, the real 71st Fibonacci number is `608061521170129` just one less.

This happens because Ruby does not have infinite precision so at some point there are rounding issues. As the algorithm calculates larger and larger Fibonacci numbers, this difference will only become bigger and bigger.

2. **Float domain errors:** At 1475, this algorithm end up erroring with a `FloatDomainError` .

This happens in Ruby because, at some point, there will not be enough precision and Ruby will give up and return Infinity. The exact exponent this happens on can vary number to number. For example, you can see for $\sqrt{5}$ if you raise that to the power of 883, it will return Infinity. Numeric method such as `round` on Infinity will raise a `FloatDomainError`.

```ruby
Math.sqrt(5) ** 883
=> Infinity
```

Can we do better?

### Closed form algorithm #2

Ruby’s `BigDecimal` library offers arbitrary precision decimal arithmetic, allowing us to mitigate rounding errors by specifying how many decimal points to calculate.

By replacing standard floating-point operations with `BigDecimal`, we can compute Fibonacci numbers with much greater accuracy and avoid early overflow issues.

```ruby
SQRT_FIVE = BigDecimal("5").sqrt(1000)
PHI = (1 + SQRT_FIVE) / 2


def fibonacci(n)
 ((PHI**n)/SQRT_FIVE).round
end
```

This is already much better! We can get the correct Fibonacci numbers well past 75. The trade-off is performance: the more precision requested, the slower the calculations become.

![graph 3](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%203.png)

BigDecimal is a great library to leverage in Ruby. It is a little slower than the standard numeric options, but it is able to avoid many of the precision pitfalls.

This gets to above the 100th fibonacci number. What do we need to do to get to the 1000th?

### Closed form algorithm #3

Ruby’s built-in support for rational numbers provides a promising alternative. Instead of decimals, rationals represent numbers as fractions of integers, avoiding floating-point precision problems.

$\sqrt{5}$ is an irrational number so by definition, there is no rational representation of it. However, there is an approximation of it.

$$\sqrt{5} = 2 + \frac{1}{4 + \frac{1}{4 + \frac{1}{4 + \frac{1}{4 + \ddots}}}}$$

Using continued fractions, we can approximate irrational numbers like $\sqrt{5}$ to arbitrary precision by expanding the fraction deeper and deeper.

$$\sqrt{5} = 2.23606797749978969640\dots$$

If we took this approximation to a couple levels, we get $\frac{38}{17}$ which is about `2.235294117647059`

Going one level deeper would give us $\frac{161}{72}$ which is about `2.2361111111111`. Each level deeper we get a better approximation of $\sqrt{5}$

We can get a pretty good representation of $\sqrt{5}$ if we go 1000 levels deep.

```ruby
APPROXIMATION_DEPTH = 1000
SQRT_FIVE = 2 +
 APPROXIMATION_DEPTH.times.inject(0) do |result, _|
   Rational(1, 4 + result)
 end
PHI = (1 + SQRT_FIVE) / 2


def fibonacci(n)
 (PHI**n/SQRT_FIVE).round
end
```

We can get well above the 1000th Fibonacci number pretty quickly!

![Graph 4](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%204.png)

However, there are still some limitations here. We are working with an approximation so as we are looking for bigger and bigger Fibonacci numbers this approximation will start to drift away from the real values. Theoretically, you could create an even better approximation by going even deeper down the continued fraction, but at some point the numerator and denominator get too large and Ruby will start raising a `FloatDomainError` again.

Rational numbers strike a good balance between precision and performance, making them a powerful tool in the Fibonacci algorithm toolbox.

How about the 10,000th Fibonacci number? Can we quickly calculate that?

## Optimising Recursion with Tail Call Optimization

To get further, we can come back to our original recursion method and make some improvements there. One way we could do this is to remove all of the redundant calculations.

```ruby
# fibonacci.rb
def fibonacci(n, a = 0, b = 1)
 return a if n == 0
 return b if n == 1


 fibonacci(n - 1, b, a + b)
end
```

Unfortunately, this algorithm has one problem. At some point, the fibonacci number will be too large and Ruby will error with a `SystemStackError`. This error usually shows up whenever there is infinite recursion, and while this doesn't recurse indefinitely, it may seem unclear if it is so an error will be raised at some point.

We can improve efficiency by using a little known feature called tail call optimization (TCO). TCO allows a recursive method whose last action is calling itself to reuse the current stack frame instead of creating a new one, effectively turning recursion into iteration.

In Ruby, tail call optimization is disabled by default but can be enabled with a small VM instruction tweak. This lets us compute Fibonacci numbers for much larger inputs (up to hundreds of thousands) without encountering system stack errors.

```ruby
# fibonacci.rb
def fibonacci(n, a = 0, b = 1)
 return a if n == 0
 return b if n == 1


 fibonacci(n - 1, b, a + b)
end


# main.rb
RubyVM::InstructionSequence.compile_option = {
 tailcall_optimization: true,
 trace_instruction: false
}
require_relative './fibonacci'
```

This is significantly better than any of the formula's we've seen so far! No only can we calculate the 10,000th Fibonacci number quickly, but we can blow past the 100,000th number too!

![Graph 5](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%205.png)

While powerful, TCO requires careful coding to ensure the recursive call is in tail position and can complicate debugging. I've never had a case where I needed to use it, but it still really neat to see this kind of feature in Ruby.

Can we calculate the 1,000,000th Fibonaccci number?

## Matrix Multiplication: A Mathematical Approach

To go further, we will need to switch to look at matrices. A matrix is just a 2 dimensional grouping of numbers. For instance this is a 2x3 Matrix containing 6 elements.

$$
\begin{bmatrix}
1 & 9 & -13 \\
20 & 5 & -6
\end{bmatrix}
$$

Matrices have a special way to perform multiplication. It works by having the left most value in a row of the first matrix, multiplied by the top most value of column of the second matrix, and moving to the second values in the row and column and so forth, and then finally summing all of those values together.

$$
\begin{bmatrix}
a_1 & a_2 \\
a_3 & a_4
\end{bmatrix}
\begin{bmatrix}
b_1 & b_2 \\
b_3 & b_4
\end{bmatrix} =
\begin{bmatrix}
a_1b_1 + a_2b_3 & a_1b_2 + a_2b_4 \\
a_3b_1 + a_4b_3 & a_3b_2 + a_4b_4 \\
\end{bmatrix}
$$

It turns out that there is an Fibonacci identity using matices. This uses a special 2x2 matrix:

$$
\begin{bmatrix}
1 & 1 \\
1 & 0
\end{bmatrix}^n =
\begin{bmatrix}
F_{n+1} & F_n \\
F_n & F_{n-1} \\
\end{bmatrix}
$$

If we take this special 2x2 matrix and raise it to the power N, we the result will be a new matrix with 4 Fibonacci numbers in it.

So we can write a new algorithm using Ruby's Matrix library.

```ruby
FIB_MATRIX = Matrix[
  [1, 1],
  [1, 0],
]
def fibonacci(n)
  return 0 if n == 0
  return 1 if n == 1

  result = n.times.inject(Matrix.I(2)) do |acc|
    acc * FIB_MATRIX
  end

  result[0, 1]
end
```

![Graph 6](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%206.png)

Turns out this algorithm is worse that our recursive version. But we can refine this method further. Turns out we can use some properties of matrix multiplication to reduce the amount of operations we are doing. If we are raising a matrix to an particular power, we can break that matrix into powers of 2 that sum to that power, and then calculate those individually and then multiple them together.

$$
\begin{aligned}
\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} ^ {11} & = \begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} ^ {8 + 2 + 1} \\
 & = \begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} ^ {8}
\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} ^ {2}
\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} ^ {1} \\
 & = \begin{bmatrix}
31 & 21 \\
21 & 13 \\
\end{bmatrix}
\begin{bmatrix}
2 & 1 \\
1 & 1 \\
\end{bmatrix}
\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix} \\
& = \begin{bmatrix}
144 & 89 \\
89 & 55 \\
\end{bmatrix}
\end{aligned}
$$

This will allow us to build an even better algorithm. This is known as the repeated squares algorithm.

```ruby
FIB_MATRIX = Matrix[
  [1, 1],
  [1, 0],
]
def fibonacci(n)
  result = Matrix.I(2)
  base = FIB_MATRIX
  while n > 0
    result *= base if (n & 1) == 1
    base *= base if n > 1
    n >>= 1
  end
  result[0, 1]
end
```

![Graph 7](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%207.png)

We were able to perform calculations so much faster than our previous algorithm. Not only finding the 1,000,000th fibonacci, but easily and efficiently finding those above the 45,000,000.

While it is possible to write the algorithm with different methods, it is useful to understand Ruby's bitwise operators. These can be really handy for a number of different purposes. For instance, it is possible to create random numbers that have the current time factored in but using the bitshifting operators to make the current timestamp the most significant digits. You could also take many different boolean configuration options and use bitwise operators to turn them on or off or check if they are set. It is a very handy tool to have access to.

45,000,000 is definitely a big Fibonacci number. But can we do even better?

## Fast Doubling Algorithm

One final leap comes from the fast doubling method, which cleverly uses matrix identities to reduce redundant calculations. This method computes two Fibonacci numbers simultaneously, using formulas that split the problem into halves. Consider the following equation:

$$
\begin{aligned}
\begin{bmatrix}
F_{2n+1} & F_{2n} \\
F_{2n} & F_{2n-1} \\
\end{bmatrix} & =
\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix}^{2n} & =
\left(\begin{bmatrix}
1 & 1 \\
1 & 0 \\
\end{bmatrix}^{n}\right)^2 \\
 & & = \begin{bmatrix}
F_{n+1} & F_{n} \\
F_{n} & F_{n-1} \\
\end{bmatrix} ^ 2 \\
 & & =
\begin{bmatrix}
F_{n+1}^2 + F_n^2 & F_{n+1}F_n + F_nF_{n-1} \\
F_{n+1}F_n + F_nF_{n-1} & F_n^2 + F_{n-1}^2 \\
\end{bmatrix} \\
 & & =
\begin{bmatrix}
F_{n+1}^2 + F_n^2 & 2F_nF_{n+1} - F_n^2 \\
2F_nF_{n+1} - F_n^2 & F_n^2 + F_{n-1}^2 \\
\end{bmatrix}
\end{aligned}
$$

This looks fairly complicated but if you look at the first and the last matrices, we can see a formula for each Fibonacci number expressed by a much smaller Fibonacci number.

$$ F*{2n+1} = F*{n+1}^2 + F_n^2 $$

$$ F*{2n} = 2F_nF*{n+1} - F_n^2 $$

Using this we can create our final algorithm.

```ruby
def fibonacci(n)
  fib_hash = Hash.new do |h, k|
    h[k] = _fib(h, k)
  end
  fib_hash[0] = 0
  fib_hash[1] = 1
  fib_hash[2] = 1
  fib_hash[n]
end

def _fib(hash, n)
  half_n = n / 2
  a = hash[half_n + 1]
  b = hash[half_n]

  if n.odd?
    a * a + b * b
  else
    2 * a * b - b * b
  end
end
```

We can leverage the default proc for Hashes to ensure that we are not doing any redundant calculation.

![Graph 8](https://github.com/doliveirakn/fibonacci/blob/main/images/Graph%208.png)

The performance of this is even better than our repeated squares algorithm! In fact, using this algorithm, you can calculate the 1,000,000,000th Fibonacci number in just a few seconds.

## Conclusion: Ruby and Fibonacci - A Perfect Match

From the naive recursive method to advanced matrix algebra and fast doubling, exploring Fibonacci numbers reveals a rich landscape of algorithms and optimisations. Ruby, with its expressive syntax and powerful libraries like `BigDecimal` and support for rationals and tail call optimization, proves to be an excellent language for both learning and high-performance computation.

Whether you’re preparing for interviews, tackling mathematical problems, or simply fascinated by algorithmic elegance, these Fibonacci funhouse techniques add valuable tools to your programming arsenal.

So next time someone asks you to write a Fibonacci algorithm, you can confidently say: “Yes, and here’s how to do it better.”
